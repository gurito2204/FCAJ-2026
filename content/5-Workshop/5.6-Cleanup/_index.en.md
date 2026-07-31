---
title: "Resource Clean-up"
date: 2026-06-15
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Enterprise architectures utilize resources billed on an hourly consumption basis (such as NAT Gateways and Application Load Balancers). Ensure that you execute the teardown procedures in the specified order below to prevent ongoing charges to your AWS account.

---

### Step-by-Step Teardown Instructions

#### 1. Dismantling CloudFront & AWS WAF
1. Navigate to **AWS Console** -> **CloudFront**.
2. Select your distribution -> Click **Disable** and wait for the status to transition to disabled.
3. Once disabled, check the box next to the distribution and click **Delete**.
4. Navigate to **AWS WAF** -> **Web ACLs** -> Select the Web ACL created for the project -> Click **Delete**.

#### 2. Terminating the Application Load Balancer (ALB)
1. Navigate to **EC2** -> **Load Balancers**.
2. Select `shopsflow-backend-alb` -> Click **Actions** -> **Delete**.
3. Choose **Target Groups** in the left menu -> Select the `shopsflow-backend-tg` Target Group -> Click **Actions** -> **Delete**.

#### 3. Deleting the Auto Scaling Group (ASG) & Launch Template
1. Navigate to **EC2** -> **Auto Scaling Groups**.
2. Select `shopsflow-backend-asg` -> Click **Delete**. 
   * *Note:* Deleting the ASG automatically triggers termination routines (shutdowns) for all EC2 instances inside the Private subnets. Wait for the processes to complete.
3. Choose **Launch Templates** in the left menu -> Select `shopsflow-backend-template` -> Click **Actions** -> **Delete template**.

#### 4. Deleting the Multi-AZ RDS PostgreSQL Instance
1. Navigate to **RDS** -> **Databases**.
2. Select `shopsflow-db` -> Click **Actions** -> **Delete**.
3. In the confirmation pane:
   * Uncheck *Create final snapshot?*
   * Check *I acknowledge...*
   * Type `delete me` in the confirmation input box.
   * Click **Delete** and wait for the database removal to complete.
4. Choose **Subnet groups** in the left menu -> Select `shopsflow-db-subnet-group` -> Click **Delete**.

#### 5. Deleting S3 Buckets
1. Navigate to **S3**.
2. Perform an **Empty** and then a **Delete** operation on the following two buckets:
   * Frontend static bucket: `shopsflow-frontend-static-999`
   * Database backups bucket: `shopsflow-db-backup-999`

#### 6. Deleting Secrets & KMS Keys
1. Navigate to **Secrets Manager** -> Select `shopsflow/production/secrets` -> Click **Actions** -> Select **Delete secret** -> Set the minimum retention period (7 days).
2. Navigate to **KMS** -> Select `shopsflow-kms-key` -> Click **Key actions** -> Choose **Schedule key deletion** -> Set the minimum retention period (7 days).

#### 7. Dismantling Network Infrastructure (NAT Gateway & VPC)
Once all compute and database resources have been deleted, proceed with releasing network resources:
1. **Delete NAT Gateway:**
   * Navigate to **VPC** -> **NAT Gateways**.
   * Select `shopsflow-nat-gw` -> Click **Delete NAT gateway** and wait for its status to transition to *Deleted*.
2. **Release Elastic IP:**
   * Choose **Elastic IPs** in the left menu -> Select the static IP allocated to the NAT gateway -> Click **Actions** -> Select **Release Elastic IP addresses**.
3. **Delete VPC Endpoints:**
   * Choose **Endpoints** in the left menu -> Select your S3 Gateway Endpoint -> Click **Actions** -> Select **Delete VPC endpoints**.
4. **Delete Custom VPC:**
   * Choose **Your VPCs** in the left menu.
   * Select `shopsflow-vpc` -> Click **Actions** -> Choose **Delete VPC**.
   * *Note:* AWS automatically deletes all associated subnets, route tables, network ACLs, and internet gateways when the VPC is deleted.

---

### Reflection & Lessons Learned

#### 1. Challenges Faced
* **Network Topology Complexity:** Dividing the VPC into 6 subnets across 2 Availability Zones required meticulous route table configurations. Initially, private instances failed to connect to the internet to download application repositories due to routing misconfigurations pointing to the NAT Gateway.
* **Secrets Manager & KMS Key Integration:** Configuring the EC2 User Data script to retrieve database credentials and JWT keys automatically encountered permission issues. This required adjusting the KMS Key policy to explicitly allow decrypt permissions for the EC2 IAM Instance Profile role.
* **CloudFront Routing & CORS:** Consolidating the static S3 frontend and the dynamic ALB backend under a single CloudFront distribution using `/api/*` path behaviors required precise cache policy and origin request configuration to avoid CORS issues and header losses at the backend.

#### 2. Lessons Learned
* **Understanding High Availability (HA) Implementations:** This hands-on lab provided practical experience in setting up Application Load Balancers and Auto Scaling Groups to distribute traffic and execute automated self-healing server replacements.
* **Defense in Depth Security Principles:** Gained a deep understanding of network isolation by locking down database nodes and application runtimes inside private subnets, exposing them only via controlled perimeter proxies (WAF, CloudFront, ALB).
* **Cost Optimization with VPC Endpoints:** Learned to deploy Gateway VPC Endpoints for S3 to route large transactional backup files over the private AWS backbone, avoiding expensive data processing fees incurred through public NAT Gateways.

#### 3. Future Improvements
* **Infrastructure as Code (IaC):** Adopt IaC frameworks like **Terraform** or the **AWS CDK** to model the entire architecture as code, enabling repeatable, automated deployments and preventing human configuration errors on the AWS Console.
* **Advanced Serverless/Container Platforms:** Investigate migrating the Spring Boot backend execution environment to **AWS ECS Fargate** to remove EC2 operating system management overhead, aiming for a fully managed Cloud-native architecture.