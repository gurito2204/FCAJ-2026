---
title: "Monitoring, Backups & Verification"
date: 2026-06-15
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

This section details configuring centralized monitoring via **Amazon CloudWatch**, implementing a secure database backup routine using an **S3 VPC Gateway Endpoint**, and executing test scenarios to validate Auto Scaling policies and system resiliency.

---

### 1. System Monitoring (CloudWatch Agent & Alarm)

Because EC2 instances reside inside an Auto Scaling Group, they automatically install and start the CloudWatch Agent using the User Data configurations defined in the Launch Template.

#### Configuring CloudWatch Alarm for the Auto Scaling Group
1. Navigate to **AWS Console** -> **CloudWatch** -> Choose **Alarms** -> Click **Create alarm**.
2. Click **Select metric** -> Choose **EC2** -> Choose **By Auto Scaling Group** -> Select the `CPUUtilization` metric for the `shopsflow-backend-asg` group.
3. Configure the threshold settings:
   * **Threshold type:** Static
   * **Condition:** Greater than `80%` (Triggered if average CPU utilization exceeds 80% for 2 consecutive evaluation periods of 5 minutes each).
4. **Configure actions:**
   * **Alarm state trigger:** Select **In alarm**.
   * **Send a notification to:** Create a new or select an existing **SNS Topic** subscribed to your email address (Allows AWS to automatically email alarm updates to the administrator).
5. Name the alarm: `shopsflow-asg-high-cpu-alarm`. Click **Create alarm**.

---

### 2. Performing Secure Database Backups via VPC Endpoint

With the S3 VPC Gateway Endpoint established in the previous section, upload traffic transferring database dumps from private EC2 instances to S3 flows entirely over the internal AWS backbone. This protects transactional data and bypasses NAT Gateway data transfer fees.

1. Connect to one of the EC2 instances running inside the Private Subnet using AWS Systems Manager Session Manager or SSH.
2. Navigate to the deployment directory containing the backup script:
   ```bash
   cd ~/shopsflow/deploy/aws
   ```
3. Configure your S3 backup bucket name in the script config variable inside `backup_to_s3.sh` (e.g., `S3_BUCKET="shopsflow-db-backup-999"`).
4. Execute the backup script:
   ```bash
   ./backup_to_s3.sh
   ```
5. **Verify Private Route (VPC Endpoint Check):**
   * Run network traceroute tools or execute a listing command:
     ```bash
     aws s3 ls s3://shopsflow-db-backup-999 --region ap-southeast-1
     ```
   * *Validation:* Verify that the compressed database archive file `.sql.gz` is uploaded to the S3 bucket from the private instance without public IP addresses and without NAT Gateway data processing increments.

---

### 3. Validation Scenarios

#### 🧪 Scenario 1: Edge-to-Backend Routing Validation
* **Procedure:** Open a web browser on your workstation and access the **CloudFront Distribution Domain Name** (e.g., `https://dxxxxx.cloudfront.net`).
* **Expected Result:** 
  * The React Frontend loads successfully from S3 via the CloudFront CDN.
  * When browsing products, CloudFront routes paths starting with `/api/*` to the Application Load Balancer (ALB). The ALB distributes requests (Round Robin) to the two backend EC2 instances in the Private Subnets, fetching data from RDS PostgreSQL to display on screen.

#### 🧪 Scenario 2: Centralized CloudWatch Log Streams
* **Procedure:** Go to **AWS Console** -> **CloudWatch** -> **Log groups** -> Select `/shopsflow/ec2/docker`.
* **Expected Result:** Log streams corresponding to the active EC2 instances in the Auto Scaling Group are listed. As new instances are launched by scaling policies, their container logs stream automatically to this log group.

#### 🧪 Scenario 3: Concurrency Stock Checkout (Optimistic Locking)
* **Procedure:** Execute a concurrency benchmark using Apache Benchmark (`ab`) to send simultaneous checkout requests for a single remaining stock item:
  ```bash
  ab -n 20 -c 10 -p payload.json -T application/json https://dxxxxx.cloudfront.net/api/checkout
  ```
* **Expected Result:** 
  * Only a single transaction executes successfully. All other concurrent transactions are rejected with an HTTP `409 Conflict` response code.
  * Spring Boot logs show an `OptimisticLockingFailureException`. The database inventory stock remains consistent and non-negative.

#### 🧪 Scenario 4: Load Simulation to Test Auto Scaling & Alarm
* **Procedure:** Access one of the private EC2 instances and execute a CPU stress simulation:
  ```bash
  sudo apt-get install -y stress
  stress --cpu 4 --timeout 300s
  ```
* **Expected Result:**
  1. Average **CPU Utilization** for the Auto Scaling Group rises on the CloudWatch Dashboard.
  2. The `shopsflow-asg-high-cpu-alarm` changes state from `OK` to `ALARM`, triggering an email notification via SNS.
  3. The **Auto Scaling Group** detects the load spike -> Triggers a scale-out action -> Automatically provisions a 3rd EC2 instance using the Launch Template.
  4. The 3rd instance initializes, configures the backend services, and automatically registers with the target group `shopsflow-backend-tg` of the ALB to balance incoming requests. Once the stress simulation ends, the ASG scales in back to 2 instances.
