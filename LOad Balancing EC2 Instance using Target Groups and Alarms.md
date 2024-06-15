How to create CloudWatch alarms to monitor the health of your instances and automatically restart them if they become unhealthy.

### Step 1: Create a Target Group

1. **Navigate to the EC2 Dashboard** and select **Target Groups** from the left-hand menu.
2. **Create a target group**:
   - **Choose a target type**: Instances.
   - **Give your target group a name**.
   - **Protocol and port**: HTTP, Port 80.
   - **VPC**: Select the VPC that your instances are in.
   - **Health checks**: Keep the default settings or adjust according to your needs.
   - Health checks fails sometimes even when the Url is correct, trick is to ensure nginix server or app returns a 404 status code when a route is not found. then use 404 status code as the status code. Trick behind this is that if the server fails the error code will either be a 403 or a 500.  
3. **Register targets**:
   - Select your two EC2 instances.
   - Add them to the target group.

### Step 2: Create an Application Load Balancer

1. **Navigate to the EC2 Dashboard** and select **Load Balancers** from the left-hand menu.
2. **Create a new load balancer**:
   - **Select Load Balancer type**: Application Load Balancer.
   - **Name your load balancer**.
   - **Scheme**: Internet-facing.
   - **IP address type**: IPv4.
   - **Listeners**: Add a listener on port 80.
   - **Availability Zones**: Select the VPC and Availability Zones where your instances are located.

### Step 3: Configure Security Groups for the Load Balancer

1. **Create or select an existing security group** that allows inbound traffic on port 80 (HTTP).
2. **Assign the security group** to the load balancer.

### Step 4: Configure Routing for the Load Balancer

1. **Select the target group** you created earlier.
2. **Configure the default action** to forward traffic to this target group.

### Step 5: Register Targets with the Target Group

1. **Navigate back to Target Groups** in the EC2 Dashboard.
2. **Select your target group**.
3. **Edit and register targets** if not done earlier, ensuring your two instances are added.

### Step 6: Create CloudWatch Alarms

1. **Navigate to the CloudWatch Dashboard**.
2. **Select Alarms** from the left-hand menu and click on **Create Alarm**.
3. **Choose a metric**:
   - **Select the EC2 metrics** and choose **Per-Instance Metrics**.
   - Find and select the **Status Check Failed** metric for your instance(s).
4. **Configure the alarm**:
   - Set the threshold to alarm if the status check fails.
   - Choose the period and evaluation period according to your needs (e.g., 5 minutes).
5. **Add actions**:
   - Click on **Add action** and select **EC2 actions**.
   - Choose **Reboot this instance** or **Recover this instance**.
6. **Name your alarm** and complete the setup.

### Step 7: Test the Setup

1. **Find the DNS name** of your load balancer from the Load Balancers section of the EC2 Dashboard.
2. **Open the DNS name in your browser**. You should see the content served by one of your EC2 instances.
3. **Test by stopping one instance** to see if the load balancer successfully routes traffic to the other instance.
4. **Simulate a failure** to see if the CloudWatch alarm triggers and the instance restarts.

### Detailed Step-by-Step Summary

1. **Launch EC2 Instances**:
   - Ensure two instances are running with HTTP and SSH access.

2. **Create a Target Group**:
   - Navigate to EC2 Dashboard > Target Groups > Create target group.
   - Name the target group and configure settings.
   - Register your EC2 instances.

3. **Create an Application Load Balancer**:
   - Navigate to EC2 Dashboard > Load Balancers > Create load balancer.
   - Choose Application Load Balancer, configure settings, and select VPC and subnets.
   - Assign a security group allowing inbound traffic on port 80.

4. **Configure Routing**:
   - Select the target group for routing.
   - Set the default action to forward traffic to the target group.

5. **Register Targets**:
   - Ensure your EC2 instances are registered with the target group.

6. **Create CloudWatch Alarms**:
   - Navigate to CloudWatch Dashboard > Alarms > Create Alarm.
   - Choose EC2 metrics > Per-Instance Metrics > Status Check Failed.
   - Set threshold, period, and evaluation period.
   - Add actions to reboot or recover the instance.

7. **Test the Setup**:
   - Access the load balancer DNS name to verify traffic routing.
   - Stop one instance to ensure failover works correctly.
   - Simulate a failure to test CloudWatch alarms.

### Conclusion

You have now created an Application Load Balancer, set up a target group, registered your instances, configured the load balancer to route traffic, and set up CloudWatch alarms to monitor the health of your instances and automatically restart them if they become unhealthy. This setup helps distribute incoming traffic across multiple instances, improving availability, reliability, and resilience.
