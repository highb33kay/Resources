# Migrating Multiple Web Applications to a New Server with Docker, Traefik, and Nginx

We migrated several applications from an old Linux server to a new one. The stack included:

* A **Laravel PHP application**
* A **Node.js application**
* An **automation/workflow tool** (running in Docker)
* **Traefik** as the edge reverse proxy and TLS terminator
* **Nginx** + **PHP-FPM** behind Traefik
* **MySQL 8** as the main database

## 1. Target Architecture

On the **new server**, the final architecture looks like this:

* **Traefik (Docker)**

  * Listens on ports **80** and **443**
  * Handles **Let’s Encrypt** certificates via ACME
  * Routes traffic based on host names to the correct backend

* **Host Nginx**

  * Listens only on **high ports** (e.g., `8082`) instead of 80/443
  * Serves the **Laravel** app (and other PHP/static apps)

* **PHP-FPM 8.x**

  * Executes PHP for the Laravel application

* **MySQL 8.x**

  * Hosts the Laravel database

* **Node.js app (managed by pm2)**

  * Runs on an internal port (e.g. `3000`)
  * Traefik forwards traffic for a specific domain to that port

Traefik is the single entry point from the internet; everything else is internal.

---

## 2. Copying Nginx Virtual Hosts from the Old Server

On the **old server**, Nginx was configured with multiple virtual hosts located in:

* `/etc/nginx/sites-available`
* `/etc/nginx/sites-enabled` (containing symlinks)

To move those host configurations to the **new server**, we used `rsync` over SSH from the new server:

```bash
# On new-server
rsync -avz root@old-server:/etc/nginx/sites-available/ /etc/nginx/sites-available/
```

After that, we symlinked whichever sites we wanted active:

```bash
cd /etc/nginx/sites-enabled
ln -s ../sites-available/laravel_app .
# ...repeat for other vhosts as required
```

Now the new server has the same virtual host definitions as the old one, but they still need to be adapted to the new environment.

---

## 3. Resolving Port Conflicts (Traefik vs Nginx)

Initially, Nginx refused to start because ports **80** and **443** were already in use. That’s expected, because **Traefik (in Docker)** is designed to own those ports.

We confirmed the conflict with:

```bash
ss -tlnp | grep 80
# Shows Docker/Traefik bound to 0.0.0.0:80 and 0.0.0.0:443
```

To fix this, we updated the Nginx virtual host for the Laravel app, using a high port instead of 80:

```nginx
server {
    listen 8082;
    server_name app.example.com www.app.example.com;

    root /var/www/laravel_app/public;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Then we:

```bash
nginx -t        # configuration test
systemctl start nginx
systemctl status nginx
```

Now Nginx runs happily on port `8082`, while Traefik continues to handle ports `80` and `443`.

---

## 4. Traefik as Edge Proxy and TLS Termination

We run Traefik using Docker Compose. A simplified `docker-compose.yml` for Traefik and an automation tool (like n8n) looks like this:

```yaml
version: "3.7"

services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
      - "--providers.file.directory=/etc/traefik/conf"
      - "--providers.file.watch=true"
    ports:
      - "80:80"
      - "443:443"
      - "8081:8080"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik-conf:/etc/traefik/conf:ro
      - ./traefik-conf:/etc/traefik/dynamic:ro
      - /etc/letsencrypt:/etc/certs:ro

  automation-tool:
    image: some/automation-image
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.automation.rule=Host(`automation.example.com`)
      - traefik.http.routers.automation.tls=true
      - traefik.http.routers.automation.entrypoints=web,websecure
      - traefik.http.routers.automation.tls.certresolver=mytlschallenge

volumes:
  traefik_data:
    external: true
```

A matching static Traefik config (`traefik.yml`) defines the entrypoints and providers:

```yaml
global:
  checkNewVersion: true

api:
  dashboard: true
  insecure: true

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: ":443"

providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true
  docker:
    exposedByDefault: false

certificatesResolvers:
  mytlschallenge:
    acme:
      email: "${SSL_EMAIL}"
      storage: "/letsencrypt/acme.json"
      tlsChallenge: {}
```

### ACME and DNS Gotchas

During setup we saw errors from Traefik about Let’s Encrypt being rate-limited and DNS issues (for example: “DNS problem: NXDOMAIN looking up A for …”). These came from:

* Trying to issue certificates for domains that **did not yet have DNS records**, or
* Repeatedly failing validation, triggering Let’s Encrypt’s **rate limits**.

The fix was simple but important:

1. Ensure **DNS A/AAAA records** for all domains point to the correct server.
2. Avoid requesting certificates for domains that don’t exist or are just placeholders.
3. Wait for rate limits to reset if already hit.

Once DNS was correct, certificate issuance started working reliably.

---

## 5. Routing HTTPS to Laravel via Traefik + Nginx

To route a Laravel app through Traefik, we used a **file provider** configuration in `traefik-conf/laravel.yml`:

```yaml
http:
  routers:
    laravel-router:
      rule: "Host(`app.example.com`)"
      entryPoints: ["websecure"]
      service: "laravel-service"
      tls: {}  # enable TLS with the default resolver

  services:
    laravel-service:
      loadBalancer:
        servers:
          - url: "http://172.17.0.1:8082"
```

Here:

* `app.example.com` is the public domain.
* `172.17.0.1` is the Docker host from Traefik’s perspective.
* `8082` is the port where Nginx serves the Laravel app.

The Laravel Nginx vhost pairs with this by listening on `8082` and handling PHP via PHP-FPM.

---

## 6. Installing PHP-FPM and Fixing 502 Bad Gateway

Early on, accessing the Laravel domain over HTTPS produced:

> 502 Bad Gateway
> nginx/1.24.0 (Ubuntu)

This meant Nginx couldn’t reach PHP-FPM (the `fastcgi_pass` target didn’t exist yet).

We installed PHP-FPM and common extensions:

```bash
sudo apt update
sudo apt install -y php8.3-fpm php8.3-mysql php-redis
```

Then:

```bash
sudo systemctl enable php8.3-fpm
sudo systemctl start php8.3-fpm
sudo systemctl status php8.3-fpm
```

Once PHP-FPM was running and listening on the configured Unix socket:

```nginx
fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
```

the 502 errors disappeared.

---

## 7. Resolving the “Class Redis Not Found” Error in Laravel

After PHP-FPM was working, Laravel threw:

> Class "Redis" not found
> PHP 8.x — Laravel 11.x

This meant the **PHP Redis extension** was missing. Installing `php-redis` (and restarting PHP-FPM) fixed it:

```bash
sudo apt install -y php-redis
sudo systemctl restart php8.3-fpm
```

No application code changes were necessary; the framework could now use Redis as configured in `.env` and `config/cache.php`.

---

## 8. Installing MySQL 8 and Migrating the Laravel Database

The Laravel `.env` was configured roughly as:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=app_database
DB_USERNAME=app_user
DB_PASSWORD=********
```

### 8.1 Installing MySQL on the New Server

On the new server we installed official MySQL 8.x:

```bash
sudo apt update
# Using official MySQL APT repo or distribution packages
sudo apt install -y mysql-server mysql-client

sudo systemctl enable mysql
sudo systemctl start mysql
sudo systemctl status mysql
```

### 8.2 Creating the Database and User

Inside MySQL:

```bash
sudo mysql
```

Then:

```sql
CREATE DATABASE IF NOT EXISTS app_database
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS 'app_user'@'127.0.0.1'
  IDENTIFIED BY '********';

GRANT ALL PRIVILEGES ON app_database.* TO 'app_user'@'127.0.0.1';

FLUSH PRIVILEGES;
EXIT;
```

This matches `.env` exactly (username, password, host).

### 8.3 Dumping the Database from the Old Server

On the **old server**, using a privileged MySQL account:

```bash
mysqldump -u root -p app_database > /root/app_database.sql
```

### 8.4 Copying the Dump with rsync

From the **new server**, we pulled the dump via `rsync`:

```bash
cd /root
rsync -avz root@old-server:/root/app_database.sql /root/
ls -lh /root/app_database.sql
```

### 8.5 Importing into MySQL on the New Server

```bash
mysql -h 127.0.0.1 -u app_user -p app_database < /root/app_database.sql
```

We validated:

```bash
mysql -h 127.0.0.1 -u app_user -p app_database -e "SHOW TABLES;"
```

Finally, from the Laravel project dir:

```bash
cd /var/www/laravel_app
php artisan config:clear
php artisan cache:clear
# php artisan migrate --force  # only if needed
```

The Laravel app now ran on the new server with the restored data.

---

## 9. Hosting the Node.js Application with pm2

We also migrated a Node.js application. The desired setup:

* Node app running on an internal port (e.g. `3000`)
* Managed via pm2
* Traefik routing `node-app.example.com` to that port

### 9.1 Installing Node and pm2

On the new server:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

### 9.2 Starting the App

In the app directory, e.g. `/var/www/node_app`:

```bash
cd /var/www/node_app
pm2 start ecosystem.config.js
pm2 save
pm2 startup systemd
```

The app then listens on an internal port, defined either in `ecosystem.config.js` or environment variables.

### 9.3 Routing Through Traefik

A simple Traefik config (`traefik-conf/node-app.yml`) might be:

```yaml
http:
  routers:
    node-app-router:
      rule: "Host(`node-app.example.com`)"
      entryPoints: ["websecure"]
      service: "node-app-service"
      tls: {}

  services:
    node-app-service:
      loadBalancer:
        servers:
          - url: "http://172.17.0.1:3000"
```

This mirrors the Laravel pattern: Traefik front-end, app behind it on a high port.

---

## 10. Key Takeaways

1. **Single entry point.**
   Traefik owns ports 80 and 443, does TLS, and routes everything. Nginx and Node only listen internally.

2. **Separation of concerns.**

   * Traefik: edge + SSL + routing
   * Nginx: static files + PHP gateway
   * PHP-FPM: executes PHP
   * MySQL: data
   * pm2: manages Node processes

3. **DNS before certificates.**
   Always verify DNS A/AAAA records before triggering Let’s Encrypt. It avoids rate limits and confusing ACME errors.

4. **Match DB config exactly.**
   Database name, user, password, and host must align between `.env` and MySQL grants (`'user'@'host'`).

5. **Extension errors point to missing packages.**
   Messages like `Class "Redis" not found` usually mean a missing PHP extension — fixable by installing and restarting PHP-FPM.

6. **rsync is your friend.**
   For config files and SQL dumps, `rsync` provides a simple, reliable way to move data between servers over SSH.
