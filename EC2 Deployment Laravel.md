# Comprehensive Guide: Automate Laravel Deployment to AWS EC2 with GitHub Actions

Automating the deployment process using GitHub Actions for a Laravel application to an AWS EC2 instance can significantly streamline your development workflow. This detailed guide breaks down each step involved in the process, providing a thorough understanding and actionable insights.

## Prerequisites

Before beginning, ensure you have:

- An active AWS account.
- A Laravel application hosted on GitHub.
- Basic familiarity with AWS EC2, IAM, Nginx, and SSH.

## Step 1: Prepare Your AWS EC2 Instance

1. **Launch an EC2 Instance**:
   - Navigate to the EC2 Dashboard in the AWS Management Console.
   - Click on "Launch Instance" and select an Ubuntu Server image. Ubuntu is recommended for its wide support and ease of use with Laravel and Nginx.
   - Choose an instance type. `t2.micro` is sufficient for demonstration purposes.
   - Configure instance details. Leave the defaults unless you have specific requirements.
   - Add storage if needed. The default allocated storage should suffice for a basic Laravel application.
   - Tag your instance for easier management. For example, Key: Name, Value: LaravelServer.
   - Configure the security group to allow inbound traffic on ports 22 (SSH), 80 (HTTP), and 443 (HTTPS).
   - Review and launch the instance, selecting a key pair for SSH access.

2. **Security Group Configuration**:
   - Ensure your security group allows inbound connections on ports 22, 80, and 443 to enable web traffic and SSH access.

## Step 2: Set Up SSH Key Pair

1. **Generate SSH Key Pair** (if not already done):
   - On your local machine, use `ssh-keygen` to generate a new SSH key pair.
   - Follow the prompts, and save your key pair in a secure location.

2. **Add SSH Public Key to EC2**:
   - Navigate to your EC2 instance dashboard.
   - Select your instance and use the "Actions" menu to manage SSH keys.
   - Add the public key you generated to the `authorized_keys` for the default user (typically `ubuntu`).

## Step 3: Configure Nginx on Your EC2 Instance

1. **SSH into Your EC2 Instance**:
   - Use `ssh -i /path/to/your/private/key.pem ubuntu@your_instance_ip` to connect to your instance.

2. **Install Nginx**:
   - Update your package lists: `sudo apt-get update`.
   - Install Nginx: `sudo apt-get install nginx`.
   - Check Nginx is running: `systemctl status nginx`.

3. **Configure Nginx for Laravel**:
   - Navigate to `/etc/nginx/sites-available/`.
   - Create a new configuration file for your Laravel application. You can copy the default config: `sudo cp default laravel`.
   - Edit the `laravel` file, setting the `root` directive to point to your Laravel project's `public` directory and updating the `server_name` directive to your domain or IP address.
   - Symbolically link your config file in `sites-enabled` to enable it: `sudo ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/`.
   - Test Nginx configuration: `sudo nginx -t`.
   - Reload Nginx to apply changes: `sudo systemctl reload nginx`.

## Step 4: Prepare Your Laravel Project for Deployment

1. **Ensure Composer Dependencies**: Verify that `composer.json` and `composer.lock` are up to date and committed to your repository.

2. **Environment Configuration**: Make sure your `.env` file variables are appropriate for production and exclude the `.env` file from source control. Use environment secrets for sensitive information.

## Step 5: Set Up AWS Access Credentials

1. **Create an IAM User**:
   - In the AWS IAM Dashboard, create a new user with programmatic access.
   - Attach policies granting necessary permissions for deployment, such as EC2 and S3 access.

ref: https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/

2. **Record Access Keys**:
   - Securely store the access key ID and secret access key generated upon creating your IAM user.

## Step 6: Set Up GitHub Actions Workflow

1. **Define Workflow File**:
   - In your Laravel project repository, create a new directory `.github/workflows` if it doesn't exist.
   - Create a YAML file for your workflow, e.g., `laravel_ci_cd.yml`.

2. **Workflow Configuration**:
   - Specify the trigger for your workflow, such as on push to the `main` branch.
   - Define jobs, including checking out the repository, setting up PHP, installing Composer dependencies, and deploying to EC2 using SSH and `rsync`.
   - Use actions like `actions/checkout@v2` for checking out code, `shivammathur/setup-php@v2` for setting up PHP, and custom run commands for deployment tasks.

3. **Store SSH and AWS Credentials in GitHub Secrets**:
   - Navigate to your repository's Settings > Secrets.
   - Add your EC2 instance's SSH private key, AWS access key ID, and secret access key as secrets.

## Step 7: Configure GitHub Secrets

- Navigate to "Settings > Secrets" in your repository and add secrets for `SSH_PRIVATE_KEY`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY` with the values obtained in previous steps.

## Step 8: Test the Deployment

- Push changes to your repository to trigger the workflow.
- Monitor the workflow run in the "Actions" tab of your GitHub repository to ensure successful execution.
- Access your application via the EC2 instance's public IP or domain name to verify the deployment.

## Conclusion

Automating your Laravel application deployment to AWS EC2 using GitHub Actions simplifies the process, reduces the potential for human error, and ensures consistent deployments. This guide has detailed each step required to set up this automation, from preparing your AWS EC2 instance and configuring Nginx to setting up the GitHub Actions workflow and securely storing credentials. With this setup, you can focus more on development and less on the intricacies of deployment.

Remember, this guide serves as a starting point. Depending on your specific project needs, consider customizing the Nginx configuration, adjusting file permissions, or enhancing the GitHub Actions workflow for a more tailored deployment process.
