
How to Obtain and Deploy an SSL Certificate for Your Website Using cPanel and acme.sh

acme.sh is a simple command-line tool for obtaining and deploying SSL certificates from Let's Encrypt. It is a popular choice for users of cPanel, as it can be easily integrated with the cPanel UAPI.

To obtain an SSL certificate for your website using cPanel and acme.sh, follow these steps:

Install acme.sh:
curl https://get.acme.sh | sh

Source your .bashrc file:
source ~/.bashrc

Create a Let's Encrypt account:
acme.sh --register-account --accountemail hiibeekayvibe@gmail.com

Upgrade acme.sh:
acme.sh --upgrade

Register your domain name with acme.sh:
acme.sh --register-account --accountemail hiibeekayvibe@gmail.com --server letsencrypt

List your scheduled tasks to see if acme.sh is already running:
crontab -l | grep acme.sh

Get the document root for your domain name from the cPanel UAPI:
uapi DomainInfo single_domain_data domain=propertek.com.ng | grep documentroot

Issue a Let's Encrypt SSL certificate for your domain name using acme.sh and the document root you obtained in the previous step:
acme.sh --issue --webroot /home/anchtdwz/propertek.com.ng -d propertek.com.ng --staging

Note: The --staging flag is optional, but it is recommended to use it first to test your SSL certificate before deploying it to production.

Force acme.sh to issue a new SSL certificate, even if one already exists:
acme.sh --issue --webroot /home/anchtdwz/propertek.com.ng -d propertek.com.ng --force

Force acme.sh to issue a new SSL certificate from Let's Encrypt, even if one already exists:
acme.sh --issue --webroot /home/anchtdwz/propertek.com.ng -d propertek.com.ng --force --server letsencrypt

Deploy the SSL certificate to your cPanel server using acme.sh and the cPanel UAPI:
acme.sh --deploy --deploy-hook cpanel_uapi --domain propertek.com.ng

Once you have completed these steps, your website will be secured using an SSL certificate.

Troubleshooting

If you encounter any problems while obtaining or deploying an SSL certificate using cPanel and acme.sh, please consult the acme.sh documentation or contact your cPanel provider for assistance.
