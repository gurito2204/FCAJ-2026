---
title: "Deploying Frontend & Backend API"
date: 2026-06-15
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

This section details how to perform the decoupled deployment of the **Web Frontend** (S3 + CloudFront CDN + WAF) and the **Backend API** running on an automated **Application Load Balancer + Auto Scaling Group** configuration inside the Private App Subnets, alongside setting up an **S3 VPC Gateway Endpoint**.

---

### 1. Web Frontend Deployment (S3 + CloudFront CDN + WAF)

We store the React static assets on **Amazon S3** and distribute them using **Amazon CloudFront** CDN with **Origin Access Control (OAC)** to block direct public S3 reads.

#### Step 1: Provision S3 Bucket for Frontend Hosting
1. Navigate to **AWS Console** -> **S3** -> Click **Create bucket**.
2. **Bucket name:** `shopsflow-frontend-static-999` (Use a globally unique name).
3. Keep default settings: **Block all public access** checked (The bucket remains private, allowing access only via CloudFront).
4. Click **Create bucket** and upload your compiled React Frontend static assets (the contents of the `dist/` directory, including `index.html` and `assets/`).

#### Step 2: Establish CloudFront CDN Distribution
1. Navigate to **CloudFront** -> Click **Create distribution**.
2. **Origin:**
   * **Origin domain:** Select the S3 bucket `shopsflow-frontend-static-999.s3.amazonaws.com`.
   * **Origin access:** Select **Origin access control settings (recommended)** -> Click **Create control setting** -> Select **Sign requests (recommended)** and click **Create**.
3. **Default cache behavior:**
   * **Viewer protocol policy:** Select **Redirect HTTP to HTTPS**.
4. **Web Application Firewall (WAF):**
   * Select **Enable security protections** -> Choose **Create Web ACL** (Protects the site from common OWASP Top 10 vulnerabilities at edge).
5. Click **Create distribution**.
6. **Update S3 Bucket Policy:**
   * Once created, CloudFront displays a banner prompting you to update the S3 bucket policy. Copy the policy block.
   * Return to S3 -> Select `shopsflow-frontend-static-999` -> **Permissions** -> **Bucket policy** -> Click **Edit** -> Paste the copied JSON policy (authorizes CloudFront OAC read access to S3 resources) and save.

---

### 2. High Availability Backend Deployment (ALB + ASG)

We will launch a public-facing **Application Load Balancer (ALB)** to distribute API requests to EC2 instances inside the Private Subnets managed via an **Auto Scaling Group (ASG)**.

#### Step 1: Create Application Load Balancer
1. Navigate to **EC2** -> **Load Balancers** -> Click **Create load balancer** -> Choose **Application Load Balancer**.
2. Configure settings:
   * **Load balancer name:** `shopsflow-backend-alb`
   * **Scheme:** **Internet-facing** (ALB acts as the public entry point and proxies requests to private servers).
   * **Network mapping:** Select the custom `shopsflow-vpc`.
   * **Mappings:** Choose the two **Public Subnets** (`shopsflow-public-1` and `shopsflow-public-2`) across different AZs.
3. **Security groups:** Select the `shopsflow-alb-sg`.
4. **Listeners and routing:**
   * Protocol: `HTTP`, Port: `80`.
   * **Default action:** Click **Create target group**:
     * *Target type:* **Instances**
     * *Target group name:* `shopsflow-backend-tg`
     * *Protocol:* `HTTP`, Port: `8080` (Port the Spring Boot application listens on).
     * *VPC:* Select `shopsflow-vpc`.
     * *Health checks:* Path `/api/products` (or your application health endpoint).
     * Click **Create target group**.
   * F5 to refresh target group selection on ALB creation page and select `shopsflow-backend-tg`.
5. Click **Create load balancer** and copy its DNS Name.

#### Step 2: Establish Launch Template for EC2 Backend
The Launch Template defines the baseline configuration for compute instances provisioned by the Auto Scaling Group.

1. Navigate to **EC2** -> **Launch Templates** -> Click **Create launch template**.
2. Configure settings:
   * **Launch template name:** `shopsflow-backend-template`
   * **OS Image (AMI):** **Ubuntu Server 24.04 LTS**.
   * **Instance type:** `t3.micro`.
   * **Key pair:** Select your SSH key pair.
   * **Network settings:** Select Security Group `shopsflow-ec2-sg`.
   * **Advanced details:**
     * **IAM instance profile:** Select the `ShopsflowEC2Role`.
     * **User data (Automated boot configuration script):**
       Paste the following script to automate packages setup, fetch values from Secrets Manager, and launch the Spring Boot backend container:
       ```bash
       #!/bin/bash
       apt-get update -y
       apt-get install -y docker.io awscli jq git
       systemctl enable --now docker
       usermod -aG docker ubuntu

       # Create directory structure for CloudWatch Agent config
       mkdir -p /opt/aws/amazon-cloudwatch-agent/etc/

       # Retrieve and decode Secrets from AWS Secrets Manager using IAM Role credentials
       SECRET_RAW=$(aws secretsmanager get-secret-value --secret-id shopsflow/production/secrets --region ap-southeast-1 --query SecretString --output text)
       DB_PASS=$(echo $SECRET_RAW | jq -r .SPRING_DATASOURCE_PASSWORD)
       JWT_KEY=$(echo $SECRET_RAW | jq -r .JWT_SECRET)

       # Clone repository and configure environment variables
       git clone <YOUR_GITHUB_LINK> /home/ubuntu/shopsflow
       cd /home/ubuntu/shopsflow/deploy/aws
       
       cat <<EOF > .env.aws
       SPRING_DATASOURCE_URL=jdbc:postgresql://<RDS_ENDPOINT>:5432/shopsflow
       SPRING_DATASOURCE_USERNAME=postgres
       SPRING_DATASOURCE_PASSWORD=$DB_PASS
       JWT_SECRET=$JWT_KEY
       APP_SEED_DEMO_DATA=true
       EOF

       # Install docker compose plugin
       mkdir -p ~/.docker/cli-plugins/
       curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
       chmod +x ~/.docker/cli-plugins/docker-compose

       # Launch backend application container
       /usr/local/bin/docker-compose --env-file .env.aws -f docker-compose.aws.yml up -d backend-service
       ```
       *(Make sure to replace `<RDS_ENDPOINT>` with your actual RDS host endpoint address and input your repository URL).*
3. Click **Create launch template**.

#### Step 3: Configure Auto Scaling Group (ASG)
1. Navigate to **EC2** -> **Auto Scaling Groups** -> Click **Create Auto Scaling group**.
2. **Name:** `shopsflow-backend-asg`.
3. **Launch template:** Select `shopsflow-backend-template` -> Click **Next**.
4. **Network:**
   * **VPC:** Select `shopsflow-vpc`.
   * **Subnets:** Associate with **Private App Subnet 1** (`shopsflow-private-app-1`) and **Private App Subnet 2** (`shopsflow-private-app-2`). This isolates backend compute nodes.
5. **Configure advanced options:**
   * Check **Load balancing** -> Select **Attach to an existing load balancer**.
   * Choose Target group `shopsflow-backend-tg` -> Click **Next**.
6. **Group size and scaling policies:**
   * **Desired capacity:** `2` (Runs two instances across separate AZs to ensure active-active high availability).
   * **Min capacity:** `1`, **Max capacity:** `4`.
   * **Scaling policies:** Select **Target tracking scaling policy** -> Metric type: *Average CPU utilization*, Target value: `70` (Triggers scale up operations if average CPU utilization exceeds 70%).
7. Complete reviews and click **Create Auto Scaling group**. ASG will start provisioning two instances inside the Private subnets.

---

### 3. Configuring CloudFront Routing for API Backend

To route both the frontend static website and backend API calls under a single unified domain (and avoid CORS preflight overhead), we add a backend origin mapping to CloudFront.

1. Go to **CloudFront** -> Select your distribution.
2. Under the **Origins** tab -> Click **Create origin**:
   * **Origin domain:** Select the DNS name of the Load Balancer `shopsflow-backend-alb`.
   * **Protocol:** Select `HTTP Only`.
   * Click **Create origin**.
3. Under the **Behaviors** tab -> Click **Create behavior**:
   * **Path pattern:** `/api/*` (Redirects API routes starting with `/api/` to the backend ALB origin).
   * **Target origin:** Choose the ALB origin created above.
   * **Allowed HTTP methods:** Select `GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE`.
   * **Cache policy:** Select **CachingDisabled** (Ensures backend responses bypass edge caching).
   * **Origin request policy:** Select **AllViewerAndCloudFrontHeaders-2022-06**.
   * Click **Create behavior**.

---

### 4. Provisioning Gateway VPC Endpoint for S3

Because the backend EC2 instances reside inside Private subnets, uploading DB backups to S3 through a public NAT Gateway would incur costly data processing charges. We create an S3 VPC Gateway Endpoint to route traffic privately and free of charge over the AWS network backbone.

1. Navigate to **VPC** -> **Endpoints** -> Click **Create endpoint**.
2. **Service category:** Select **AWS services**.
3. **Services:** Search for `com.amazonaws.ap-southeast-1.s3` and select the **Gateway** type.
4. **VPC:** Select `shopsflow-vpc`.
5. **Route tables:** Check the box next to `shopsflow-private-rt` (Bảng định tuyến managing the Private App Subnets).
   * *Outcome:* AWS automatically updates the route table for private subnets so that S3-destined traffic routes directly via the gateway endpoint instead of escaping to the public internet.
6. Click **Create endpoint**.
