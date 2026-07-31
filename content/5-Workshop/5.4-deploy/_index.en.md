---
title: "Frontend & Backend API Deployment"
date: 2026-06-15
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

This article guides you through the separate deployment of the Frontend (S3 + CloudFront CDN + WAF) and the Backend running in containers on Amazon ECS Fargate (serverless container) located in a Private Subnet, behind an Application Load Balancer.

---

### 1. Deploy Web Frontend (S3 + CloudFront CDN + WAF)

#### Step 1: Create an S3 Bucket for the Frontend
1. Navigate to **S3** -> **Create bucket**.
2. **Bucket name:** `shopsflow-frontend-static-999`.
3. Keep the **Block all public access** option enabled.
4. Build the frontend (`npm install && npm run build`), and upload all contents of the `dist/` directory to the bucket.

#### Step 2: Create a CloudFront Distribution
1. Navigate to **CloudFront** -> **Create distribution**.
2. **Origin domain:** `shopsflow-frontend-static-999.s3.amazonaws.com`.
3. **Origin access:** Select **Origin access control settings (recommended)** -> **Create control setting** -> **Sign requests**.
4. **Default cache behavior:** Redirect HTTP to HTTPS.
5. **Default root object:** `index.html`.
6. **Custom error response:** 403/404 -> return `/index.html` (HTTP 200) to allow React Router to handle client-side routing (SPA fallback).
7. **WAF:** Select **Enable security protections** -> **Create Web ACL**.
8. Click **Create distribution**, then copy the **Bucket Policy** provided by CloudFront and paste it into the **Permissions** -> **Bucket policy** section of your S3 bucket.

![S3 bucket containing the frontend](/images/5-Workshop/5.4-deploy/s3_bucket.jpg)
![CloudFront distribution](/images/5-Workshop/5.4-deploy/fe_cloudfront.jpg)

---

### 2. Deploy Backend on ECS Fargate

**Step 1: Create an ECR Repository and push the image**
* ECR → Create repository → Private, name it `shopsflow-repo`.
* Build the Docker image from the Dockerfile in the backend repo.
* Authenticate Docker with ECR (`aws ecr get-login-password | docker login ...`), tag the image, and push:
  ```bash
  docker build -t shopsflow-repo .
  aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com
  docker tag shopsflow-repo:latest <account-id>[.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest](https://.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest)
  docker push <account-id>[.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest](https://.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest)
  ```
![ECR Image](/images/5-Workshop/5.4-deploy/ecr_image.jpg)

**Step 2: Create an Application Load Balancer**
* EC2 → Load Balancers → Create load balancer → Application Load Balancer.
* Configuration:
  * Load balancer name: `shopsflow-backend-alb`
  * Scheme: Internet-facing
  * VPC: `shopsflow-vpc`
  * Mappings: 2 public subnets (`shopsflow-public-1`, `shopsflow-public-2`)
  * Security groups: `alb-sg`
  * Listener HTTP:80 → Create target group:
    * Target type: IP (required for Fargate awsvpc — do not use "Instances")
    * Target group name: `shopsflow-backend-tg`
    * Protocol/Port: HTTP 8080
    * VPC: `shopsflow-vpc`
    * Health check path: `/api/products`
* Create load balancer, and record the DNS Name.

![ALB](/images/5-Workshop/5.4-deploy/alb.jpg)

**Step 3: Create an ECS Cluster and Task Definition**
* ECS → Clusters → Create cluster → select AWS Fargate (serverless) → name it `shopsflow-cluster`.
* Task definitions → Create new task definition:
  * Task definition family: `shopsflow-backend-task`
  * Launch type: AWS Fargate
  * Network mode: awsvpc (default with Fargate)
  * Task size: e.g., 0.5 vCPU / 1 GB memory (sufficient for the lab)
  * Task execution role: `shopsflow-ecs-task-execution-role` (created in 5.2)
  * Task role: `shopsflow-ecs-task-role` (created in 5.2)
  * Container definition:
    * Container name: `shopsflow-backend`
    * Image URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest`
    * Container port: 8080
    * Logging: enable awslogs, log group `/shopsflow/ecs/backend`, region `ap-southeast-1`
    * Environment variables:
      * `SPRING_DATASOURCE_URL` = `jdbc:postgresql://<rds-endpoint>:5432/shopsflow` (endpoint from 5.3)
      * `SPRING_DATASOURCE_USERNAME` = `postgres`
    * Secrets (do not use Environment variables for sensitive data): point to the `shopsflow/production/secrets` secret created in 5.2:
      * `SPRING_DATASOURCE_PASSWORD` ← `SPRING_DATASOURCE_PASSWORD` key in the secret
      * `JWT_SECRET` ← `JWT_SECRET` key in the secret
* Click Create.

**Step 4: Create an ECS Service**
* Go to `shopsflow-cluster` → Create service.
* Launch type: Fargate
* Task definition: `shopsflow-backend-task` (latest revision)
* Service name: `shopsflow-backend-service`
* Desired tasks: 2 (maintaining the concept of running in parallel across 2 AZs)
* Networking:
  * VPC: `shopsflow-vpc`
  * Subnets: `shopsflow-private-1`, `shopsflow-private-2`
  * Security group: `ecs-sg`
  * Public IP: Turned off
* Load balancing: select Application Load Balancer → target group `shopsflow-backend-tg`.
* (Optional) Service Auto Scaling: enable, Target tracking, metric `ECSServiceAverageCPUUtilization`, target 70, min 1 / max 4 — replacing the exact role of the Auto Scaling Group in the old EC2 version.
* Create service, and wait for the task to transition to RUNNING and the ALB target to become healthy.

![ECS Cluster Service](/images/5-Workshop/5.4-deploy/ecs_cluster_service.jpg)

### 3. Configure CloudFront Routing for the Backend API
* Return to the CloudFront distribution created in section 1.
* Tab Origins → Create origin:
  * Origin domain: DNS Name of `shopsflow-backend-alb`
  * Protocol: HTTP Only
* Tab Behaviors → Create behavior:
  * Path pattern: `/api/*`
  * Target origin: the ALB origin just created
  * Allowed HTTP methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
  * Cache policy: CachingDisabled
  * Origin request policy: AllViewerAndCloudFrontHeaders-2022-06
* Create behavior.
* Once the distribution is fully deployed, open the CloudFront domain to verify that both the frontend and `/api/*` are operating under the same domain (to avoid CORS issues).