---
layout: post
title: "Deploying a Next.js App Behind Traefik on Ubuntu (with PM2)"
tags: [reference]
---

# Deploying a Next.js App Behind Traefik on Ubuntu (with PM2)

This is a generic, step-by-step tutorial you can follow on any Ubuntu server that already runs other services (e.g., a PHP app, an automation tool, etc.). We’ll keep each app on its **own internal port** and let **Traefik** handle HTTPS and routing by hostname.

---

## Overview

* Reverse proxy: **Traefik** (Docker)
* App runtime: **Node.js** (Next.js)
* Process manager: **PM2**
* Strategy: Each app listens on a unique host port; Traefik proxies HTTPS traffic to the target port based on the request’s hostname.

---

## 1) Create and register a deploy key (SSH → Git host)

Generate a key on the server and add the **public** key to your Git hosting (user key or repo deploy key):

```bash
ssh-keygen -t ed25519 -C "deploy@example"   # save as ~/.ssh/app_deploy
chmod 600 ~/.ssh/app_deploy
ssh-keygen -y -f ~/.ssh/app_deploy > ~/.ssh/app_deploy.pub
chmod 644 ~/.ssh/app_deploy.pub
```

(If you use multiple keys, add a host alias in `~/.ssh/config` and point your repo remote to it.)

---

## 2) Fetch the Next.js app and build it

```bash
# Place code under /var/www/<your-app>
sudo mkdir -p /var/www/app && sudo chown -R $USER:$USER /var/www/app
git clone <git-ssh-url> /var/www/app
cd /var/www/app

# Install & build for production
npm ci
npm run build
```

> If `npm run build` hasn’t been run, `next start` will fail with “no production build.”

---

## 3) Run the app on its own port with PM2

Pick a free port (example: **3020**) and bind to **0.0.0.0** so Traefik (in Docker) can reach it via the host’s bridge IP.

Create `/var/www/app/ecosystem.config.js`:

```js
module.exports = {
  apps: [
    {
      name: 'next-app',
      script: 'node_modules/next/dist/bin/next',
      args: 'start -p 3020 -H 0.0.0.0',
      cwd: '/var/www/app',
      watch: false,
      env: { NODE_ENV: 'production' },
    },
  ],
};
```

Start & persist:

```bash
cd /var/www/app
pm2 startOrReload ecosystem.config.js
pm2 save

# sanity checks
ss -ltnp | grep 3020
curl -I http://127.0.0.1:3020
```

---

## 4) Traefik basics (Docker + file provider)

Traefik runs as a Docker container and watches a directory on the host for dynamic configs. The key flags look like:

* `--providers.file.directory=/etc/traefik/conf`
* `--providers.file.watch=true`
* `--entrypoints.web.address=:80`
* `--entrypoints.websecure.address=:443`
* `--certificatesresolvers.<name>.acme.tlschallenge=true`
* `--certificatesresolvers.<name>.acme.email=<you@example.com>`
* `--certificatesresolvers.<name>.acme.storage=/letsencrypt/acme.json`

The directory you mount to `/etc/traefik/conf` (e.g., `~/traefik-conf`) is where you’ll put host rules.

---

## 5) Add a Traefik dynamic file for your app

Create a file in the watched directory (e.g., `~/traefik-conf/host-next-app.yml`) and point it at the app’s host port via the Docker bridge IP **172.17.0.1**:

```yaml
http:
  routers:
    nextapp-router:
      rule: "Host(`app.example.com`)"   # replace with your domain
      entryPoints: ["websecure"]
      service: "nextapp-service"
      tls:
        certResolver: mytlschallenge    # must match your Traefik ACME resolver name

  services:
    nextapp-service:
      loadBalancer:
        servers:
          - url: "http://172.17.0.1:****"
```

> Why `172.17.0.1`? Traefik is inside Docker; this IP reaches services bound to the **host**. Binding the app to `0.0.0.0` allows Traefik to connect.

Traefik hot-reloads file changes automatically (if `watch=true` is set). You can confirm via the Traefik dashboard if enabled.

---

## 6) DNS and TLS

Point your app’s hostname (e.g., `app.example.com`) to the server’s public IP. Traefik will request/renew certificates automatically using your configured ACME resolver.

---

## 7) Health checks & logs

* App process:
  `pm2 status`
  `pm2 logs next-app --lines 100`

* Port listening:
  `ss -ltnp | grep 3020`

* Local HTTP check:
  `curl -I http://127.0.0.1:3020`

* Through Traefik (public):
  `curl -I https://app.example.com`

---

## 8) Updates (zero-fuss flow)

```bash
cd /var/www/app
git pull
npm ci
npm run build
pm2 restart next-app
```

---

## 9) Common pitfalls & quick fixes

* **“No production build”**: Run `npm run build` before `next start`.
* **Traefik can’t reach the app**: Ensure the app binds to `0.0.0.0`, and the service URL uses `http://172.17.0.1:<port>`.
* **Wrong port checks**: Verify with `pm2 logs`, `ss -ltnp`, and the actual PM2 `args` in `ecosystem.config.js`.
* **SSH auth errors**: Add the **public** key to your Git host; test with `ssh -T <your-alias-or-host>`.

---

### Done!

You now have a production-grade setup: Next.js managed by PM2 on its own port, fronted by Traefik with automatic HTTPS, and clean, hostname-based routing—ready to scale alongside your other services.
