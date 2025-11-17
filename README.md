AWS CLI: ALB With Multiple Target Groups (Mobile, Grocery, Home)
🚀 Project Overview

This project demonstrates how to deploy an Application Load Balancer (ALB) with multiple Target Groups using AWS CLI, supporting:

📱 Mobile path

🛒 Grocery path

🏠 Home path

Each service runs on its own EC2 instances, target group, health checks, and listener rules, all behind a single ALB using path-based routing.

🏗️ Architecture Diagram

![Capture (2)](https://github.com/user-attachments/assets/dad7364a-2052-4597-a455-9bacde7267bc)

📌 Prerequisites

AWS CLI configured (aws configure)

VPC ID, Subnet IDs, SG IDs

ALB listener created (for first stage)

AMI ID

EC2 key pair

📘 SECTION 1: MOBILE DEPLOYMENT
✅ Step 1: Create EC2 Instances for Mobile App
aws ec2 run-instances \
    --image-id ami-052064a798f08f0d3 \
    --count 2 \
    --instance-type t3.micro \
    --key-name fortuner \
    --security-group-ids sg-0886d9e47dcd59601 \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
dnf update -y
dnf install -y httpd
echo "<h1>hello mobile loadbalancer page.. </h1>" >> /var/www/html/mobile/index.html
hostname >> /var/www/html/index.html
systemctl start httpd
systemctl enable httpd' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mobile-1}]'

✅ Step 2: Create Target Group
aws elbv2 create-target-group \
    --name mobile-tg-1 \
    --protocol HTTP \
    --port 80 \
    --target-type instance \
    --vpc-id vpc-0a1dcfa6a57f1a059 \
    --health-check-protocol HTTP \
    --health-check-path / \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text

✅ Step 3: Create Application Load Balancer
aws elbv2 create-load-balancer \
    --name application-load-balancer-Nakul \
    --type application \
    --scheme internet-facing \
    --ip-address-type ipv4 \
    --security-groups sg-0886d9e47dcd59601 \
    --subnets subnet-0a5dc029efb8217b3 subnet-0c480ef07e7b38644 subnet-09437e31b1c03074c \
    --query 'LoadBalancers[0].LoadBalancerArn' \
    --output text

✅ Step 4: Create Listener for ALB
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:loadbalancer/app/application-load-balancer-Nakul/a2a00547cf691c3e \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:359210881289:targetgroup/mobile-tg-1/d8e6b38eb6b9be73

✅ Step 5: Register Instances to Mobile Target Group
aws elbv2 register-targets \
     --target-group-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:targetgroup/mobile-tg-1/d8e6b38eb6b9be73 \
     --targets Id=i-0aa2d21414cb884fd Id=i-046e6892a1157d7f3

🔍 Step 6: Health Check (Optional)
aws elbv2 describe-target-health \
     --target-group-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:targetgroup/mobile-tg-1/d8e6b38eb6b9be73

📘 SECTION 2: GROCERY DEPLOYMENT
✅ Step 1: Create Grocery EC2 Instances
aws ec2 run-instances \
    --image-id ami-052064a798f08f0d3 \
    --count 2 \
    --instance-type t3.micro \
    --key-name fortuner \
    --security-group-ids sg-0886d9e47dcd59601 \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
dnf update -y
dnf install -y httpd
mkdir /var/www/html/grocery
echo "<h1>hello grocery loadbalancer page.. </h1>" >> /var/www/html/grocery/index.html
hostname >> /var/www/html/grocery/index.html
systemctl start httpd
systemctl enable httpd' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=grocery-1}]'

✅ Step 2: Create Grocery Target Group
aws elbv2 create-target-group \
    --name grocery-tg-1 \
    --protocol HTTP \
    --port 80 \
    --target-type instance \
    --vpc-id vpc-0a1dcfa6a57f1a059 \
    --health-check-protocol HTTP \
    --health-check-path /grocery \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text

✅ Step 3: Add Listener Rule for Grocery Path
aws elbv2 create-rule \
    --listener-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:listener/app/application-load-balancer-Nakul/bf82d785ec866801/99f533f4ee5a4332 \
    --priority 10 \
    --conditions Field=path-pattern,Values=/grocery* \
    --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:359210881289:targetgroup/grocery-tg-1/49ccdaf510944f0e

✅ Step 4: Attach Instances to Grocery Target Group
aws elbv2 register-targets \
     --target-group-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:targetgroup/grocery-tg-1/85a80f0e888ddfc6 \
     --targets Id=i-0ac69d87ff2ca10d4 Id=i-06a0fdc3a4356c949

✅ Step 5: Modify Listener to Add Forwarding for Grocery
aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:listener/app/application-load-balancer-Nakul/a2a00547cf691c3e/bbafff5612222dad \
    --default-actions '[
        {
            "Type": "forward",
            "ForwardConfig": {
                "TargetGroups": [
                    {"TargetGroupArn": "arn:mobile", "Weight": 1},
                    {"TargetGroupArn": "arn:grocery", "Weight": 1}
                ]
            }
        }
    ]'

📘 SECTION 3: HOME DEPLOYMENT
✅ Step 1: Create Home EC2 Instances
aws ec2 run-instances \
    --image-id ami-052064a798f08f0d3 \
    --count 2 \
    --instance-type t3.micro \
    --key-name fortuner \
    --security-group-ids sg-0886d9e47dcd59601 \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
dnf update -y
dnf install -y httpd
mkdir /var/www/html/home
echo "<h1>hello  home pagr.. </h1>" >> /var/www/html/home/index.html
hostname >> /var/www/html/home/index.html
systemctl start httpd
systemctl enable httpd' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=home-1}]'

✅ Step 2: Create Home Target Group
aws elbv2 create-target-group \
    --name home-tg-1 \
    --protocol HTTP \
    --port 80 \
    --target-type instance \
    --vpc-id vpc-0a1dcfa6a57f1a059 \
    --health-check-protocol HTTP \
    --health-check-path /home \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text

✅ Step 3: Attach Home Instances
aws elbv2 register-targets \
     --target-group-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:targetgroup/home-tg-1/b564ed5520c485c8 \
     --targets Id=i-0ceff682496fd5936 Id=i-0e19633e42d8f616b

✅ Step 4: Add Home Rule to Existing Listener
aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:us-east-1:359210881289:listener/app/application-load-balancer-Nakul/a2a00547cf691c3e/bbafff5612222dad \
    --default-actions '[
        {
            "Type": "forward",
            "ForwardConfig": {
                "TargetGroups": [
                    {"TargetGroupArn": "arn:mobile", "Weight": 1},
                    {"TargetGroupArn": "arn:grocery", "Weight": 1},
                    {"TargetGroupArn": "arn:home", "Weight": 1}
                ]
            }
        }
    ]'

✅ Conclusion

This setup demonstrates:

Multi-target group ALB

Path-based routing (/mobile, /grocery, /home)

Independent EC2 autoscaling endpoints

Proper listener rules and health checks
