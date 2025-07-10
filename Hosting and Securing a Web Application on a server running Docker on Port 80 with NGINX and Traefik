# Documentation: Hosting Multiple Applications with Docker and Traefik

This guide details how to add a second, Nginx-based web application to an existing server setup that is already managed by Traefik. It assumes you have successfully completed the initial setup for your first application.

### Table of Contents
1.  **Prerequisites**
2.  **Core Concept: How Traefik Manages Multiple Services**
3.  **Step 1: Prepare and Run Your Second Application Container**
4.  **Step 2: Configure Traefik to Route to the New Application**
5.  **Step 3: Secure the New Application with HTTPS**
6.  **Step 4: Final Deployment**
7.  **Troubleshooting**

---

### 1. Prerequisites

*   A server with a working Traefik instance and at least one other application already configured, as per the previous documentation.
*   Your new application packaged in a Docker container (we will use a standard Nginx image as an example).
*   A new domain or subdomain (e.g., `app2.example.com`) with its DNS A record pointing to your server's IP address.

---

### 2. Core Concept: How Traefik Manages Multiple Services

The beauty of Traefik is that you **do not need to reconfigure Traefik itself** to add new services. Traefik *watches* for new configuration files and running containers.

Our goal is to:
1.  Run our new Nginx application container on the same Docker network as Traefik.
2.  Create a new, separate `.yml` file in the `traefik-conf` directory that tells Traefik how to route traffic to this new container.

Traefik will see the new file and the new container and automatically create the routes.

---

### 3. Step 1: Prepare and Run Your Second Application Container

First, we need to define our new Nginx application in the main `docker-compose.yml` file. This tells Docker how to run it.

Open your `docker-compose.yml` file and add a new service for the second application.

```yaml
# /home/user/my_project/docker-compose.yml

version: '3.8'

services:
  traefik:
    # ... your existing traefik configuration ...
    # ... it should already be on the 'web' network ...
    networks:
      - web

  # This is your FIRST application (example)
  app-one:
    image: laravel-app:latest
    container_name: laravel-app
    restart: unless-stopped
    expose:
      - "8080"
    networks:
      - web

  # --- ADD THE NEW NGINX APPLICATION SERVICE BELOW ---
  app-two-nginx:
    image: nginx:latest # Using a standard Nginx image for this example
    container_name: my-nginx-app
    restart: unless-stopped
    expose:
      # Expose port 80 INSIDE the container. No host port mapping needed!
      - "80"
    volumes:
      # Optional: Mount your website's HTML files into the container
      - ./nginx-site-files:/usr/share/nginx/html:ro
    networks:
      - web # CRITICAL: It must be on the same network as Traefik.

networks:
  web:
    name: web_network
```
**Key Points:**
*   **`app-two-nginx`:** This is the service name we'll use to refer to this container.
*   **`expose: - "80"`:** We are only exposing the port *internally* to the Docker network. We do NOT map it to the host (e.g., `ports: - "8081:80"`) because Traefik will handle all external traffic.
*   **`networks: - web`:** This is the most critical part. The new application **must** be on the same network as Traefik so they can communicate.

---

### 4. Step 2: Configure Traefik to Route to the New Application

Now, create a new dynamic configuration file just for this Nginx app. Traefik will automatically discover and load it.

**Create file:** `traefik-conf/app-two.yml`

```yaml
# /home/user/my_project/traefik-conf/app-two.yml

http:
  routers:
    # A unique name for the new router
    nginx-app-router:
      rule: "Host(`app2.example.com`)" # The domain for this new app
      entryPoints: ["websecure"]
      service: "nginx-app-service"
      tls: {}

  services:
    # A unique name for the new service
    nginx-app-service:
      loadBalancer:
        servers:
          # Point to the container by its service name and internal port
          - url: "http://app-two-nginx:80"
```
**Key Points:**
*   **`rule: "Host(...)`:** This defines the domain that will route to this container.
*   **`url: "http://app-two-nginx:80"`:** This is the "magic" of Docker networking. We use the *service name* from `docker-compose.yml` (`app-two-nginx`) and the port it exposes internally (`80`). Traefik will resolve this to the container's internal IP address. This is more robust than using a hardcoded IP.

---

### 5. Step 3: Secure the New Application with HTTPS

Just like before, you need an SSL certificate for `app2.example.com`.

**A. Obtain the Certificate**
On your host machine, run Certbot for the new domain:
```bash
sudo certbot certonly --manual --preferred-challenges=dns -d app2.example.com
```
Complete the DNS validation process.

**B. Update the TLS Configuration**
Edit your **existing** `traefik-conf/tls.yml` file and **add** the new certificate to the list.

```yaml
# /home/user/my_project/traefik-conf/tls.yml
tls:
  certificates:
    # --- Your FIRST certificate ---
    - certFile: /etc/certs/live/app.example.com/fullchain.pem
      keyFile: /etc/certs/live/app.example.com/privkey.pem
      
    # --- ADD THE NEW certificate for the second app ---
    - certFile: /etc/certs/live/app2.example.com/fullchain.pem
      keyFile: /etc/certs/live/app2.example.com/privkey.pem
```
Traefik will load all certificates from this list and automatically use the correct one based on the `Host()` rule in each router.

---

### 6. Step 4: Final Deployment

Now that all the configuration is in place, go to your project directory and run Docker Compose. It will start your new Nginx container and Traefik will automatically configure the routing.

```bash
cd /home/user/my_project/
docker-compose up -d
```

Your new application should now be live and accessible at `https://app2.example.com`.

---

### 7. Troubleshooting

*   **404 Not Found:** Check for typos in your `app-two.yml` file (`http`, `Host()`, etc.). Verify the new router appears in the Traefik Dashboard.
*   **502 Bad Gateway:** Traefik can't connect to your Nginx container.
    1.  Confirm both Traefik and `app-two-nginx` are on the `web_network`. Run `docker network inspect web_network` to see which containers are attached.
    2.  Check that the service name in the `url` (`http://app-two-nginx:80`) exactly matches the service name in `docker-compose.yml`.
    3.  Check the logs of your Nginx container (`docker-compose logs app-two-nginx`) to ensure it started correctly.
