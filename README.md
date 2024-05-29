# Hosting a WordPress Website on AWS

This project demonstrates how to host a WordPress website on AWS, leveraging various AWS services and resources to ensure a scalable, fault-tolerant, and secure environment. The setup includes VPC configuration, EC2 instances, an Application Load Balancer, Auto Scaling, RDS for the database, EFS for shared file storage, and more.

## Architecture Overview

The architecture is designed to provide high availability and reliability across multiple Availability Zones. The main components include:

1. **Virtual Private Cloud (VPC)**: Configured with both public and private subnets across two Availability Zones.
2. **Internet Gateway**: Facilitates connectivity between VPC instances and the internet.
3. **Security Groups**: Acts as a network firewall for the instances.
4. **Public Subnets**: Used for infrastructure components like NAT Gateway and Application Load Balancer.
5. **Private Subnets**: Hosts EC2 instances for enhanced security.
6. **NAT Gateway**: Allows instances in private subnets to access the internet.
7. **Application Load Balancer (ALB)**: Distributes incoming web traffic across multiple EC2 instances.
8. **Auto Scaling Group**: Automatically adjusts the number of EC2 instances based on traffic load.
9. **Elastic File System (EFS)**: Provides a shared file system for web files.
10. **Relational Database Service (RDS)**: Manages the WordPress database.
11. **Certificate Manager**: Secures application communications.
12. **Simple Notification Service (SNS)**: Sends notifications about activities within the Auto Scaling Group.
13. **Route 53**: Manages domain name registration and DNS records.

## Setup Instructions

### Prerequisites

- AWS account
- Domain name registered in Route 53
- IAM user with necessary permissions

### Steps to Deploy

#### 1. VPC and Subnet Configuration

Create a VPC with public and private subnets across two Availability Zones. Ensure you have an Internet Gateway attached to the VPC and route tables configured accordingly.

#### 2. Security Groups

Create security groups to control inbound and outbound traffic for your instances, ALB, and other resources.

#### 3. Launch EC2 Instances

Deploy EC2 instances within the private subnets for the web servers. Use the provided script to configure the instances:

```bash
# Script to install WordPress
sudo su
sudo yum update -y
sudo mkdir -p /var/www/html

# Environment variable for EFS DNS name
EFS_DNS_NAME=fs-064e9505819af10a4.efs.us-east-1.amazonaws.com

# Mount EFS
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# Install Apache, PHP, and MySQL
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

sudo dnf install -y php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server

sudo systemctl start mysqld
sudo systemctl enable mysqld

sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
sudo chown apache:apache -R /var/www/html

# Download and configure WordPress
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php

# Restart Apache
sudo service httpd restart
```

#### 4. Application Load Balancer and Auto Scaling Group

Configure an Application Load Balancer to distribute traffic to the EC2 instances. Create an Auto Scaling Group to manage the number of instances based on demand.

Use the following script for the launch template in the Auto Scaling Group:

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

sudo dnf install -y php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server

sudo systemctl start mysqld
sudo systemctl enable mysqld

# Mount EFS
EFS_DNS_NAME=fs-02d3268559aa2a318.efs.us-east-1.amazonaws.com
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

sudo chown apache:apache -R /var/www/html
sudo service httpd restart
```

#### 5. RDS Database

Set up an RDS instance for the WordPress database. Ensure the database security group allows connections from the EC2 instances.

#### 6. Configure WordPress

Edit the `wp-config.php` file in the WordPress directory to connect to the RDS database. Set the necessary database credentials and parameters.

#### 7. SSL/TLS Configuration

Use AWS Certificate Manager to provision an SSL certificate and configure it with your Application Load Balancer to secure communications.

#### 8. DNS Configuration

Use Route 53 to manage your domain name and create DNS records that point to the Application Load Balancer.

#### 9. Monitoring and Notifications

Set up CloudWatch Alarms and SNS topics to monitor the health and performance of your instances and notify you of any significant events.

## Conclusion

By following the above steps, you can successfully host a WordPress website on AWS, utilizing best practices for security, scalability, and fault tolerance. The provided scripts automate the setup process, ensuring a consistent and repeatable deployment.

