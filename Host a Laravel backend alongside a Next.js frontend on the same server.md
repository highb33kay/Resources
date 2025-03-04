Below is a step‐by‐step tutorial based on the terminal session you provided. This guide explains how to host a Laravel backend alongside a Next.js frontend on the same server—and how to troubleshoot common crashes along the way.

---

## 1. Server Setup and Directory Structure

Assume your web root is located at `/var/www`. Organize your projects into separate directories. For example: 

- **Next.js App:** `/var/www/commande`  
- **Laravel App:** `/var/www/ordering`

You can verify your current directories:

```bash
cd /var/www
ls
```

---

## 2. Managing the Next.js App with PM2

### a. Viewing Running Processes and Logs

You can use [PM2](https://pm2.keymetrics.io/) to run and monitor your Node.js (Next.js) application. Check the list of processes:

```bash
pm2 list
```

If you suspect issues with the Next.js app, check its logs:

```bash
pm2 logs nextjs-app
```

For more detailed info on a specific process, describe it by its PM2 id:

```bash
pm2 describe 0
```

### b. Starting and Restarting the App

When you make changes (or rebuild the app), you might need to restart it. For example, to start the app from the `/var/www/commande` directory:

```bash
pm2 start npm --name "nextjs-app" -- start --cwd "/var/www/commande"
```

If the process is misbehaving or you need a clean slate, delete the existing process:

```bash
pm2 delete 0
```

Then, restart the process as shown above.

---

## 3. Hosting the Laravel Application

### a. Checking Logs and Permissions

Laravel writes its application logs to `storage/logs/laravel.log`. To monitor logs in real time:

```bash
tail -f storage/logs/laravel.log
```

If the log file does not exist or if Laravel is crashing due to file permission issues, create (or adjust) the log file and its permissions:

```bash
cd /var/www/ordering
touch storage/logs/laravel.log
chmod 777 storage/logs/laravel.log
```

If you experience broader permission issues with the Laravel storage directory, set the correct permissions recursively:

```bash
sudo chmod -R 777 storage
```

### b. Environment Configuration

Editing your Laravel `.env` file may be necessary to update configurations (such as database credentials or app debug settings):

```bash
cd /var/www/commande    # or /var/www/ordering if applicable
sudo vi .env
```

Make the required changes, then restart your Laravel service (using your web server or a process manager if needed).

---

## 4. Configuring Nginx as a Reverse Proxy

Edit your Nginx configuration so that it routes requests to the correct application. Open your Nginx config file:

```bash
sudo vi /etc/nginx/sites-available/your-config-file
```

Below is a sample configuration snippet:

```nginx
server {
    listen 80;
    server_name mycommande.com;

    # For Next.js app
    location / {
        proxy_pass http://127.0.0.1:3000;  # Port where Next.js is running
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

server {
    listen 80;
    server_name ordering.mycommande.com;

    root /var/www/ordering/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;  # Adjust for your PHP version
    }

    location ~ /\.ht {
        deny all;
    }
}
```

After saving changes, test and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

You can also check access and error logs for Nginx to diagnose potential issues:

```bash
sudo cat /var/log/nginx/access.log
sudo cat /var/log/nginx/error.log
```

---

## 5. Debugging Application Crashes

### a. Monitor System Resources

Use `htop` to view CPU and memory usage. This helps determine if resource limits are causing crashes:

```bash
htop
```

### b. Review Application Logs

- **Next.js:** Use PM2 logs as shown earlier.  
- **Laravel:** Tail the log file to spot runtime errors:

  ```bash
  tail -f /var/www/ordering/storage/logs/laravel.log
  ```

### c. Rebuild and Restart Applications

For the Next.js app, rebuild if there are changes:

```bash
cd /var/www/commande
npm run build
npm run start
```

For Laravel, ensure that caches are cleared and dependencies are updated:

```bash
cd /var/www/ordering
php artisan cache:clear
php artisan config:clear
composer install
```

---

## 6. Final Checklist

- **Directory Structure:** Confirm that your Laravel and Next.js apps are in separate directories.  
- **Process Management:** Use PM2 to manage the Next.js process.  
- **Nginx Setup:** Ensure your Nginx server blocks are correctly routing traffic to your apps.  
- **File Permissions:** Check and adjust permissions on Laravel’s storage and log files.  
- **Logs:** Regularly check application logs (Laravel logs, PM2 logs, and Nginx logs) to troubleshoot issues.

---

By following these steps, you should be able to host a Laravel backend and a Next.js frontend on the same server while diagnosing and fixing common application crashes. If you encounter any specific errors, reviewing the corresponding logs usually points to the root cause.
