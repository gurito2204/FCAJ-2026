---
title: "Network, Permissions & Secrets Setup"
date: 2026-06-15
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

This article guides you through preparing a standard virtual private network environment (**Amazon VPC**) (VPC Multi-AZ), configuring encryption keys (**AWS KMS**), storing sensitive information in **AWS Secrets Manager**, configuring **Security Groups**, and assigning **IAM** permissions.

---

### 1. Prerequisites
* **AWS Account:** Administrator privileges.
* **AWS Region:** Select `us-east-1an`.
* **Personal Tools:** Pre-installed AWS CLI, Git, and SSH Client.

---

### 2. Network Setup (Amazon VPC Multi-AZ)

We need to build a secure tiered network structure using 6 subnets distributed across 2 Availability Zones.

#### Step 1: Network Initialization (VPC & Subnets)

1. Access **AWS Console** -> **VPC** -> Select **Create VPC**.
2. Settings:
   * **VPC settings:** Select **VPC and more** (automatically creates Subnets and Routing).
   * **Name tag auto-generation:** `shopsflow-vpc`
   * **IPv4 CIDR block:** `10.0.0.0/16`
   * **Number of Availability Zones (AZs):** `2`
   * **Number of Public subnets:** `2`
   * **Number of Private subnets:** `2`
   * **NAT gateways:** Select **In 1 AZ** (or **1 per AZ** for high availability).
3. Select **Create VPC**.

![VPC Creation Result](vpc.jpg)

#### Step 1: Network Initialization (VPC & Subnets)

##### 1.1. VPC Initialization
1. Access **AWS Console** -> **VPC** -> Select **Create VPC**.
2. Settings:
   * **VPC settings:** Select **VPC and more** (automatically creates Subnets and Routing).
   * **Name tag auto-generation:** `shopsflow-vpc`
   * **IPv4 CIDR block:** `10.0.0.0/16`
   * **Number of Availability Zones (AZs):** `2`
   * **Number of Public subnets:** `2`
   * **Number of Private subnets:** `2`
   * **NAT gateways:** Select **In 1 AZ** (or **1 per AZ** for high availability).
3. Select **Create VPC**.

![VPC Creation Result](/images/5-Workshop/5.2-Prerequisite/vpc.jpg)

---

##### 1.2. Create 4 Subnets (Distributed across 2 Availability Zones)
1. Access **VPC** -> **Subnets** -> Select **Create subnet**.
2. **VPC ID:** Select `shopsflow-vpc`.
3. Configure information for each Subnet:
   * **Subnet 1 (Public AZ 1):** Name: `shopsflow-public-1` | AZ: `ap-southeast-1a` | CIDR: `10.0.1.0/24`
   * **Subnet 2 (Public AZ 2):** Name: `shopsflow-public-2` | AZ: `ap-southeast-1b` | CIDR: `10.0.2.0/24`
   * **Subnet 3 (Private AZ 1):** Name: `shopsflow-private-1` | AZ: `ap-southeast-1a` | CIDR: `10.0.3.0/24`
   * **Subnet 4 (Private AZ 2):** Name: `shopsflow-private-2` | AZ: `ap-southeast-1b` | CIDR: `10.0.4.0/24`
4. Select **Create subnets**.
5. After creation, check the 2 Public Subnets (`shopsflow-public-1`, `shopsflow-public-2`) -> Select **Actions** -> **Edit subnet settings** -> Check **Enable auto-assign public IPv4 address** so that resources launched in these subnets automatically receive a Public IP.

![Create Public Subnet 1](/images/5-Workshop/5.2-Prerequisite/public_subnet1.jpg)
![Create Public Subnet 2](/images/5-Workshop/5.2-Prerequisite/public_subnet2.jpg)
![Create Private Subnet 1](/images/5-Workshop/5.2-Prerequisite/private_subnet1.jpg)
![Create Private Subnet 2](/images/5-Workshop/5.2-Prerequisite/private_subnet2.jpg)

---

#### Step 2: Configure Security Groups

##### 2.1. Create Security Group for ALB (alb-sg)
1. Access **EC2 Console** -> **Security Groups** -> Select **Create security group**.
2. General settings:
   * **Security group name:** `alb-sg`
   * **VPC:** Select `shopsflow-vpc`
3. Configure **Inbound Rules**:
   * **Rule 1:** Type: **HTTP (80)** | Source: `0.0.0.0/0` (Anywhere)
   * **Rule 2:** Type: **HTTPS (443)** | Source: `0.0.0.0/0` (Anywhere)
4. Select **Create security group**.

![ALB Security Group Creation Result](/images/5-Workshop/5.2-Prerequisite/alb-sg.jpg)
![ALB Configuration](/images/5-Workshop/5.2-Prerequisite/alb.jpg)

##### 2.2. Create Security Group for ECS (ecs-sg)
1. Select **Create security group**.
2. General settings:
   * **Security group name:** `ecs-sg`
   * **VPC:** Select `shopsflow-vpc`
3. Configure **Inbound Rules**:
   * Type: **Custom TCP** | Port range: `8080` | Source: Select `alb-sg`
4. Select **Create security group**.

![ECS Security Group Creation Result](/images/5-Workshop/5.2-Prerequisite/ecs-sg.jpg)

##### 2.3. Create Security Group for RDS (rds-sg)
1. Select **Create security group**.
2. General settings:
   * **Security group name:** `rds-sg`
   * **VPC:** Select `shopsflow-vpc`
3. Configure **Inbound Rules**:
   * Type: **PostgreSQL** | Port range: `5432` | Source: Select `ecs-sg`
4. Select **Create security group**.

![RDS Security Group Creation Result](/images/5-Workshop/5.2-Prerequisite/rds-sg.jpg)

#### Step 3: Create Internet Gateway (IGW), NAT Gateway & Network Routing

##### 3.1. Initialize and Attach Internet Gateway (IGW)
1. Access **AWS Console** -> **VPC** -> **Internet gateways** -> Select **Create internet gateway**.
2. Settings:
   * **Name tag:** `shopsflow-igw`
3. Select **Create internet gateway**.
4. On the details screen of the newly created IGW, select **Actions** -> **Attach to VPC**.
5. Attachment configuration:
   * **VPC:** Select `shopsflow-vpc`
6. Select **Attach internet gateway**.

---

##### 3.2. Initialize 02 NAT Gateways (Ensure High Availability)
* **Create NAT Gateway for AZ 1:**
  1. Access **VPC** -> **NAT gateways** -> Select **Create NAT gateway**.
  2. Settings:
     * **Name:** `shopsflow-nat-gw-1`
     * **Subnet:** Select `shopsflow-public-1` (Public Subnet)
     * **Elastic IP allocation ID:** Select **Allocate Elastic IP**
  3. Select **Create NAT gateway**.

* **Create NAT Gateway for AZ 2:**
  1. Select **Create NAT gateway**.
  2. Settings:
     * **Name:** `shopsflow-nat-gw-2`
     * **Subnet:** Select `shopsflow-public-2` (Public Subnet)
     * **Elastic IP allocation ID:** Select **Allocate Elastic IP**
  3. Select **Create NAT gateway**.

*(Note: The NAT Gateway initialization process may take 2–3 minutes to transition to the Available state).*

---

##### 3.3. Configure Route Tables
* **Configure Route for Public Subnets (Point to IGW):**
  1. Access **VPC** -> **Route tables** -> Select the Route Table associated with the Public Subnets (`shopsflow-public-1`, `shopsflow-public-2`).
  2. Open the **Routes** tab -> Select **Edit routes** -> Select **Add route**.
  3. Settings:
     * **Destination:** `0.0.0.0/0`
     * **Target:** Select **Internet Gateway** -> Point to `shopsflow-igw`
  4. Select **Save changes**.

* **Configure Route for Private Subnet 1 (Point to NAT 1):**
  1. Select the Route Table of `shopsflow-private-1` -> Open the **Routes** tab -> Select **Edit routes** -> **Add route**.
  2. Settings:
     * **Destination:** `0.0.0.0/0`
     * **Target:** Select **NAT Gateway** -> Point to `shopsflow-nat-gw-1`
  3. Select **Save changes**.

* **Configure Route for Private Subnet 2 (Point to NAT 2):**
  1. Select the Route Table of `shopsflow-private-2` -> Open the **Routes** tab -> Select **Edit routes** -> **Add route**.
  2. Settings:
     * **Destination:** `0.0.0.0/0`
     * **Target:** Select **NAT Gateway** -> Point to `shopsflow-nat-gw-2`
  3. Select **Save changes**.


#### Step 4: Set up Route Tables

##### 4.1. Public Route Table (For Public Subnets)
1. Access **AWS Console** -> **VPC** -> **Route tables** -> Select **Create route table**.
2. Settings:
   * **Name:** `shopsflow-public-rt`
   * **VPC:** Select `shopsflow-vpc`
3. Select **Create route table**.
4. Route configuration:
   * Open the **Routes** tab -> Select **Edit routes** -> Select **Add route**.
   * **Destination:** `0.0.0.0/0` | **Target:** Select **Internet Gateway** (`shopsflow-igw`)
   * Select **Save changes**.
5. Subnet association:
   * Open the **Subnet associations** tab -> Select **Edit subnet associations**.
   * Check `shopsflow-public-1` and `shopsflow-public-2`.
   * Select **Save associations**.

---

##### 4.2. Private Route Table 1 (For Private Subnet in AZ 1)
1. Select **Create route table**.
2. Settings:
   * **Name:** `shopsflow-private-rt-1`
   * **VPC:** Select `shopsflow-vpc`
3. Select **Create route table**.
4. Route configuration:
   * Open the **Routes** tab -> Select **Edit routes** -> Select **Add route**.
   * **Destination:** `0.0.0.0/0` | **Target:** Select **NAT Gateway** (`shopsflow-nat-gw-1`)
   * Select **Save changes**.
5. Subnet association:
   * Open the **Subnet associations** tab -> Select **Edit subnet associations**.
   * Check `shopsflow-private-1` *(contains ECS Backend and RDS in AZ 1)*.
   * Select **Save associations**.

---

##### 4.3. Private Route Table 2 (For Private Subnet in AZ 2)
1. Select **Create route table**.
2. Settings:
   * **Name:** `shopsflow-private-rt-2`
   * **VPC:** Select `shopsflow-vpc`
3. Select **Create route table**.
4. Route configuration:
   * Open the **Routes** tab -> Select **Edit routes** -> Select **Add route**.
   * **Destination:** `0.0.0.0/0` | **Target:** Select **NAT Gateway** (`shopsflow-nat-gw-2`)
   * Select **Save changes**.
5. Subnet association:
   * Open the **Subnet associations** tab -> Select **Edit subnet associations**.
   * Check `shopsflow-private-2` *(contains ECS Backend and RDS in AZ 2)*.
   * Select **Save associations**.

---

### 4. Security Configuration (KMS & Secrets Manager)

To protect sensitive connection strings and passwords, we use AWS Secrets Manager encrypted by AWS KMS.

#### Step 1: Create AWS KMS Key
1. Access **AWS Console** -> **Key Management Service (KMS)** -> **Customer managed keys** -> **Create key**.
2. Select **Symmetric**, Key usage: **Encrypt and decrypt**.
3. Alias: `shopsflow-kms-key`. Select **Create**.

#### Step 2: Initialize Secret in Secrets Manager
1. Access **Secrets Manager** -> Select **Store a new secret**.
2. **Secret type:** Select **Other type of secret**.
3. Enter the key/value pairs for database and JWT information:
   * Key: `SPRING_DATASOURCE_PASSWORD` / Value: `ShopsflowPass123!`
   * Key: `JWT_SECRET` / Value: `ThayTheBangChuoiSecretCucKyDaiVaMat1234567890!`
4. **Encryption key:** Select the exact KMS key `shopsflow-kms-key` just created.
5. Secret name: `shopsflow/production/secrets`. Select **Store**.

---

### 5. Set up Virtual Firewall (Security Groups)

Access **AWS Console** -> **EC2** -> **Security Groups** and sequentially create 3 Security Groups to control tiered access according to the architecture.

#### 5.1. Create ALB Security Group (alb-sg)
1. Select **Create security group**.
2. General settings:
   * **Security group name:** `alb-sg`
   * **VPC:** Select `shopsflow-vpc`
3. Configure **Inbound rules**:
   * **Rule 1:** Type: **HTTP** | Port: `80` | Source: `0.0.0.0/0`
   * **Rule 2:** Type: **HTTPS** | Port: `443` | Source: `0.0.0.0/0`
4. Select **Create security group**.

![ALB Security Group Creation Result](/images/5-Workshop/5.2-Prerequisite/alb-sg.jpg)

---

#### 5.2. Create ECS Security Group (ecs-sg)
1. Select **Create security group**.
2. General settings:
   * **Security group name:** `ecs-sg`
   * **VPC:** Select `shopsflow-vpc`
3. Configure **Inbound rules**:
   * Type: **Custom TCP** | Port range: `8080` | Source: Select Security Group `alb-sg`
4. Select **Create security group**.

*(Note: This configuration ensures the backend only receives requests routed through the Load Balancer).*

![ECS Security Group Creation Result](/images/5-Workshop/5.2-Prerequisite/ecs-sg.jpg)

---

#### 5.3. Create RDS Security Group (rds-sg)
1. Select **Create security group**.
2. General settings:
   * **Security group name:** `rds-sg`
   * **VPC:** Select `shopsflow-vpc`
3. Configure **Inbound rules**:
   * Type: **PostgreSQL** | Port range: `5432` | Source: Select Security Group `ecs-sg`
4. Select **Create security group**.

![RDS Security Group Creation Result](/images/5-Workshop/5.2-Prerequisite/rds-sg.jpg)

---

#### 5.4. Check the Security Groups list
Return to the main Security Groups list screen, verify that the 3 security groups `alb-sg`, `ecs-sg`, and `rds-sg` have been successfully initialized and correctly associated with the VPC.

![Overall Security Groups List](/images/5-Workshop/5.2-Prerequisite/sg.jpg)

---

### 6. Set up IAM Roles for ECS

To ensure security permissions follow the Principle of Least Privilege for the container system, we need to initialize 2 separate IAM Roles: one for the ECS Fargate execution environment (Task Execution Role) and one specifically for the application code (Task Role).

#### Step 1: Create ECS Task Execution Role (shopsflow-ecs-task-execution-role)
This role grants permissions for the ECS environment (Fargate agent) to perform infrastructure tasks such as pulling Docker images from ECR and pushing system logs (container logs) to CloudWatch.

1. Access **AWS Console** -> **IAM Console** -> Select **Roles** in the left menu -> Select **Create role**.
2. Configure Trusted Entity:
   * **Trusted entity type:** Select **AWS service**.
   * **Use case:** Find the **Use cases for other AWS services** section -> Select **Elastic Container Service** -> Select **Elastic Container Service Task** *(Allows ECS tasks to call AWS services on your behalf)*.
   * Select **Next**.
3. Grant Policy permissions:
   * On the **Add permissions** page, search for the keyword: `AmazonECSTaskExecutionRolePolicy`.
   * Check the `AmazonECSTaskExecutionRolePolicy` policy (an AWS managed policy).
   * Select **Next**.
4. Complete initialization:
   * **Role name:** `shopsflow-ecs-task-execution-role`
   * **Description:** `Role allowing Fargate to pull images from ECR and push logs to CloudWatch`
   * Select **Create role**.

---

#### Step 2: Create ECS Task Role (shopsflow-ecs-task-role)
This role grants permissions to the Application Code itself running inside the container. Any AWS SDK calls from the Spring Boot code (such as writing files to S3, reading environment variables from Secrets Manager, sending SQS messages) will use permissions from this role.

1. Return to **IAM Console** -> Select **Roles** -> Select **Create role**.
2. Configure Trusted Entity:
   * **Trusted entity type:** Select **AWS service**.
   * **Use case:** Select **Elastic Container Service** -> Select **Elastic Container Service Task**.
   * Select **Next**.
3. Grant Policy permissions (depending on the actual needs of the Shopsflow backend):
   * *Example 1:* If the app needs to interact with an S3 bucket to store static files, find and check an S3-related policy (e.g., `AmazonS3FullAccess` or a custom policy).
   * *Example 2:* If the app needs to fetch configuration information from Parameter Store or Secrets Manager, check the policy granting `ssm:GetParameters` or `secretsmanager:GetSecretValue` permissions.
   * *(Note: If the current application only interacts with RDS using a standard user/password and does not call any other AWS APIs, you may choose not to check any policies and add them later when needed).*
   * Select **Next**.
4. Complete initialization:
   * **Role name:** `shopsflow-ecs-task-role`
   * **Description:** `Role granting permissions to Application Code inside the container`
   * Select **Create role**.