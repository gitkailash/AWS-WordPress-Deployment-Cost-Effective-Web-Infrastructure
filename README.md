# AWS-WordPress-Deployment-Cost-Effective-Web-Infrastructure

This project demonstrates the setup of an AWS environment, including a Virtual Private Cloud (VPC), Internet Gateway(IGW), Security Groups(SG), Application Load Balancer (ALB), Auto Scaling Group (ASG), and EC2 instances running a WordPress application. The project follows best practices for network configuration, security, and scalability, ensuring a robust and highly available web infrastructure.

---
## Project Overview

The goal of this project is to create a scalable web application infrastructure using AWS services, with a focus on automating the deployment of web servers via Auto Scaling and distributing traffic using an Application Load Balancer.

---

## Prerequisites

Before starting, ensure you have:

- An AWS account with necessary IAM permissions.
- AWS CLI installed and configured.
- Basic knowledge of AWS services (VPC, EC2, ALB, ASG).
- Familiarity with Bash scripting for user data.

---

## Steps to Implement

### 1. Create a VPC

1.1 **Create VPC**  
- Navigate to the VPC Dashboard.
- Click on "Create VPC".
- Name: `MyVPC` (Example)
- IPv4 CIDR block: `10.0.0.0/20`
- Leave other settings as default.

---
### 2. Set Up Internet Gateway

2.1 **Create an Internet Gateway**  
- In the VPC Dashboard, go to "Internet Gateways".
- Click on "Create Internet Gateway".
- Name: `MyInternetGateway`
- Attach the Internet Gateway to `MyVPC`.

---
### 3. Configure Security Groups

3.1 **Create Security Group for ALB (ALB-SG)**  
- In the VPC Dashboard, go to "Security Groups".
- Click "Create Security Group".
- Name: `ALB-SG`
- VPC: `MyVPC`
- Inbound Rules:
  - Type: HTTP, Source: `0.0.0.0/0`
  - Type: SSH, Source: `0.0.0.0/0`
  
3.2 **Create Security Group for Web Tier (Web-SG)**  
- Click "Create Security Group".
- Name: `Web-SG`
- VPC: `MyVPC`
- Inbound Rules:
  - Type: HTTP, Source: `ALB-SG`
  - Type: SSH, Source: `ALB-SG`

---
### 4. Create Public Subnets

4.1 **Create Two Public Subnets**  
- In the VPC Dashboard, go to "Subnets".
- Click "Create Subnet".
- Name: `Public-Subnet-1`
- VPC: `MyVPC`
- Availability Zone: `us-east-1a` (Example)
- IPv4 CIDR block: `10.0.0.64/26`
- Repeat for the second subnet.
  - Name: `Public-Subnet-2`
  - Availability Zone: `us-east-1b`
  - IPv4 CIDR block: `10.0.0.128/26`
  
4.2 **Enable Auto-Assign Public IP**  
- Select subnet and click "Actions" -> "Edit subnet settings".
- Check "Enable auto-assign public IPv4 address".
- Do for both subnets

---
### 5. Set Up Route Table

5.1 **Create and Configure Route Table**  
- In the VPC Dashboard, go to "Route Tables".
- Click "Create route tables".
- Name: `Public-Route-Table`
- VPC: `MyVPC`
- Click "Edit routes" and add a route:
  - Destination: `0.0.0.0/0`
  - Target: `MyInternetGateway`
  
5.2 **Associate Subnets with Route Table**  
- Select the `Public-Route-Table`.
- Click "Subnet Associations".
- Select both public subnets created earlier.

---
### 6. Set Up Target Group

6.1 **Create a Target Group for the ALB**  
- Navigate to the EC2 Dashboard.
- Go to "Target Groups".
- Click "Create target group".
- Name: `MyTargetGroup`
- Target type: `Instance`
- Protocol: `HTTP`, Port: `80`
- VPC: `MyVPC`

---
### 7. Create Launch Template

7.1 **Create a Launch Template for EC2 Instances**  
- In the EC2 Dashboard, go to "Launch Templates".
- Click "Create launch template".
- Name: `MyLaunchTemplate`
- AMI: Select an appropriate Amazon Linux 2 AMI.
- Instance type: `t2.micro` (Example)
- Security Group: `Web-SG`
- User Data (Bash Script to Install Apache):
  ```bash
  #!/bin/bash

  # Variables
  DB_NAME="wordpress_db"
  DB_USER="wordpress_user"
  DB_PASS="yourpassword"
  
  # Update the system and install necessary packages
  sudo yum update -y
  sudo yum install -y httpd mariadb105-server wget php php-mysqlnd php-fpm
  
  # Start and enable Apache
  sudo systemctl start httpd
  sudo systemctl enable httpd
  
  # Download and extract WordPress
  cd /var/www/html
  sudo wget https://wordpress.org/latest.tar.gz
  sudo tar -xzvf latest.tar.gz
  sudo rm -f latest.tar.gz
  
  # Set permissions for WordPress
  sudo chown -R apache:apache /var/www/html/wordpress
  sudo chmod -R 755 /var/www/html/wordpress
  
  # Create Apache Virtual Host for WordPress
  sudo bash -c 'cat <<EOF > /etc/httpd/conf.d/wordpress.conf
  <VirtualHost *:80>
      DocumentRoot /var/www/html/wordpress
      <Directory /var/www/html/wordpress>
          AllowOverride All
          Require all granted
      </Directory>
      ErrorLog /var/log/httpd/wordpress-error.log
      CustomLog /var/log/httpd/wordpress-access.log combined
  </VirtualHost>
  EOF'
  
  # Restart Apache to apply the new virtual host configuration
  sudo systemctl restart httpd
  
  # Start and enable MariaDB
  sudo systemctl start mariadb
  sudo systemctl enable mariadb
  
  # Secure MariaDB installation (e.g., setting root password)
  sudo mysql_secure_installation <<EOF
  
  y
  password
  password
  y
  y
  y
  y
  EOF
  
  # Create a WordPress database and user
  sudo mysql -u root -ppassword <<EOF
  CREATE DATABASE ${DB_NAME} CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
  CREATE USER '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}';
  GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'localhost';
  FLUSH PRIVILEGES;
  EXIT;
  EOF
  
  # Configure WordPress
  sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
  sudo sed -i "s/database_name_here/${DB_NAME}/" /var/www/html/wordpress/wp-config.php
  sudo sed -i "s/username_here/${DB_USER}/" /var/www/html/wordpress/wp-config.php
  sudo sed -i "s/password_here/${DB_PASS}/" /var/www/html/wordpress/wp-config.php
  
  # Set final permissions
  sudo chown -R apache:apache /var/www/html/wordpress
  sudo chmod -R 755 /var/www/html/wordpress
  
  # Restart Apache to ensure everything is applied correctly
  sudo systemctl restart httpd
  
  # Script completed
  echo "WordPress installation completed. Access your site to complete the setup."

  ```

---
### 8. Set Up Application Load Balancer (ALB)

8.1 **Create an Application Load Balancer**  
- In the EC2 Dashboard, go to "Load Balancers".
- Click "Create Load Balancer" -> "Application Load Balancer".
- Name: `MyALB`
- Scheme: `internet-facing`
- Listeners: `HTTP:80`
- VPC: `MyVPC`
- Availability Zones: Select both public subnets.
- Security groups: ALB-SG
- Select a target group created in step 6 (MyTargetGroup)
- Create load balancer

---
### 9. Create Auto Scaling Group

9.1 **Create an Auto Scaling Group from Launch Template**  
- In the EC2 Dashboard, go to "Auto Scaling Groups".
- Click "Create Auto Scaling Group".
- Name: `MyASG`
- Launch Template: Select `MyLaunchTemplate`.
- VPC: `MyVPC`
- Subnets: Select both public subnets.
- Click attach to an existing load balancer
- Select load balancer (`MyALB)
- In EC2 health checks: Check Turn on Elastic Load Balancing health checks
- seclect your capacity (Desired capacity = 2, Min desired capacity = 2, Max desired capacity = 4)
- Click Next, Review and Create Auto Scaling Group

---
### 10. Test the Setup

10.1 **Access the Wordpress Application via ALB DNS**  
- Find the DNS name of your ALB under "Load Balancers".
- Open the DNS name in a browser to verify that your web server is running and serving the content.
- **Note:** You’ll notice that accessing the EC2 instances directly via their public IP addresses won’t work. This is by design, as we’ve configured the Security Group to only allow traffic from the ALB. This ensures that your instances are secure and only reachable through the Load Balancer.

---
### Troubleshooting

- **Application Not Accessible via ALB DNS:**
  - Check that ALB Security Group allows inbound HTTP traffic on port 80 from `0.0.0.0/0`.
  - Verify Target Group health; ensure instances are healthy.
  - Ensure ALB is associated with correct public subnets and that they have a route to the Internet Gateway.

- **Instances Not Scaling Properly:**
  - Check Auto Scaling Group settings (desired capacity, min/max instance count).
  - Verify the Launch Template (correct AMI, instance type, user data script).
  - Ensure AWS account hasn’t reached instance quotas for the region.

- **Instances Accessible via Public IP:**
  - Double-check Security Group settings; EC2 instances should only allow inbound traffic from ALB Security Group.
  - Ensure no Elastic IPs are assigned to instances.

- **WordPress Setup Page Not Loading:**
  - Verify PHP and MySQL are correctly installed and configured.
  - Check user data script in Launch Template for errors during WordPress setup.
  - Review instance logs for any startup errors.

- **Slow Application Performance:**
  - Ensure EC2 instance type has sufficient resources (CPU, memory, network).
  - Verify database server capacity and performance.
  - Check for network latency; distribute application across multiple Availability Zones.

---
## Conclusion

This project successfully sets up a scalable web application environment on AWS using best practices. The infrastructure is designed for high availability and security, with the capability to scale automatically based on traffic demands.

---

### Note on NAT Gateway and Security

You might notice that we didn't include a NAT Gateway in this setup. Why, you ask? Well, as much as we love spending money on AWS services, we decided to skip the NAT Gateway to avoid those delightful hourly charges. After all, who wouldn't want to save a bit of cash, right?

Instead, we're opting for a more *budget-friendly* approach by using Security Groups to restrict traffic. Only our trusty ALB is allowed to communicate with the EC2 instances. So, even without a NAT Gateway, your instances are safe and sound, shielded by the mighty Security Group that allows traffic *only* from the ALB. It’s almost like having a guard dog that only listens to one person—cost-effective and secure for practice!

---
