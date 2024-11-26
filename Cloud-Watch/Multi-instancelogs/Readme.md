# Automating EC2 Logs to CloudWatch with Terraform 
 
This project demonstrates how to deploy an EC2 instance, install a sample application, and send application logs to AWS CloudWatch using Terraform. 
 
## Overview 
This setup performs the following tasks: 
 
1.Create an EC2 instance with the necessary IAM role and policies. 
2.Install a sample application. 
 
3.Set up AWS CloudWatch Agent to collect and send logs to CloudWatch. 
 
4.Create CloudWatch Log Group to store the logs. 
 
## Prerequisites 
1.An AWS account. 
2.Terraform installed on your local machine. 
3.AWS CLI configured with appropriate permissions for Terraform to create resources. 
 
## How It Works 
### 1. IAM Role and Policy Setup 
Terraform creates an IAM role (ec2-cloudwatch-role) that allows EC2 instances to assume the role and push logs to CloudWatch. The IAM policy (cloudwatch-log-policy) is attached to this role, granting the necessary permissions to create log groups, streams, and put log events. 
 
### 2. EC2 Instance Setup 
An EC2 instance is provisioned using the specified AMI and instance type. This instance will have a security group to allow SSH (port 22) access and use the IAM instance profile to interact with CloudWatch. 
 
### 3. CloudWatch Logs Configuration 
A CloudWatch Log Group (/ec2/app-logs) is created. The CloudWatch Agent is configured to collect logs from /var/log/messages and send them to the CloudWatch Log Group. The configuration is placed in a config.json file that is deployed via EC2's user data script during boot-up. 
 
### 4. Application Deployment 
A basic Nginx web server is installed, and a sample template (ListRace) is downloaded, extracted, and served on the EC2 instance. 
 
## Explanation of Key Components 
#### 1. IAM Role and Policy: 
 
The aws_iam_role resource creates a role that the EC2 instance will assume. 
The aws_iam_policy defines the permissions to push logs to CloudWatch (including creating log groups, streams, and sending log events). 
The aws_iam_role_policy_attachment attaches the CloudWatch policy to the IAM role. 
 
#### 2. CloudWatch Log Group: 
The aws_cloudwatch_log_group creates a log group where EC2 instance logs will be stored. 
The log group will have a retention policy (7 days in this case) to automatically delete logs after a certain period. 
 
#### 3. User Data Script: 
 
The data.template_file.user_data configures a shell script to install the CloudWatch agent on the EC2 instance. 
It also sets up the agent's configuration to collect logs from /var/log/messages and push them to the CloudWatch log group. 
#### 4. EC2 Instance: 
 
The EC2 instance is launched with the IAM instance profile that grants permissions to push logs to CloudWatch. 
The user_data script is embedded to ensure the CloudWatch agent is installed and configured as part of the instance setup. 
 
 
Terraform Script Breakdown 
 
```hcl 
provider "aws" { 
  region = "us-east-1" # Specify your region 
} 
 
# IAM Role for EC2 
resource "aws_iam_role" "ec2_cloudwatch_role" { 
  name = "ec2-cloudwatch-role" 
  assume_role_policy = jsonencode({ 
    Version = "2012-10-17", 
    Statement = [ 
      { 
        Action = "sts:AssumeRole", 
        Effect = "Allow", 
        Principal = { 
          Service = "ec2.amazonaws.com" 
        } 
      } 
    ] 
  }) 
} 
 
# IAM Policy to Allow Logs to CloudWatch 
resource "aws_iam_policy" "cloudwatch_policy" { 
  name        = "cloudwatch-log-policy" 
  description = "Allow EC2 to write logs to CloudWatch" 
  policy = jsonencode({ 
    Version = "2012-10-17", 
    Statement = [ 
      { 
        Action = [ 
          "logs:CreateLogGroup", 
          "logs:CreateLogStream", 
          "logs:PutLogEvents", 
          "logs:DescribeLogGroups", 
          "logs:DescribeLogStreams" 
        ], 
        Effect   = "Allow", 
        Resource = "*" 
      } 
    ] 
  }) 
} 
 
# Attach the Policy to the Role 
resource "aws_iam_role_policy_attachment" "attach_cloudwatch_policy" { 
  role       = aws_iam_role.ec2_cloudwatch_role.name 
  policy_arn =
aws_iam_policy.cloudwatch_policy.arn 
} 
 
# EC2 Instance Profile 
resource "aws_iam_instance_profile" "ec2_instance_profile" { 
  name = "ec2-instance-profile" 
  role = aws_iam_role.ec2_cloudwatch_role.name 
} 
 
# CloudWatch Log Group 
resource "aws_cloudwatch_log_group" "app_logs" { 
  name              = "/ec2/app-logs" 
  retention_in_days = 7 
} 
 
# Security Group for EC2 
resource "aws_security_group" "ec2_sg" { 
  name_prefix = "ec2-sg" 
  ingress { 
    from_port   = 22 
    to_port     = 22 
    protocol    = "tcp" 
    cidr_blocks = ["0.0.0.0/0"] 
  } 
  egress { 
    from_port   = 0 
    to_port     = 0 
    protocol    = "-1" 
    cidr_blocks = ["0.0.0.0/0"] 
  } 
} 
``` 
# User Data Script for CloudWatch Agent 
data "template_file" "user_data" { 
  template = <<EOT 
#!/bin/bash 
# Update the instance 
```sudo yum update -y 
```
 
# Install Nginx and Unzip (for application setup) 
```sudo yum install -y nginx unzip
``` 
 
# Download and extract the application 
```cd /tmp 
wget https://www.free-css.com/assets/files/free-css-templates/download/page296/listrace.zip 
unzip listrace.zip 
```
 
# Move the application files to the Nginx directory 
```sudo mv listrace-v1.0/ /usr/share/nginx/html/ 
   sudo mv /usr/share/nginx/html/listrace-v1.0 /usr/share/nginx/html/lists
``` 
 
# Start Nginx 
```bash sudo systemctl start nginx 
sudo systemctl enable nginx 
yum install -y amazon-cloudwatch-agent 
cat <<EOF > /opt/aws/amazon-cloudwatch-agent/bin/config.json 
{ 
  "logs": { 
    "logs_collected": { 
      "files": { 
        "collect_list": [ 
          { 
            "file_path": "/var/log/messages", 
            "log_group_name": "${aws_cloudwatch_log_group.app_logs.name}", 
            "log_stream_name": "{instance_id}/messages", 
            "timestamp_format": "%b %d %H:%M:%S" 
          } 
        ] 
      } 
    } 
  } 
} 
EOF 
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a start -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -m ec2 
EOT 
} 
```
# EC2 Instance 
```
resource "aws_instance" "ec2_instance" { 
  ami           = "ami-0c02fb55956c7d316" # Replace with your AMI 
  instance_type = "t2.micro" 
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name 
  security_groups      = [aws_security_group.ec2_sg.name] 
  user_data = data.template_file.user_data.rendered 
  tags = { 
    Name = "EC2-CloudWatch-Logs" 
  } 
} 
```
 
## Application Installation (User Data) 
The user data script installs Nginx, downloads the Listrace template, unzips it, and moves it to the appropriate directory to serve it via Nginx. 
```bash 
sudo yum update -y 
sudo yum install nginx unzip -y 
cd /tmp 
wget https://www.free-css.com/assets/files/free-css-templates/download/page296/listrace.zip 
unzip listrace.zip 
sudo mv listrace-v1.0/ /usr/share/nginx/html/ 
sudo mv /usr/share/nginx/html/listrace-v1.0 /usr/share/nginx/html/lists 
```  
 
Initialize Terraform: Run the following command to initialize Terraform in your directory: 
 

```
terraform init 
```  
Review the Terraform Plan: Execute this command to see what resources Terraform plans to create: 
 
```
terraform plan
``` 
 Apply the Terraform Script: Run the following command to create the resources: 
 
``` 
terraform apply
``` 
 This will prompt you for approval. Type yes to proceed with the creation of resources. 
  
After the EC2 instance is created, you can SSH into it and verify that the application is running: 

```bash 
ssh -i your-key.pem ec2-user@<your-ec2-public-ip> 
``` 
Visit the public IP of the EC2 instance in your browser, and you should see the Nginx page with the Listrace template. 
 
### To view logs in CloudWatch: 
 
1. Go to the CloudWatch console in AWS. 
2. Navigate to Logs and select the /ec2/app-logs log group. 
3. You'll see logs from the EC2 instance here.