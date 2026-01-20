# AWS-CapStone-Project

Project Overview

This project demonstrates the design and implementation of a resilient, scalable, and highly available web application on AWS. The architecture leverages core AWS services such as VPC, EC2, Auto Scaling, Application Load Balancer (ALB), Elastic File System (EFS), and Route 53 to ensure fault tolerance, secure access, and dynamic scaling across multiple Availability Zones. A custom VPC with public and private subnets is used to isolate resources, where internet-facing traffic is handled by the ALB in public subnets and application workloads run securely in private subnets under an Auto Scaling Group. Shared application data is stored on EFS to maintain consistency across stateless EC2 instances. Route 53 provides reliable DNS-based routing, completing an end-to-end architecture capable of handling variable traffic loads with minimal downtime.

![Capstone+Project-1_page-0001](https://github.com/user-attachments/assets/77917806-d6bc-4250-8c8c-f1ef85ed5564)


