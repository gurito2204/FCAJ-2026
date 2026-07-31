---
title: "Network Setup, Access Control & Secrets"
date: 2026-06-15
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

This section guides you through preparing the **Amazon VPC** network topology spanning multiple Availability Zones (Multi-AZ), creating custom encryption keys with **AWS KMS**, securely storing credentials in **AWS Secrets Manager**, configuring **Security Groups**, and establishing access permissions via **IAM**.

---

### 1. Prerequisites
* **AWS Account:** Full administrative permissions.
* **AWS Region:** Singapore (`ap-southeast-1`).
* **Workstation Tools:** AWS CLI configured, Git, and an SSH Client.

---

### 2. Network Configuration (Amazon VPC Multi-AZ)

We will build a secure, isolated multi-tier network topology utilizing 6 subnets distributed across 2 Availability Zones (AZs) `ap-southeast-1a` and `ap-southeast-1b`.

#### Step 1: Initialize VPC
1. Navigate to **AWS Console** -> **VPC** -> Choose **Your VPCs** -> Click **Create VPC**.
2. Configure settings:
   * **VPC settings:** Select **VPC only**.
   * **Name tag:** `shopsflow-vpc`
   * **IPv4 CIDR block:** `10.0.0.0/16`
3. Click **Create VPC**.

#### Step 2: Create Subnets
Navigate to **VPC** -> **Subnets** -> Click **Create subnet**. Select the `shopsflow-vpc`. Provision the following 6 subnets:

1. **Public Subnet 1 (AZ1):** CIDR `10.0.1.0/24`, AZ: `ap-southeast-1a`, Name: `shopsflow-public-1`
2. **Public Subnet 2 (AZ2):** CIDR `10.0.2.0/24`, AZ: `ap-southeast-1b`, Name: `shopsflow-public-2`
3. **Private App Subnet 1 (AZ1):** CIDR `10.0.3.0/24`, AZ: `ap-southeast-1a`, Name: `shopsflow-private-app-1`
4. **Private App Subnet 2 (AZ2):** CIDR `10.0.4.0/24`, AZ: `ap-southeast-1b`, Name: `shopsflow-private-app-2`
5. **Private DB Subnet 1 (AZ1):** CIDR `10.0.5.0/24`, AZ: `ap-southeast-1a`, Name: `shopsflow-private-db-1`
6. **Private DB Subnet 2 (AZ2):** CIDR `10.0.6.0/24`, AZ: `ap-southeast-1b`, Name: `shopsflow-private-db-2`

#### Step 3: Create Internet Gateway (IGW) & NAT Gateway
1. **Create Internet Gateway:**
   * Navigate to **VPC** -> **Internet gateways** -> **Create internet gateway**.
   * Name: `shopsflow-igw`. Click **Create**.
   * Select Actions -> **Attach to VPC** -> Choose `shopsflow-vpc`.
2. **Create NAT Gateway:**
   * Navigate to **VPC** -> **NAT gateways** -> **Create NAT gateway**.
   * Name: `shopsflow-nat-gw`.
   * **Subnet:** Select `shopsflow-public-1` (Must reside in a Public subnet).
   * **Elastic IP allocation ID:** Click **Allocate Elastic IP** to assign a static public IP for the NAT gateway.
   * Click **Create NAT gateway** and wait for the status to transition to *Available*.

#### Step 4: Configure Route Tables
1. **Public Route Table (For Public Subnets):**
   * Navigate to **Route tables** -> **Create route table**. Name: `shopsflow-public-rt`, VPC: `shopsflow-vpc`.
   * Under the **Routes** tab -> Click **Edit routes** -> Add route: Destination `0.0.0.0/0`, Target: **Internet Gateway** (`shopsflow-igw`).
   * Under the **Subnet associations** tab -> Click **Edit subnet associations** -> Associate `shopsflow-public-1` and `shopsflow-public-2`.
2. **Private Route Table (For EC2 Backend Subnets):**
   * Create route table: Name: `shopsflow-private-rt`.
   * Under the **Routes** tab -> Add route: Destination `0.0.0.0/0`, Target: **NAT Gateway** (`shopsflow-nat-gw`).
   * Under **Subnet associations** -> Associate `shopsflow-private-app-1` and `shopsflow-private-app-2`.
3. **DB Route Table (Fully Isolated):**
   * Create route table: Name: `shopsflow-db-rt` (No routes pointing to the Internet).
   * Under **Subnet associations** -> Associate `shopsflow-private-db-1` and `shopsflow-private-db-2`.

---

### 3. Encryption & Credentials Management (KMS & Secrets Manager)

To secure our backend database credentials and JSON Web Token secrets, we utilize AWS Secrets Manager encrypted with a customer-managed AWS KMS Key.

#### Step 1: Create AWS KMS Customer Managed Key
1. Navigate to **AWS Console** -> **Key Management Service (KMS)** -> **Customer managed keys** -> Click **Create key**.
2. Select **Symmetric**, Key usage: **Encrypt and decrypt**.
3. Alias: `shopsflow-kms-key`. Click **Create**.

#### Step 2: Store Secrets in Secrets Manager
1. Navigate to **Secrets Manager** -> Click **Store a new secret**.
2. **Secret type:** Select **Other type of secret**.
3. Input the database and JWT key/value pairs:
   * Key: `SPRING_DATASOURCE_PASSWORD` / Value: `ShopsflowPass123!`
   * Key: `JWT_SECRET` / Value: `ThayTheBangChuoiSecretCucKyDaiVaMat1234567890!`
4. **Encryption key:** Choose the newly created KMS key `shopsflow-kms-key`.
5. Secret name: `shopsflow/production/secrets`. Click **Store**.

---

### 4. Firewall Settings (Security Groups)

We will configure 3 Security Groups to establish a secure tiered architecture:

1. **ALB Security Group (`shopsflow-alb-sg`):**
   * Inbound: Allow `HTTP` (Port 80) and `HTTPS` (Port 443) from anywhere (`0.0.0.0/0`).
2. **EC2 Security Group (`shopsflow-ec2-sg`):**
   * Inbound 1: Allow `TCP` (Port 8080) with Source set to the `shopsflow-alb-sg` Security Group (Restricts application backend requests to only those routed through the Load Balancer).
   * Inbound 2: Allow `SSH` (Port 22) from your workstation's static IP (or rely on AWS Systems Manager Session Manager).
3. **RDS Security Group (`shopsflow-rds-sg`):**
   * Inbound: Allow `PostgreSQL` (Port 5432) with Source set to the `shopsflow-ec2-sg` Security Group.

---

### 5. Creating IAM Role for EC2

1. Navigate to **IAM** -> **Roles** -> **Create role** -> Select **EC2** as the trusted entity.
2. Attach the following managed policies:
   * `CloudWatchAgentServerPolicy`: Grants the agent permission to publish logs and metrics to CloudWatch.
   * `AmazonS3FullAccess`: Grants S3 read/write permissions for backup storage.
3. Attach an Inline Policy allowing the instances to retrieve secrets:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "secretsmanager:GetSecretValue",
           "kms:Decrypt"
         ],
         "Resource": "*"
       }
     ]
   }
   ```
4. Role name: `ShopsflowEC2Role`. Click **Create role**.