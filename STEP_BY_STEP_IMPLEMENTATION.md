# Step-by-Step Implementation Guide  
## Resilient and Scalable Web Application on AWS

This guide explains how to build the project **step by step**, in the exact order it should be implemented.  
Each phase depends on the previous one and should be completed sequentially.

---

## Phase 1: Network Setup (VPC, Subnets, Routing)

Start by creating a dedicated **Virtual Private Cloud (VPC)** to isolate all application resources.

- Create a VPC with CIDR block `192.168.0.0/24`.
- Create an **Internet Gateway (IGW)** and attach it to the VPC.
- Select two Availability Zones to ensure high availability:
  - `ap-south-1A`
  - `ap-south-1B`

### Public Subnets
Create public subnets to host internet-facing resources:

- `ap-south-1A` → `192.168.0.0/26`
- `ap-south-1B` → `192.168.0.64/26`

### Private Subnets
Create private subnets for backend resources:

- `ap-south-1A` → `192.168.0.128/26`
- `ap-south-1B` → `192.168.0.192/26`

### Routing Configuration

- Create a **public route table** and add the route:
  - `0.0.0.0/0 → Internet Gateway`
- Associate the public route table with both public subnets.
- Create a **NAT Gateway** in the public subnet (`ap-south-1A`) and assign an Elastic IP.
- Create a **private route table** and add the route:
  - `0.0.0.0/0 → NAT Gateway`
- Associate the private route table with both private subnets.

At this stage:
- Public subnets have direct internet access.
- Private subnets have outbound-only internet access via the NAT Gateway.

---

## Phase 2: Security Group Design

Create three security groups to enforce least-privilege access.

### ALB Security Group
- Allow inbound traffic on port **80** from anywhere (customer-facing).
- Allow inbound traffic on port **8080** only from internal or admin IPs.
- Allow all outbound traffic.

### EC2 Security Group
- Allow inbound traffic on ports **80** and **8080** only from the ALB Security Group.
- Allow all outbound traffic.

### EFS Security Group
- Allow inbound traffic on port **2049 (NFS)** only from the EC2 Security Group.
- No inbound access from the ALB.

These security groups will be reused in later phases.

---

## Phase 3: Amazon EFS Setup

Create an **Amazon Elastic File System (EFS)** as a **regional file system** so it can be accessed from both Availability Zones.

While creating EFS:
- Select the same VPC.
- Create mount targets in the private subnets.
- Attach the **EFS Security Group**.

Before mounting EFS, ensure that **DNS Hostnames are enabled** in the VPC.  
This allows EC2 instances to connect to EFS using its DNS name.

EFS will be used to provide shared persistent storage across Auto Scaling instances.

---

## Phase 4: Base EC2 Instance Preparation and AMI Creation

A **temporary EC2 instance** is launched manually to prepare the base image (golden AMI).
This instance is used only for initial configuration and is terminated after the AMI is created.

### EC2 Launch Configuration (Temporary Instance)

- Launch the instance in a **public subnet**.
- Assign a **public IP address** to allow SSH access.
- Attach the **EC2 Security Group**.
- Allow SSH (port 22) **only from your own IP address**.
- This instance is **not part of Auto Scaling**.

> This approach allows secure administrative access while keeping Auto Scaling instances private.

---

### Application Setup and Amazon EFS Mounting (Inside EC2 Instance)

Connect to the temporary EC2 instance using SSH. All the following steps are performed **inside the instance**.

---

#### Step 1: Update OS and Install Apache HTTP Server

	Update the operating system and install Apache, which will serve the web application.
	
	```bash
	sudo yum update -y
	sudo yum install httpd -y
	sudo systemctl enable httpd
	sudo systemctl start httpd
	sudo systemctl status httpd

#### Step 2: Install Amazon EFS Utilities

	Install the required Amazon EFS utilities to enable secure mounting of EFS using the NFS protocol.
	
	```bash
	sudo yum install -y amazon-efs-utils

#### Step 3: Verify or Create the Apache Web Root Directory
	Ensure that the Apache document root directory exists before mounting EFS.

sudo mkdir -p /var/www/html

#### Step 4: Mount Amazon EFS to the Web Root Directory
	Mount the Amazon EFS file system to /var/www/html so that application content is shared across all EC2 instances.
	
	sudo mount -t efs -o tls fs-xxxx:/ /var/www/html
	
	Replace fs-xxxx with your actual EFS File System ID.
	
	Verify that the file system is mounted successfully:
	
	df -h | grep efs

#### Step 5: Set Correct Permissions for Apache

	Update ownership and permissions so Apache can read and write to the EFS-mounted directory.
	
	sudo chown -R apache:apache /var/www/html
	sudo chmod -R 755 /var/www/html
	
	Create a test file for the main application running on port 80 to verify that Amazon EFS is mounted correctly and Apache is serving content from shared storage:

	```bash
	echo "Main application is running successfully using Amazon EFS" | sudo tee /var/www/html/index.html



#### Step 6: Make the EFS Mount Persistent Across Reboots

	Add the EFS mount entry to /etc/fstab so it mounts automatically after reboot and on Auto Scaling instances.
	
	sudo vi /etc/fstab
	
	Add the following line:
	
	fs-xxxx.efs.ap-south-1.amazonaws.com:/ /var/www/html efs defaults,_netdev 0 0

#### Step 7: Reboot and Validate Persistence

	Reboot the instance to validate that Apache and EFS start automatically.
	sudo reboot
	
	After reconnecting, verify:
	systemctl status httpd
	df -h | grep efs

#### Step 8: Proceed to AMI Creation

	After verifying that Apache is running and Amazon EFS is mounted persistently, create an AMI from the temporary EC2 instance using the EC2 Console → Actions → Image and templates → Create image. AWS briefly 				stops the instance and creates EBS snapshots to capture the OS, Apache configuration, and EFS mount settings. Once the AMI status becomes Available, terminate the temporary EC2 instance. This AMI is then used in the 	Launch Template so all Auto Scaling instances start with Apache installed and EFS mounted automatically.
---

## 🔹 Phase 5: Launch Template and Auto Scaling Group

Create a **Launch Template** that defines how EC2 instances are launched by Auto Scaling.

### Launch Template Configuration
- Select the **custom AMI** created in Phase 4.
- Attach the **EC2 Security Group**.
- Choose the required instance type.
- Add **user data** (stress script if required).

### Application Ports
- **Port 80** → Main application.
- **Port 8080** → Stress / load testing application.

### Auto Scaling Group Creation
- Create an **Auto Scaling Group (ASG)** using the Launch Template.
- Select **only private subnets**.
- Do **not** assign public IP addresses.
- Configure:
  - Desired capacity
  - Minimum capacity
  - Maximum capacity
- Add **CPU-based dynamic scaling policies**.

At this stage, EC2 instances are automatically launched and managed by ASG.

---

## 🔹 Phase 6: Application Load Balancer and Target Groups

### Target Groups
Create target groups **before** creating the Load Balancer.

- Target Group 1:
  - Protocol: HTTP
  - Port: **80**
  - Purpose: Main application
- Target Group 2:
  - Protocol: HTTP
  - Port: **8080**
  - Purpose: Stress testing

### Application Load Balancer (ALB)
Create an **internet-facing Application Load Balancer**.

- Place the ALB in **both public subnets**.
- Attach the **ALB Security Group**.
- Configure listeners:
  - Port **80** → Forward to Target Group (80)
  - Port **8080** → Forward to Target Group (8080)

### ASG Integration
- Attach both target groups to the Auto Scaling Group.
- EC2 instances launched by ASG are automatically registered.
- Health checks are managed through the ALB.

---

## 🔹 Phase 7: Route 53 and Load Testing

### Route 53 Configuration

Use **Amazon Route 53** to map a custom domain name to the **Application Load Balancer (ALB)** DNS name.  
This domain is used to access the customer-facing application.

- **Main Application**  
  Access the application using the Route 53 domain on **port 80**:
  http://<your-domain-name>:80

- **Stress / Load Testing**  
The load-testing application runs on **port 8080** and can be accessed directly using the ALB DNS name:
http://<alb-dns-name>:8080
After accessing this endpoint, load can be generated to increase CPU utilization and **validate Auto Scaling behavior**, including **automatic EC2 instance creation and scaling based on load**.


This separation ensures that customer traffic is routed through the domain name, while internal load testing is performed directly against the ALB without exposing the testing endpoint publicly.



  


