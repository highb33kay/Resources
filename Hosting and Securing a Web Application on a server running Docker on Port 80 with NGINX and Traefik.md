# Documentation: Using Traefik in Docker to Manage an Nginx Service on the Host Machine

This guide details how to use a Dockerized Traefik instance to act as a reverse proxy for a web application served by Nginx running directly on the host operating system.

### Table of Contents
1.  **Prerequisites**
2.  **Core Concept: How Traefik Connects to the Host**
3.  **Step 1: Verify Your Host Nginx Setup**
4.  **Step 2: Configure Traefik with Docker Compose**
5.  **Step 3: Configure the Traefik Router for Nginx**
6.  **Step 4: Securing the Nginx Service with HTTPS**
7.  **Step 5: Launching Traefik**
8.  **Troubleshooting**

---

### 1. Prerequisites

*   A server with a public IP address.
*   Docker and Docker Compose installed.
*   **Nginx installed and running as a service directly on the host machine** (e.g., installed via `apt`, `yum`, etc.).
*   Your Nginx service is configured to serve a website on a specific port (e.g., `localhost:8080`).
*   A registered domain name (e.g., `nginx-app.example.com`) pointing to your server's public IP.

---

### 2. Core Concept: How Traefik Connects to the Host

When Traefik is inside a Docker container, `localhost` or `127.0.0.1` refers to the *container itself*, not the host machine. To connect to a service running on the host, Traefik needs the host's IP address on the Docker network.

Docker creates a virtual network bridge, typically named `docker0`. By default, the host machine is accessible at the IP address of this bridge interface, which is usually `172.17.0.1`.

**Our goal:** Configure a Traefik router to forward traffic for `nginx-app.example.com` to `http://172.17.0.1:8080`, where your host's Nginx is listening.

---

### 3. Step 1: Verify Your Host Nginx Setup

Before configuring Traefik, ensure your Nginx service is working correctly.

1.  **Configure Nginx to Listen on a High Port:** Do not let Nginx try to use ports 80 or 443, as Traefik will need those. Configure your Nginx virtual host to listen on a different port, like `8080`.

    Example Nginx site config (`/etc/nginx/sites-available/my-app`):
    ```nginx
    server {
        # Listen on a non-standard port
        listen 8080;
        listen [::]:8080;

        server_name nginx-app.example.com;
        root /var/www/my-app;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```
    Enable the site and restart Nginx: `sudo systemctl restart nginx`.

2.  **Test Locally:** From your server's command line, test that Nginx is responding on that port.
    ```bash
    curl http://localhost:8080
    ```
    This should return the HTML of your website. If it doesn't, resolve your Nginx configuration first.

---

### 4. Step 2: Configure Traefik with Docker Compose

This is the standard Traefik setup. If you already have Traefik running, you can skip to the next step.

**File:** `docker-compose.yml`
```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik-conf:/etc/traefik/dynamic:ro
      # Volume for certificates will be added in the HTTPS step
```

**File:** `traefik.yml`
```yaml
global:
  checkNewVersion: false

api:
  dashboard: true
  insecure: true # Secure this for production

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint: { to: websecure, scheme: https }
  websecure:
    address: ":443"

providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true
```

---

### 5. Step 3: Configure the Traefik Router for Nginx

This is the key step. We will create a dynamic configuration file that tells Traefik how to find the Nginx service running on the host.

**Create file:** `traefik-conf/host-nginx-app.yml`
```yaml
# /home/user/my_project/traefik-conf/host-nginx-app.yml

http:
  routers:
    # A unique name for this router
    nginx-host-router:
      rule: "Host(`nginx-app.example.com`)"
      entryPoints: ["websecure"]
      service: "nginx-host-service"
      tls: {} # Enable TLS

  services:
    # A unique name for this service
    nginx-host-service:
      loadBalancer:
        servers:
          # IMPORTANT: Point to the Docker host's gateway IP and port
          - url: "http://172.17.0.1:8080"
```
**To find your Docker host gateway IP:**
If `172.17.0.1` doesn't work, run `ip addr show docker0` on your host machine and use the `inet` address you see.

---

### 6. Step 4: Securing the Nginx Service with HTTPS

You need an SSL certificate for `nginx-app.example.com`.

**A. Obtain the Certificate**
Use Certbot on your host machine:
```bash
sudo certbot certonly --manual --preferred-challenges=dns -d nginx-app.example.com
```
Complete the DNS validation. Your files will be in `/etc/letsencrypt/live/nginx-app.example.com/`.

**B. Mount the Certificates into Traefik**
Edit `docker-compose.yml` and add the certificate volume:
```yaml
# docker-compose.yml
services:
  traefik:
    # ... other config ...
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik-conf:/etc/traefik/dynamic:ro
      - /etc/letsencrypt:/etc/certs:ro # <-- ADD THIS LINE
```

**C. Create the TLS Configuration**
Create a file to tell Traefik about your certificate. If you already have this file, just add the new certificate to it.

**File:** `traefik-conf/tls.yml`
```yaml
# traefik-conf/tls.yml
tls:
  certificates:
    - certFile: /etc/certs/live/nginx-app.example.com/fullchain.pem
      keyFile: /etc/certs/live/nginx-app.example.com/privkey.pem
```

---

### 7. Step 5: Launching Traefik

With all the configuration in place, start the Traefik container.

```bash
cd /path/to/your/project
docker-compose up -d
```
Traefik will start, read its configuration files, and begin routing traffic for `https://nginx-app.example.com` to your Nginx service running on `localhost:8080`.

---

### 8. Troubleshooting

*   **502 Bad Gateway:** This is the most common error in this setup. It means Traefik cannot reach your Nginx service.
    1.  **Check the IP:** Is `172.17.0.1` correct? Verify with `ip addr show docker0`.
    2.  **Check the Port:** Is Nginx on the host listening on the correct port (`8080` in this example)? Use `ss -tulpn | grep 8080`.
    3.  **Check Firewall:** Is a host firewall (like `ufw`) blocking traffic from the Docker network (`172.17.0.0/16`) to port 8080? You may need to add an allow rule.

*   **404 Not Found:** Traefik is not recognizing the router.
    *   Check for typos in `traefik-conf/host-nginx-app.yml` (especially `http:` and the `Host()` rule).
    *   Check the Traefik Dashboard (`http://<your_ip>:8080`) to see if the router is listed and active.
 


# Documentation: Deploying a Host Application with Docker, Traefik, and Nginx

This guide provides a complete, real-world walkthrough for deploying a web application. It uses Traefik (in Docker) as a reverse proxy to manage and secure an Nginx web server running directly on the host machine.

This documentation uniquely includes a full troubleshooting section based on common, real-world errors, such as typos and misconfigurations.

### Table of Contents
1.  **Prerequisites**
2.  **Core Architecture**
3.  **Step 1: The Initial (Flawed) Configuration**
4.  **Step 2: Initial Deployment and First Error (404 Not Found)**
5.  **Step 3: Debugging with Logs - Finding the Typo**
6.  **Step 4: The Corrected Configuration**
7.  **Step 5: Securing with a Custom SSL Certificate**
8.  **Step 6: Final Deployment and Verification**
9.  **Further Troubleshooting Guide**

---

### 1. Prerequisites

*   A server with a public IP and Docker/Docker Compose installed.
*   Nginx installed and running on the host, serving a site on a non-standard port (e.g., `8080`).
*   A domain name (e.g., `app.example.com`) pointed to your server's IP.

---

### 2. Core Architecture

*   **Internet Traffic** hits your server on ports 80 (HTTP) and 443 (HTTPS).
*   **Traefik (in Docker)** listens on these ports.
*   **Routing Rule:** Traefik sees a request for `app.example.com` and, based on our rules, forwards it internally.
*   **Host Nginx Service:** The request is sent to `http://172.17.0.1:8080`, where `172.17.0.1` is the IP of your host machine as seen from inside a Docker container.

---

### 3. Step 1: The Initial (Flawed) Configuration

We will start with a configuration that contains a subtle, common typo. This is intentional to demonstrate the debugging process.

**Directory Structure:**
```
/home/user/my_project/
├── docker-compose.yml
├── traefik.yml
└── traefik-conf/
    └── my-app.yml
```

**`docker-compose.yml`:**
```yaml
# This file defines the Traefik service
version: '3.8'

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik-conf:/etc/traefik/dynamic:ro
      # Certificate volume will be added later
```

**`traefik.yml`:**
```yaml
# This file contains Traefik's static startup configuration
global:
  checkNewVersion: false

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true
```

**`traefik-conf/my-app.yml` (WITH THE TYPO):**
This file tells Traefik how to handle our app. Notice the `gttp:` typo on the first line.
```yaml
# A typo on the first line will cause this entire file to be ignored
gttp:
  routers:
    my-app-router:
      rule: "Host(`app.example.com`)"
      entryPoints: ["websecure"]
      service: "my-app-service"
      tls: {}

  services:
    my-app-service:
      loadBalancer:
        servers:
          # Point to the Nginx service running on the host
          - url: "http://172.17.0.1:8080"
```

---

### 4. Step 2: Initial Deployment and First Error (404 Not Found)

With the flawed configuration, let's deploy and see what happens.

1.  Start Traefik:
    ```bash
    cd /home/user/my_project/
    docker-compose up -d
    ```
2.  **Test in Browser:** Navigate to `http://app.example.com`.
3.  **Result:** You will receive a **404 Not Found** error page. This 404 is coming directly from Traefik because it has no idea what `app.example.com` is.

---

### 5. Step 3: Debugging with Logs - Finding the Typo

The first step in debugging is always to check the logs.

1.  Run the log command:
    ```bash
    docker-compose logs traefik
    ```

2.  **Analyze the Output:**
    You might expect a big error message, but in our real example, the output was clean:
    ```
    WARN[0000] /root/docker-compose.yml: the attribute `version` is obsolete...
    ```
    The absence of an error is a clue. It means the `my-app.yml` file has no *syntax* errors, but Traefik is still not loading it. This points to a problem with a top-level key.

3.  **Manual Inspection:**
    Upon manually inspecting `traefik-conf/my-app.yml`, we find the problem:
    *   **Incorrect:** `gttp:`
    *   **Correct:** `http:`

    Traefik doesn't recognize the `gttp:` key, so it silently ignores the entire file, leading to the 404.

---

### 6. Step 4: The Corrected Configuration

Let's fix the typo.

**Edit `traefik-conf/my-app.yml`:**
```yaml
# Corrected top-level key
http:
  routers:
    my-app-router:
      rule: "Host(`app.example.com`)"
      entryPoints: ["websecure"]
      service: "my-app-service"
      tls: {}

  services:
    my-app-service:
      loadBalancer:
        servers:
          - url: "http://172.17.0.1:8080"
```
**Do not restart yet!** We still need to configure SSL.

---

### 7. Step 5: Securing with a Custom SSL Certificate

Now we add the HTTPS layer.

**A. Obtain the Certificate:**
Run Certbot on the host machine for your domain.
```bash
sudo certbot certonly --manual --preferred-challenges=dns -d app.example.com
```
Follow the prompts to validate your domain. Your certificate will be at `/etc/letsencrypt/live/app.example.com/`.

**B. Mount Certificates into Traefik:**
Edit `docker-compose.yml` to add the `letsencrypt` volume.

```yaml
# docker-compose.yml
services:
  traefik:
    # ...
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik-conf:/etc/traefik/dynamic:ro
      - /etc/letsencrypt:/etc/certs:ro # <-- ADD THIS LINE
```

**C. Create the TLS Configuration File:**
This file tells Traefik which certificates to load.

**Create File:** `traefik-conf/tls.yml`
```yaml
# traefik-conf/tls.yml
tls:
  certificates:
    - certFile: /etc/certs/live/app.example.com/fullchain.pem
      keyFile: /etc/certs/live/app.example.com/privkey.pem
```
Because this file is in the `traefik-conf` directory, Traefik's file provider will automatically discover and load it.

---

### 8. Step 6: Final Deployment and Verification

With the typo fixed and SSL configured, redeploy Traefik.

```bash
docker-compose up -d --force-recreate traefik
```

1.  **Check Logs:** Run `docker-compose logs traefik`. You should now see logs indicating that `my-app.yml` and `tls.yml` were loaded successfully.
2.  **Test in Browser:** Navigate to `https://app.example.com`.
3.  **Result:** Your site should now load correctly with a valid HTTPS certificate.

---

### 9. Further Troubleshooting Guide

*   **502 Bad Gateway:** Traefik is configured correctly but cannot reach your host's Nginx service.
    *   **Check IP:** Confirm your host's Docker IP with `ip addr show docker0`. Ensure it matches the `url` in `my-app.yml`.
    *   **Check Port:** Is Nginx listening on the correct port? Use `ss -tulpn | grep 8080`.
    *   **Check Firewall:** Ensure your host firewall (e.g., `ufw`) is not blocking connections from the Docker network (`172.17.0.0/16`) to port `8080`.

*   **Certificate Errors:**
    *   The `Host()` rule in `my-app.yml` does not exactly match the domain name in the SSL certificate loaded by `tls.yml`.
    *   File permissions on `/etc/letsencrypt/` might be too restrictive. Run `sudo chmod -R 755 /etc/letsencrypt/{live,archive}` to make them readable.
