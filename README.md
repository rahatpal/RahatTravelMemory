TravelMemory Application
AWS EC2 Deployment Guide
MERN Stack  |  Load Balancing  |  Cloudflare DNS  |  Nginx Reverse Proxy

  
  Repository: https://github.com/UnpredictablePrashant/TravelMemory
  Stack: MongoDB · Express.js · React.js · Node.js
  Infrastructure: AWS EC2 · Application Load Balancer · Nginx · Cloudflare

Table of Contents

1. Prerequisites & Architecture Overview	
2. EC2 Instance Setup	
3. Backend Configuration	
4. Frontend Configuration	
5. Nginx Reverse Proxy Setup	
6. Scaling — Multiple Instances	
7. AWS Application Load Balancer	
8. Cloudflare Domain Setup	
9. Security Best Practices	
10. Troubleshooting	

 
1. Prerequisites & Architecture Overview
1.1 Prerequisites
Ensure the following are in place before starting deployment:

Component	Details
AWS Account	Active account with EC2, VPC, and ALB permissions
EC2 Instance	Ubuntu 22.04 LTS, t2.micro or larger (t3.small recommended)
MongoDB Atlas	Cloud MongoDB cluster with connection string ready
Node.js & npm	Version 18.x or higher (installed on EC2)
Nginx	Latest stable version for reverse proxy
Domain Name	Custom domain registered and managed via Cloudflare
GitHub	Access to the TravelMemory repository
SSH Key Pair	AWS EC2 key pair (.pem file) for instance access

1.2 Architecture Overview
The deployment follows this high-level architecture:

  Architecture Flow
  User Request → Cloudflare DNS → AWS Application Load Balancer
      → EC2 Instance 1 (Nginx → React Frontend + Node.js Backend)
      → EC2 Instance 2 (Nginx → React Frontend + Node.js Backend)
      → MongoDB Atlas (Cloud Database)
  
  Nginx acts as a reverse proxy: port 80/443 → Frontend (React build)
                                  /api/* → Backend (Node.js on port 3000)

 
2. EC2 Instance Setup
2.1 Launch EC2 Instance
Follow these steps to create the primary EC2 instance:

1.	Log in to the AWS Management Console and navigate to EC2 > Instances.
2.	Click Launch Instance and configure the following:
◦	Name: TravelMemory-Server-1
◦	AMI: Ubuntu Server 22.04 LTS (HVM), SSD Volume Type
◦	Instance Type: t3.small (2 vCPU, 2 GB RAM) or t2.micro for free tier
◦	Key Pair: Create new or use existing — save the .pem file securely
◦	Network: Default VPC, enable Auto-assign Public IP
◦	Storage: 20 GB gp3 SSD
3.	Configure Security Group — add the following inbound rules:

Type	Protocol	Port	Source
SSH	TCP	22	Your IP only
HTTP	TCP	80	0.0.0.0/0
HTTPS	TCP	443	0.0.0.0/0
Custom TCP	TCP	3000	0.0.0.0/0 (Backend)

2.2 Connect to Instance
Use SSH to connect to your newly launched EC2 instance:

# Set correct permissions on key file
chmod 400 your-key-pair.pem

# SSH into the instance (replace with your public IP)
ssh -i your-key-pair.pem ubuntu@<EC2_PUBLIC_IP>

2.3 Install Required Dependencies
Update the system and install all required software:

# Update package list
sudo apt update && sudo apt upgrade -y

# Install Node.js 18.x via NodeSource
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify versions
node --version    # Should show v18.x.x
npm --version     # Should show 9.x.x or higher

# Install Nginx
sudo apt install -y nginx

# Install Git
sudo apt install -y git

# Install PM2 (process manager for Node.js)
sudo npm install -g pm2

 
3. Backend Configuration
3.1 Clone Repository

# Clone the TravelMemory repository
cd /home/ubuntu
git clone https://github.com/UnpredictablePrashant/TravelMemory.git

# Navigate to backend directory
cd TravelMemory/backend

# Install dependencies
npm install

3.2 Configure Environment Variables
Create the .env file in the backend directory with your MongoDB connection string:

# Create the .env file
nano /home/ubuntu/TravelMemory/backend/.env

Add the following contents to .env (replace placeholders with real values):

# MongoDB Connection String (from MongoDB Atlas)
MONGO_URI=mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/travelmemory?retryWrites=true&w=majority

# Server Port
PORT=3000

# Environment
NODE_ENV=production

  ⚠️  Important: MongoDB Atlas Network Access
  In MongoDB Atlas, go to Network Access and add your EC2 instance's public IP
  address, OR allow access from 0.0.0.0/0 (all IPs) for development purposes.
  For production, always whitelist specific IP addresses.

3.3 Test Backend Manually

# Start the backend server for testing
cd /home/ubuntu/TravelMemory/backend
node index.js

# In another terminal, test the health endpoint
curl http://localhost:3000
# Expected: Server is running or similar response

# Stop the test server (Ctrl+C) before starting with PM2

3.4 Run Backend with PM2
PM2 ensures the Node.js backend runs continuously and restarts on crash or reboot:

# Start backend with PM2
cd /home/ubuntu/TravelMemory/backend
pm2 start index.js --name travelmemory-backend

# Configure PM2 to start on system reboot
pm2 startup systemd
# Run the command output from above (starts with: sudo env PATH=...)
pm2 save

# Verify the process is running
pm2 status
pm2 logs travelmemory-backend

 
4. Frontend Configuration
4.1 Update Frontend API URL
Before building the frontend, update the urls.js file so the React app knows where the backend lives:

# Open urls.js in the frontend directory
nano /home/ubuntu/TravelMemory/frontend/src/urls.js

Update the file to point to your backend:

// urls.js — Update with your EC2 public IP or domain
// For direct EC2 deployment (no domain yet):
const BACKEND_URL = 'http://<EC2_PUBLIC_IP>:3000';

// OR if using Nginx reverse proxy on same server:
const BACKEND_URL = 'http://<EC2_PUBLIC_IP>/api';

// OR if using custom domain (final setup):
const BACKEND_URL = 'https://api.yourdomain.com';

export default BACKEND_URL;

4.2 Build the React Frontend

# Navigate to frontend directory
cd /home/ubuntu/TravelMemory/frontend

# Install dependencies
npm install

# Create production build
npm run build

# The build output will be in: /home/ubuntu/TravelMemory/frontend/build/
ls -la build/

4.3 Copy Build to Nginx Web Root

# Copy React build to Nginx serving directory
sudo cp -r /home/ubuntu/TravelMemory/frontend/build/* /var/www/html/

# Set correct permissions
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html

 
5. Nginx Reverse Proxy Setup
5.1 Configure Nginx
Create an Nginx server block that serves the React frontend and proxies API requests to Node.js:

# Create a new Nginx configuration file
sudo nano /etc/nginx/sites-available/travelmemory

Add the following Nginx configuration:

server {
    listen 80;
    server_name <YOUR_DOMAIN_OR_EC2_IP>;

    # Serve React frontend (static build)
    root /var/www/html;
    index index.html index.htm;

    # Handle React Router — serve index.html for all frontend routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to Node.js backend on port 3000
    location /api/ {
        proxy_pass http://localhost:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }

    # Security headers
    add_header X-Frame-Options 'SAMEORIGIN';
    add_header X-Content-Type-Options 'nosniff';
    add_header X-XSS-Protection '1; mode=block';
}

5.2 Enable Site and Restart Nginx

# Enable the site by creating a symlink
sudo ln -s /etc/nginx/sites-available/travelmemory /etc/nginx/sites-enabled/

# Remove the default Nginx site
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration for syntax errors
sudo nginx -t
# Expected output: nginx: configuration file ... syntax is ok
# Expected output: nginx: configuration file ... test is successful

# Restart Nginx to apply changes
sudo systemctl restart nginx

# Enable Nginx to start on boot
sudo systemctl enable nginx

# Check Nginx status
sudo systemctl status nginx

5.3 Verify Deployment
Test that the application is accessible:

# Test frontend is served
curl http://localhost

# Test backend API via Nginx proxy
curl http://localhost/api/

# From your browser, open:
# http://<EC2_PUBLIC_IP>  — should show the TravelMemory React app

 
6. Scaling — Multiple Instances
6.1 Create an AMI from Primary Instance
Create an Amazon Machine Image (AMI) from your configured primary instance to quickly launch identical copies:

4.	In the AWS Console, go to EC2 > Instances.
5.	Select TravelMemory-Server-1, right-click > Image and templates > Create image.
6.	Set Image name: TravelMemory-AMI-v1 and click Create image.
7.	Wait for the AMI to reach the Available state (usually 3–10 minutes).

6.2 Launch Additional EC2 Instances

# From AWS Console:
# 1. Go to EC2 > AMIs
# 2. Select TravelMemory-AMI-v1
# 3. Click Launch instance from AMI
# 4. Configure same instance type, security group, and key pair
# 5. Name: TravelMemory-Server-2
# 6. Launch

# Alternatively, use AWS CLI:
aws ec2 run-instances \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --instance-type t3.small \
  --key-name your-key-pair \
  --security-group-ids sg-xxxxxxxxxxxxxxxxx \
  --count 2 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=TravelMemory-Server}]'

  💡  Tip: Verify Each Instance
  After launching each instance, SSH in and verify:
    • pm2 status     — Backend should be running
    • nginx -t       — Configuration should be valid
    • curl localhost — Frontend should respond
  Ensure the .env file is present with correct MongoDB URI.

 
7. AWS Application Load Balancer
7.1 Create Target Group
A Target Group defines which EC2 instances will receive traffic from the load balancer:

8.	Go to EC2 > Target Groups > Create target group.
9.	Choose Instances as the target type.
10.	Configure:
◦	Target group name: TravelMemory-TG
◦	Protocol: HTTP, Port: 80
◦	VPC: Same as your EC2 instances
◦	Health check path: / (or /api/health if available)
11.	Click Next, select both EC2 instances, click Include as pending below, then Create target group.

7.2 Create Application Load Balancer

12.	Go to EC2 > Load Balancers > Create Load Balancer.
13.	Select Application Load Balancer.
14.	Configure:
◦	Name: TravelMemory-ALB
◦	Scheme: Internet-facing
◦	IP address type: IPv4
◦	VPC and Subnets: Select at least two Availability Zones
15.	Security Groups: Create or select one allowing ports 80 and 443.
16.	Listeners and routing: HTTP:80 → Forward to TravelMemory-TG.
17.	Click Create load balancer.
18.	Note the DNS name of the ALB (e.g., TravelMemory-ALB-xxxxxxxx.us-east-1.elb.amazonaws.com).

7.3 Verify Load Balancer

# Test the ALB DNS endpoint in browser or with curl
curl http://TravelMemory-ALB-xxxxxxxx.us-east-1.elb.amazonaws.com

# The load balancer should distribute requests between instances
# Check target group health in AWS Console:
# EC2 > Target Groups > TravelMemory-TG > Targets tab
# All instances should show: healthy

 
8. Cloudflare Domain Setup
8.1 Add Site to Cloudflare

19.	Log in to https://dash.cloudflare.com and click Add a Site.
20.	Enter your domain name and choose a plan (Free plan works for this setup).
21.	Cloudflare will scan existing DNS records. Review and continue.
22.	Update your domain registrar's nameservers to the two provided by Cloudflare.
23.	Wait for nameserver propagation (up to 48 hours, usually faster).

8.2 Create DNS Records
Once the site is active in Cloudflare, add the following DNS records:

Type	Name	Value / Target	TTL
CNAME	www	TravelMemory-ALB-xxx.us-east-1.elb.amazonaws.com	Auto
A	@	<EC2 Primary Instance Public IP>	Auto
CNAME	api	TravelMemory-ALB-xxx.us-east-1.elb.amazonaws.com	Auto

8.3 Enable Cloudflare Proxy & SSL

24.	Ensure the orange cloud icon is active for CNAME/A records (Proxied mode).
25.	Go to SSL/TLS > Overview and set encryption mode to Full (strict).
26.	Go to SSL/TLS > Edge Certificates and enable:
◦	Always Use HTTPS — redirects all HTTP requests to HTTPS
◦	Automatic HTTPS Rewrites
27.	Optionally enable Under Attack Mode during high-traffic events.

8.4 Update Nginx for Domain
Update the Nginx server_name directive to use your custom domain:

# Edit Nginx config
sudo nano /etc/nginx/sites-available/travelmemory

# Update server_name line to:
server_name yourdomain.com www.yourdomain.com;

# Test and reload Nginx
sudo nginx -t && sudo systemctl reload nginx

 
9. Security Best Practices

9.1 EC2 Security Hardening
•	Never expose port 3000 publicly once Nginx proxy is configured — remove that security group rule in production
•	Use IAM roles instead of access keys when possible
•	Enable AWS CloudTrail for audit logging
•	Restrict SSH access (port 22) to specific IP ranges — never 0.0.0.0/0
•	Use a Bastion Host or AWS Systems Manager Session Manager for SSH access

9.2 Application Security
•	Store all secrets in environment variables — never commit .env files to Git
•	Use MongoDB Atlas IP whitelisting — add only EC2 instance IPs
•	Enable CORS headers in Express.js configured to allow only your frontend domain
•	Add rate limiting middleware (e.g., express-rate-limit) to the backend API
•	Implement JWT or session-based authentication if storing user data

9.3 Nginx Security
•	Hide the Nginx version number: add server_tokens off; to nginx.conf
•	Set appropriate security headers (X-Frame-Options, CSP, HSTS)
•	Enable SSL/TLS with Cloudflare's Full (strict) mode
•	Configure fail2ban to block repeated SSH login failures

 
10. Troubleshooting

Common Issue	Resolution
Cannot connect to EC2	Check security group inbound rules. Verify the instance is running. Confirm you're using the correct .pem key file.
Backend not responding on port 3000	Run: pm2 status. Check pm2 logs travelmemory-backend. Verify .env file exists with correct MONGO_URI.
Frontend shows blank page	Run: pm2 status nginx. Check /var/log/nginx/error.log. Ensure React build files are in /var/www/html/
MongoDB connection refused	Check MongoDB Atlas network access whitelist. Verify MONGO_URI in .env is correct. Ensure MongoDB Atlas cluster is running.
Nginx 502 Bad Gateway	Backend is not running. Run: pm2 restart travelmemory-backend. Check port 3000 is in use: netstat -tlnp | grep 3000
Load Balancer targets unhealthy	Verify security group allows ALB to reach port 80 on EC2. Check Nginx is running on all instances. Verify health check path returns HTTP 200.
Domain not resolving	Check Cloudflare DNS records are correct. Wait for propagation (use: dig yourdomain.com). Verify nameservers are updated at registrar.
CORS errors in browser	Add your frontend domain to CORS allowed origins in Express.js. Rebuild and redeploy the backend after changes.

10.1 Useful Diagnostic Commands

# Check all PM2 processes
pm2 status

# View backend logs
pm2 logs travelmemory-backend --lines 50

# Check Nginx error log
sudo tail -50 /var/log/nginx/error.log

# Check Nginx access log
sudo tail -50 /var/log/nginx/access.log

# Test Nginx config
sudo nginx -t

# Check which process is using port 3000
sudo netstat -tlnp | grep 3000

# Check system resources
htop
df -h   # Disk space
free -m # Memory

# Check EC2 system log
sudo journalctl -xe --unit=nginx


