
Implementation Guide
Resilient and Scalable Web Application on AWS
This guide explains how to build the project step by step, in the exact order it should be implemented. Each phase builds on the previous one and should be completed sequentially.

Phase 1: Network Setup (VPC, Subnets, Routing)
Start by creating a dedicated Virtual Private Cloud (VPC) to isolate all application resources.
Create a VPC with the CIDR block 192.168.0.0/24. This CIDR range will later be divided into public and private subnets across multiple Availability Zones.
Next, create an Internet Gateway (IGW) and attach it to the VPC. Attaching the IGW is mandatory for enabling internet connectivity for public subnets and internet-facing resources such as the Application Load Balancer.
Select two Availability Zones to ensure high availability:
	• ap-south-1A
	• ap-south-1B
Create public subnets in each Availability Zone:
	• ap-south-1A → 192.168.0.0/26
	• ap-south-1B → 192.168.0.64/26
Create private subnets for backend resources:
	• ap-south-1A → 192.168.0.128/26
	• ap-south-1B → 192.168.0.192/26
Create a public route table and add a default route:
0.0.0.0/0 → Internet Gateway
Associate this route table with both public subnets. This makes them internet-accessible.
Create a NAT Gateway in the public subnet in ap-south-1A and assign it an Elastic IP. The NAT Gateway allows outbound internet access for private subnet resources.
Create a private route table and add the route:
0.0.0.0/0 → NAT Gateway
Associate this route table with both private subnets.
At this point:
	• Public subnets have direct internet access
	• Private subnets have outbound-only internet access via the NAT Gateway

Phase 2: Security Group Design
Create three security groups to enforce least-privilege access.
Create an ALB Security Group:
	• Allow inbound traffic on port 80 from anywhere (customer-facing)
	• Allow inbound traffic on port 8080 only from internal or admin IPs
	• Allow all outbound traffic
Create an EC2 Security Group:
	• Allow inbound traffic on ports 80 and 8080 only from the ALB Security Group
	• Allow all outbound traffic
Create an EFS Security Group:
	• Allow inbound traffic on port 2049 (NFS) only from the EC2 Security Group
	• No inbound access from the ALB
These security groups will be reused in later phases.

Phase 3: Amazon EFS Setup
Create an Amazon Elastic File System (EFS) as a regional file system so it can be accessed from both Availability Zones.
While creating EFS:
	• Select the same VPC
	• Place mount targets in the private subnets
	• Attach the EFS Security Group
Before mounting EFS, ensure that DNS Hostnames are enabled in the VPC. This allows EC2 instances to connect to EFS using its DNS name.
EFS will be used later to provide shared persistent storage across Auto Scaling instances.

Phase 4: Base EC2 Instance Preparation and AMI Creation
Launch a temporary EC2 instance manually. This instance is used only to prepare the base image.
Launch the instance:
	• In a private subnet
	• Without a public IP
	• Using the EC2 Security Group
	• Internet access is provided via the NAT Gateway
Inside this EC2 instance, update the OS and install Apache HTTP Server:
sudo yum update -y
sudo yum install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd
sudo systemctl status httpd
Next, install EFS utilities inside the instance:
sudo yum install -y amazon-efs-utils
Mount the EFS file system to the web root directory:
sudo mount -t efs -o tls fs-xxxx:/ /var/www/html
At this stage:
	• Apache serves content from /var/www/html
	• The directory is backed by EFS
	• Content is shared across instances
Make the mount persistent by adding it to /etc/fstab:
fs-xxxx.efs.ap-south-1.amazonaws.com:/ /var/www/html efs defaults,_netdev 0 0
Reboot the instance to verify:
	• Apache starts successfully
	• EFS mounts automatically
Once verified, create an AMI from this instance.
During AMI creation, the instance shuts down temporarily and EBS snapshots are taken.
This AMI becomes the base image for Auto Scaling.

Phase 5: Launch Template and Auto Scaling Group
Create a Launch Template using:
	• The custom AMI
	• EC2 Security Group
	• User data (stress script if required)
The application ports are:
	• Port 80 → Main application
	• Port 8080 → Stress/load testing
Create an Auto Scaling Group (ASG) using this Launch Template.
While creating the ASG:
	• Select only private subnets
	• Do not assign public IPs
	• Configure desired, minimum, and maximum capacity
	• Add CPU-based dynamic scaling policies
The ASG will automatically launch and terminate EC2 instances as load changes.

Phase 6: Application Load Balancer and Target Groups
Create two Target Groups before creating the Load Balancer:
	• Target Group 1 → Port 80 (main application)
	• Target Group 2 → Port 8080 (stress application)
Create an internet-facing Application Load Balancer (ALB):
	• Place it in both public subnets
	• Attach the ALB Security Group
	• Configure listeners for ports 80 and 8080
	• Forward traffic to the respective target groups
Attach both target groups to the Auto Scaling Group.
New EC2 instances launched by the ASG are automatically registered.

Phase 7: Route 53 and Testing
Use Route 53 to map a custom domain name to the ALB DNS name:
http://my-project-alb-xxxx.ap-south-1.elb.amazonaws.com
Access patterns:
	• domain_name:80 → Customer-facing application
	• domain_name:8080 → Internal stress testing
To test auto scaling, generate load manually:
sudo yum install epel-release -y
sudo yum install stress -y
stress --cpu 10
As CPU utilization increases, the Auto Scaling Group launches new instances automatically.

Final Execution Flow
Phase 1 → VPC & Networking
Phase 2 → Security Groups
Phase 3 → EFS
Phase 4 → Base EC2 & AMI
Phase 5 → Launch Template & ASG
Phase 6 → ALB & Target Groups
Phase 7 → Route 53 & Load Testing

Final Architecture Flow
Route 53 → ALB (Public Subnets) → Target Groups → ASG (Private Subnets) → EC2 → EFS
<img width="1480" height="5336" alt="image" src="https://github.com/user-attachments/assets/ad01ffe1-4357-4825-9452-e0fb2608370b" />
