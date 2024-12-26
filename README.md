# Twoge Deployment on AWS

Twoge is a social media platform where users can share their thoughts
exclusively about Doge. This project was developed and deployed using various
AWS services to ensure scalability, security, and efficiency.

## Project Overview

As part of the Codeplatoon DevOps team, the goal was to deploy Twoge on AWS,
utilizing services like:

- Elastic Compute Cloud (EC2)
- Simple Storage Service (S3)
- Identity Access Management (IAM)
- Virtual Private Cloud (VPC)
- Application Load Balancer (ALB)
- Auto Scaling Group (ASG)
- Simple Notification Service (SNS)

Below are the completed milestones, architecture diagram, and documentation for
the deployment process.

---

## Milestones

### 1. Amazon VPC with Public Subnets

- Created a VPC with two public subnets.

### 2. Host Static Files on S3

- Created an S3 bucket to store static files like images and videos.
- Configured the bucket for public read access.

### 3. IAM Role for S3 Access

- Created an IAM role that grants the required permissions for the EC2 instance
  to access the S3 bucket.

### 4. EC2 Instance Deployment

- Launched an EC2 instance using Ubuntu Linux AMI.
- Installed and configured the Twoge application on the EC2 instance.

### 5. Application Load Balancer (ALB)

- Created an ALB to distribute incoming traffic across EC2 instances.
- Configured an HTTP listener rule to forward traffic to the EC2 instances.

### 6. Auto Scaling Group (ASG)

- Created an ASG to automatically scale the number of EC2 instances based on
  traffic demand.
- Integrated the ASG with the ALB for seamless load balancing.

### 7. Simple Notification Service (SNS)

- Configured an SNS topic to send email notifications about ASG scaling
  activities.
- Set up SNS to notify when EC2 instances are added or removed from the ASG.

### 8. Server Shutdown Notifications

- Demonstrated SNS email notifications upon server shutdown.

### 9. Stress Test with ASG Scaling

- Ran a Python-based stress test to simulate high traffic.
- Observed and showcased the ASG scaling in action as new instances were
  launched and terminated.

---

## Architecture Diagram

![Architecture Diagram](images/twoge-architecture-diagram.png)

The architecture includes:

- A secure VPC with public subnets in 2 availability zones.
- IAM policy for service’s access and permissions management.
- S3 bucket for static file hosting.
- EC2 instances managed by an ASG and ALB.
- RDS for private database hosting.
- SNS for notifications.

---

## Commands and Configuration Files

### Terminal Commands

Detailed commands for each step can be found below.

### S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::mikhail-twoge-static-files/*"
    }
  ]
}
```

### IAM Role policy

Provides full access to all buckets via the AWS Management Console.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*", "s3-object-lambda:*"],
      "Resource": "*"
    }
  ]
}
```

### Nginx Configuration

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:9876;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
```

### Twoge Daemon/User-Data Script

```bash
#!/bin/bash
# Update package list and install dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-venv git postgresql-client nginx curl

# Clone the repository
git clone https://github.com/codeplatoon-devops/twoge.git /home/ubuntu/twoge
cd /home/ubuntu/twoge

# Set up Python virtual environment
python3 -m venv venv
source venv/bin/activate

# Install required Python packages
pip install -r requirements.txt

# Create the .env file with the RDS database configuration
echo "SQLALCHEMY_DATABASE_URI=postgresql://postgres:password@mikhail-twoge-database.cdomocywq43e.us-east-2.rds.amazonaws.com/posts" > .env

# Create the systemd service file
cat <<EOF > /etc/systemd/system/twoge.service
[Unit]
Description=Gunicorn instance to serve twoge
Wants=network.target
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/twoge
Environment="PATH=/home/ubuntu/twoge/venv/bin"
ExecStart=/home/ubuntu/twoge/venv/bin/gunicorn app:app -c /home/ubuntu/twoge/gunicorn_config.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd, enable, and start the service
sudo systemctl daemon-reload
sudo systemctl enable twoge
sudo systemctl start twoge

# Configure Nginx with the fetched public IP
cat <<EOF > /etc/nginx/sites-available/twoge
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:9876;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
EOF

# Create symbolic link to enable the Nginx configuration
sudo ln -s /etc/nginx/sites-available/twoge /etc/nginx/sites-enabled/

# Remove the default Nginx configuration to avoid conflicts
sudo rm /etc/nginx/sites-enabled/default

# Test the Nginx configuration for syntax errors
sudo nginx -t

# Restart Nginx to apply the configuration
sudo systemctl restart nginx

# Connect to the database instance and create the 'posts' database if it doesn't already exist
psql -h mikhail-twoge-database.cdomocywq43e.us-east-2.rds.amazonaws.com -U postgres -d postgres -c "DO \$\$ BEGIN IF NOT EXISTS (SELECT FROM pg_database WHERE datname = 'posts') THEN CREATE DATABASE posts; END IF; END \$\$;"

```

---

## Demonstration

1. **Architecture Explanation**: Overview of the VPC, subnets, ALB, and other
   AWS resources.
2. **Application Demo**: Show Twoge running and milestones completed.
3. **AWS Console**: Screenshots and walkthrough of configured services.
4. **Stress Test**: Run and demonstrate ASG scaling in action.

---

## Conclusion

The Twoge application was successfully deployed on AWS, meeting all the
specified milestones. The architecture ensures scalability, security, and
reliability for a seamless user experience.

Feel free to explore the repository for more details!

---

## Repository

[GitHub Repository Link](https://github.com/codeplatoon-devops/twoge)

---

## Author

[Mikhail Waddell]
