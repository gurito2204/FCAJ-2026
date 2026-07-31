---
title: "Introduction & Architecture"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

### 1. Project Concept & Objectives

#### Context & Problem Statement
The **Shopsflow** system is a complete full-stack e-commerce application including a Customer storefront for searching, shopping products, and online payment via the VNPay gateway, combined with an Admin Portal to manage product catalogs, track orders, manage inventory, and view sales analytics.

The target customers are small and medium-sized businesses (SMBs) and traditional retail store owners who need to transition to an online environment with optimized costs, retaining full control over source code and databases without depending on third-party SaaS platforms.

The system addresses the following problems:
* **Reduce Downtime & Deployment Risk:** Eliminate environment conflicts (such as library version mismatch between local machine and server) using containerization (Docker).
* **Data Security:** Prevent customer data leakage by hosting the database and application servers inside isolated Private Subnets.
* **Data Resiliency:** Automate periodic PostgreSQL database dumps and backups to S3 over private network paths, preventing data loss in the event of server hardware failure.
* **Centralized Observability:** Aggregate all application logs and system metrics on CloudWatch for easy troubleshooting.

#### Specific Objectives
* **Desired Outputs:**
  * **Frontend Web:** React + Vite Single Page Application (SPA) deployed statically on Amazon S3 and distributed via Amazon CloudFront CDN.
  * **Backend API:** Spring Boot RESTful API running on EC2 instances inside Private Subnets, managed automatically by an Auto Scaling Group (ASG) behind an Application Load Balancer (ALB).
  * **RDS Database:** PostgreSQL Database running in Multi-AZ Standby mode, with public access disabled.
  * **Security & Encryption:** Utilizing AWS Secrets Manager and KMS to manage database credentials and JWT tokens. Protecting CloudFront with AWS WAF against common web threats.
  * **Monitoring & Backup System:** Centralized monitoring and logging on CloudWatch. RDS PostgreSQL backups compressed and uploaded securely to S3 via an S3 VPC Gateway Endpoint.

#### Program Alignment
The project utilizes advanced and foundational AWS services: **VPC**, **EC2**, **RDS**, **CloudFront**, **WAF**, **S3**, **Secrets Manager**, **KMS**, **CloudWatch**, and **IAM**. The infrastructure aligns with AWS Well-Architected Framework principles of high availability and security, making it an ideal practical lab for students in the First Cloud Journey (FCJ) program.

---

### 2. Architecture & Technical Design

#### Architecture Diagram

Below is the architecture diagram detailing the tiered layout and data flows of the Shopsflow application when deployed on AWS:

![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.jpg)

#### Service Selection Rationale

* **Amazon CloudFront & Amazon S3 (Frontend):**
  * *Rationale:* Hosting static files (HTML/JS/CSS/Images) on S3 and distributing them via CloudFront offloads compute demands from EC2 instances, accelerates page loading globally via Edge Location Caching, and reduces operational costs. Integrating AWS WAF protects the edge distribution from DDoS attacks and SQL injection.
* **AWS ALB & Auto Scaling Group (EC2 Backend):**
  * *Rationale:* ALB distributes incoming API traffic to EC2 instances in the Private Subnets across two Availability Zones (AZs), ensuring high availability. The Auto Scaling Group automatically provisions/terminates instances based on CPU utilization, preventing downtime during traffic spikes.
* **Amazon RDS PostgreSQL Multi-AZ:**
  * *Rationale:* Multi-AZ mode automatically replicates transaction data to a hot standby instance in a separate AZ. In the event of a primary AZ outage, RDS triggers an automated failover without modifying the backend connection string.
* **AWS Secrets Manager & KMS:**
  * *Rationale:* Centrally manages sensitive configurations (Database Password, JWT Secret) encrypted via AWS KMS. EC2 instances securely fetch credentials at runtime using API calls rather than storing them in plain text configuration files.
* **VPC Gateway Endpoint (S3):**
  * *Rationale:* Enables EC2 instances in the Private Subnet to connect directly to S3 over the AWS internal backbone for database backup uploads, eliminating NAT Gateway data processing charges for large database dumps.
* **NAT Gateway:**
  * *Rationale:* Permits resource outbound connectivity from the Private Subnets (e.g., to fetch software packages or call external VNPay payment APIs) while blocking unsolicited inbound connections from the Internet.

#### Security & Access Control
* **IAM Instance Profile:** Attaches an IAM Role to the EC2 instances, permitting them to retrieve configuration details from Secrets Manager and push diagnostics to CloudWatch.
* **Network Isolation:** EC2 and RDS reside entirely within Private Subnets with no public IPs. Security Groups form a protective chain: `Internet` -> `CloudFront` -> `ALB SG` -> `EC2 SG` -> `RDS SG`.