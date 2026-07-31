---
title: "Introduction & Architecture"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---
 
### 1. Project Idea & Objectives
 
#### Background & Problem Statement
**Shopsflow** is a complete full-stack e-commerce system consisting of a Customer-facing Storefront for browsing, purchasing products, and checking out online through the VNPay payment gateway, together with an Admin Portal for managing the product catalog, tracking orders, managing inventory, and viewing revenue analytics.
 
The target customers are small and medium-sized businesses (SMBs) and traditional retail shop owners who want to move online at an optimized cost, while retaining full ownership of their source code and database without depending on third-party SaaS platforms.
 
The system addresses the following problems:
* **Reducing downtime and deployment risk:** Eliminates environment conflicts (library version mismatches between local machines and servers) through containerization (Docker), packaging the application into images and running them on **Amazon ECS (Fargate)** instead of installing directly on a server.
* **Data security:** Prevents customer data leaks by placing the backend container tasks and database inside isolated Private Subnets.
* **Data durability:** Automates periodic backups of the PostgreSQL database to avoid data loss in the event of a failure.
* **Centralized monitoring:** Centralizes all application logs and system metrics on CloudWatch for easier troubleshooting.
#### Specific Goals
* **Desired Outputs:**
  * **Frontend Web:** A React + Vite Single Page Application (SPA), deployed as static assets on Amazon S3 and distributed via the Amazon CloudFront CDN.
  * **Backend API:** A Spring Boot RESTful API, packaged as a Docker image, stored on **Amazon ECR (Elastic Container Registry)**, and run as a task on **Amazon ECS using the Fargate launch type** (serverless containers, no EC2 instances to manage), placed in a Private Subnet behind an Application Load Balancer (ALB).
  * **Database RDS:** A PostgreSQL database on Amazon RDS, placed in a Private Subnet with public accessibility fully disabled.
  * **Security & Encryption:** AWS Secrets Manager and KMS are used to store passwords and other sensitive configuration.
  * **Monitoring System:** A centralized dashboard and logs on CloudWatch for both ECS tasks and RDS.
#### Program Fit
The project uses core and advanced AWS services including: **VPC**, **ECS (Fargate)**, **ECR**, **RDS**, **CloudFront**, **S3**, **Application Load Balancer**, **Secrets Manager**, **KMS**, **CloudWatch**, and **IAM**. The infrastructure follows the security and high-availability design principles of the AWS Well-Architected Framework, making it a strong hands-on capstone topic for learners in the First Cloud Journey (FCJ) program.
 
---
 
### 2. Architecture Diagram & Technical Design
 
#### Architecture Diagram
 
Below is the architecture diagram describing the layered structure and data flow of the Shopsflow application when deployed on AWS infrastructure:
 
![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.png)
 
**Main processing flow (numbered on the diagram):**
1. The user accesses the application over the Internet.
2. CloudFront fetches static assets (HTML/JS/CSS) from the S3 FrontEnd Bucket.
3–4. CloudFront forwards API requests through the Internet Gateway to the Application Load Balancer.
5. The ALB routes traffic to the ECS Task (Fargate) inside the Backend Container.
6–7. The backend calls the VNPay API to create a payment URL and receive the payment status.
8. The backend reads/writes user, product, and order data to RDS.
On the CI/CD side: the developer commits code to GitHub, builds the Docker image, and pushes it to **Amazon ECR**; configuration/infrastructure changes are then updated into the VPC.
 
#### Service Selection Rationale
 
* **Amazon CloudFront & Amazon S3 (Frontend):**
  * *Rationale:* Hosting static assets (HTML/JS/CSS/images) on S3 and distributing them via CloudFront completely offloads the backend, improves global page load speed through edge-location caching, and optimizes cost.
* **Amazon ECS (Fargate) & Amazon ECR (Backend):**
  * *Rationale:* The Spring Boot backend is packaged as a Docker image, stored in ECR, and run as a task on ECS Fargate — a serverless container model that removes the need to manually manage, patch, or scale underlying EC2 instances. The ALB distributes traffic to ECS tasks placed in a Private Subnet, and the ECS Service can automatically adjust the number of running tasks based on system load.
* **Amazon RDS PostgreSQL:**
  * *Rationale:* A managed database service that reduces operational overhead (patching, backups, failover) compared to self-managing PostgreSQL on a server, while sitting entirely inside a Private Subnet so it is never exposed to the Internet.
* **AWS Secrets Manager & KMS:**
  * *Rationale:* Centrally stores and manages sensitive information (database password, JWT secret) encrypted with AWS KMS. The ECS Task automatically retrieves temporary credentials at runtime instead of storing plaintext configuration inside the image.
* **Application Load Balancer:**
  * *Rationale:* Distributes traffic coming from the Internet Gateway to the ECS tasks in the Private Subnet, fully separating the public tier from the internal backend tier.
#### Security & IAM (Security & Access Control)
* **IAM Task Role / Execution Role:** An IAM role is attached to the ECS Task, granting permission to read secrets from Secrets Manager, pull images from ECR, and send logs/metrics to CloudWatch — following the principle of least privilege (not an EC2 instance role, since the backend does not run directly on EC2).
* **Network Isolation:** Both the ECS Task and RDS reside entirely within Private Subnets, with no public IP addresses. Security Groups are chained as follows: `Internet` → `CloudFront` → `Internet Gateway` → `ALB SG` → `ECS Task SG` → `RDS SG`.
