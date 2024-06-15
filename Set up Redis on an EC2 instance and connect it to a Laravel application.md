Sure, here’s a comprehensive step-by-step tutorial to set up Redis on an EC2 instance and connect it to a Laravel application.

### Step 1: Launch an EC2 Instance

1. **Log in to your AWS Management Console** and navigate to the EC2 Dashboard.
2. **Launch a new instance**:
   - Choose an Amazon Machine Image (AMI): Select an Ubuntu Server AMI (e.g., Ubuntu Server 20.04 LTS).
   - Choose an Instance Type: Select an instance type (e.g., t2.micro for testing).
   - Configure Instance: Configure according to your needs.
   - Add Storage: Keep the default settings or modify as needed.
   - Add Tags: Optionally add tags.
   - Configure Security Group: Create a new security group or select an existing one. Ensure you allow inbound traffic on port 22 (SSH) and port 6379 (default Redis port).
3. **Launch the instance** and download the key pair (.pem file) if you don't already have one.

### Step 2: Connect to the EC2 Instance

1. **Open your terminal** (or use PuTTY on Windows).
2. Change permissions for your key pair file:
   ```bash
   chmod 400 your-key-pair.pem
   ```
3. **Connect to your instance**:
   ```bash
   ssh -i "your-key-pair.pem" ubuntu@your-ec2-public-dns
   ```

### Step 3: Install Redis

1. **Update the package list**:
   ```bash
   sudo apt-get update
   ```
2. **Install Redis**:
   ```bash
   sudo apt-get install redis-server
   ```
3. **Verify the installation**:
   ```bash
   redis-server --version
   ```

### Step 4: Configure Redis

1. **Open the Redis configuration file**:
   ```bash
   sudo nano /etc/redis/redis.conf
   ```
2. **Configure Redis**:
   - Change the `bind` address to `0.0.0.0` to allow connections from any IP address (or specify your IP range for more security):
     ```ini
     bind 0.0.0.0
     ```
   - Set `supervised` to `systemd`:
     ```ini
     supervised systemd
     ```
3. **Save and exit** the file (`Ctrl+X`, then `Y`, and `Enter`).

### Step 5: Start Redis

1. **Restart the Redis service**:
   ```bash
   sudo systemctl restart redis.service
   ```
2. **Enable Redis to start on boot**:
   ```bash
   sudo systemctl enable redis.service
   ```

### Step 6: Test Redis

1. **Connect to Redis CLI**:
   ```bash
   redis-cli
   ```
2. **Run a simple test**:
   ```bash
   set test "Redis is running"
   get test
   ```
   You should see:
   ```
   "Redis is running"
   ```

### Step 7: Configure Security Group

Ensure your EC2 security group allows inbound traffic on port 6379:
1. **Go to the EC2 Dashboard** > **Instances** > Select your instance.
2. **Click on the Security Group** associated with the instance.
3. **Edit Inbound Rules** to add a rule allowing TCP traffic on port 6379 from your IP address or range.

### Step 8: Install Redis PHP Extension on Laravel Server

1. **Connect to your Laravel application's server** and install the Redis PHP extension.
   ```bash
   sudo apt-get update
   sudo apt-get install php-redis
   ```
2. **Restart your web server** (e.g., Apache or Nginx) to load the new PHP extension.
   ```bash
   sudo systemctl restart apache2
   # or for Nginx
   sudo systemctl restart nginx
   ```

### Step 9: Configure Laravel to Use Redis

1. **Open your Laravel configuration file** for Redis located at `config/database.php`.

2. **Locate the Redis array configuration** section and update it to match your Redis server settings:
   ```php
   'redis' => [

       'client' => env('REDIS_CLIENT', 'phpredis'),

       'default' => [
           'host' => env('REDIS_HOST', '127.0.0.1'),
           'password' => env('REDIS_PASSWORD', null),
           'port' => env('REDIS_PORT', 6379),
           'database' => env('REDIS_DB', 0),
       ],

       'cache' => [
           'host' => env('REDIS_HOST', '127.0.0.1'),
           'password' => env('REDIS_PASSWORD', null),
           'port' => env('REDIS_PORT', 6379),
           'database' => env('REDIS_CACHE_DB', 1),
       ],

   ],
   ```

3. **Update your environment file (`.env`) to include the Redis configuration**:
   ```dotenv
   REDIS_HOST=127.0.0.1
   REDIS_PASSWORD=null
   REDIS_PORT=6379
   ```

### Step 10: Test the Connection in Laravel

1. **Create a test route in `routes/web.php`**:
   ```php
   use Illuminate\Support\Facades\Redis;

   Route::get('/redis-test', function () {
       $redis = Redis::connection();
       $redis->set('test_key', 'Redis is connected!');
       return $redis->get('test_key');
   });
   ```

2. **Access the route** in your browser by navigating to `http://your-laravel-app/redis-test`. You should see `Redis is connected!`.

### Optional: Check Redis Connection via Laravel Tinker

1. **Run Tinker**:
   ```bash
   php artisan tinker
   ```

2. **Test Redis Commands**:
   ```php
   $redis = Redis::connection();
   $redis->set('test_key', 'Redis is connected via Tinker!');
   echo $redis->get('test_key');
   ```

   You should see the output:
   ```
   Redis is connected via Tinker!
   ```

### Conclusion

You have now set up Redis on an EC2 instance and configured your Laravel application to connect to it. This setup allows your Laravel application to utilize Redis for caching, session storage, and other purposes, providing improved performance and scalability.
