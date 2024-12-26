# Setting Up Supervisor for Laravel Queue Workers

This document provides step-by-step instructions for installing and configuring Supervisor to manage your Laravel queue workers on a small server.

## Prerequisites

-   A Linux server (Ubuntu/Debian/CentOS recommended).
-   `sudo` privileges on the server.
-   Basic familiarity with the command line.
-   A Laravel application setup.

## Step 1: Install Supervisor

1.  **Update your package list:**

    ```bash
    sudo apt update    # For Debian/Ubuntu
    sudo yum update    # For CentOS/RHEL
    ```

2.  **Install Supervisor:**

    ```bash
    sudo apt install supervisor  # For Debian/Ubuntu
    sudo yum install supervisor  # For CentOS/RHEL
    ```

## Step 2: Create the Supervisor Configuration File

1.  **Create the configuration file:**

    Navigate to the Supervisor configuration directory:

    ```bash
    cd /etc/supervisor/conf.d/
    ```

    Create a new file named `laravel-worker.conf`:

    ```bash
    sudo nano laravel-worker.conf
    ```

2.  **Add the following content to the file:**

    ```ini
    [program:laravel-worker]
    process_name=%(program_name)s_%(process_num)02d
    command=php /path/to/your/project/artisan queue:work --sleep=3 --tries=3 --daemon --queue=default
    autostart=true
    autorestart=true
    user=www-data
    numprocs=4
    redirect_stderr=true
    stdout_logfile=/path/to/your/project/storage/logs/worker.log
    stopwaitsecs=3600
    startsecs=10
    startretries=3
    exitcodes=0,2
    killasgroup=true
    priority=999
    ```

    **Important:**
    - Replace `/path/to/your/project/artisan` with the actual absolute path to your Laravel application's `artisan` file.
        - For example: `/var/www/html/my-laravel-app/artisan`
    - Replace `www-data` with the correct username that runs your webserver (e.g., `apache`, `nginx`).
    -  Adjust `numprocs` based on your server's resources and workload. Start with 2-4.
    - Make sure that the directory `/path/to/your/project/storage/logs` exists and that is is writable by `www-data`.

    **Configuration Options Explanation:**

    -   `process_name`:  Names the processes for easy identification (`laravel-worker_00`, `laravel-worker_01`, etc).
    -   `command`:  The command to execute.
        -   `php`: Invokes PHP.
        -   `/path/to/your/project/artisan`: Your `artisan` file.
        -   `queue:work`: Laravel command to process jobs.
        -   `--sleep=3`: Sleep for 3 seconds between jobs when queue is empty.
        -   `--tries=3`: Retry a failed job 3 times.
        -  `--daemon`: Run the worker process in the background.
        -  `--queue=default`: Process jobs on the `default` queue.
    -   `autostart=true`: Automatically starts worker when Supervisor starts.
    -   `autorestart=true`: Automatically restarts workers if they crash.
    -   `user`: Runs the worker under the `www-data` user.
    -   `numprocs`: The number of worker processes (adjust based on needs).
    -   `redirect_stderr=true`: Redirect error outputs to the log.
    -   `stdout_logfile`: The location of the worker log file.
    -   `stopwaitsecs`: Time in seconds Supervisor will wait for jobs to complete when stopping before killing the process.
    -   `startsecs`: Time in seconds for a process to be up before being considered started.
    -   `startretries`: Number of times a worker can try to start before supervisor gives up.
    -   `exitcodes`: Codes that do not cause supervisor to try to restart a process.
    -   `killasgroup`: Forces supervisor to kill child processes on stop
    -   `priority`: The priority of the worker processes.

3.  **Save and exit the file**

## Step 3: Apply the Supervisor Configuration

1.  **Reload Supervisor configurations:**

    ```bash
    sudo supervisorctl reread
    ```

2.  **Apply the new configuration:**

    ```bash
    sudo supervisorctl update
    ```

## Step 4: Start the Queue Workers

1.  **Start all queue worker processes:**

    ```bash
    sudo supervisorctl start laravel-worker:*
    ```

## Step 5: Verify the Setup

1.  **Check the status of your queue workers:**

    ```bash
    sudo supervisorctl status
    ```

    You should see output similar to this:

    ```
    laravel-worker:laravel-worker_00    RUNNING   pid 12345, uptime 0:01:23
    laravel-worker:laravel-worker_01    RUNNING   pid 12346, uptime 0:01:23
    laravel-worker:laravel-worker_02    RUNNING   pid 12347, uptime 0:01:23
    laravel-worker:laravel-worker_03    RUNNING   pid 12348, uptime 0:01:23
    ```
2. **Check log files** Verify that your `/path/to/your/project/storage/logs/worker.log` log file is being updated with details of your queue jobs.

## Step 6: Using Supervisor

-   **Restart Queue Workers:** When you deploy new code, you should restart the queue workers:
   ```bash
     sudo supervisorctl stop laravel-worker:*
     sudo supervisorctl start laravel-worker:*
    ```
    Or as a shortcut:
    ```bash
    sudo supervisorctl restart laravel-worker:*
    ```
-   **Stopping Queue Workers:**
    ```bash
    sudo supervisorctl stop laravel-worker:*
    ```
-   **Starting Queue Workers:**
    ```bash
    sudo supervisorctl start laravel-worker:*
    ```
-   **Checking Status:**
    ```bash
    sudo supervisorctl status
    ```
-   **Viewing Logs:**  Use `tail -f` to monitor the worker logs:
    ```bash
    tail -f /path/to/your/project/storage/logs/worker.log
    ```

## Step 7: Deployment Workflow

Include the following commands in your deployment script *after* you deploy your new code:

1.  **Stop queue workers** (Optional but recommended)

    ```bash
    sudo supervisorctl stop laravel-worker:*
    ```

2.  **Restart Queue workers**

     ```bash
    sudo supervisorctl start laravel-worker:*
    ```

    or using the shortcut:

      ```bash
       sudo supervisorctl restart laravel-worker:*
      ```

**Notes**

-   Always remember to adjust file paths, usernames, and other settings to match your specific server setup.
-   Monitor your server resources and adjust the `numprocs` setting if needed.
-   Use this documentation as a starting point and tailor it to your specific needs.
-  Remember to use a deployment process that takes into account restarts of queue workers.

This `instructions.md` file gives you a comprehensive guide to set up your queue workers in Supervisor. This document is intended to be a complete guide, which will save you time, and avoid unnecessary errors when configuring Supervisor.
