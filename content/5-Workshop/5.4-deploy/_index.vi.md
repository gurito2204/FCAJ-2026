---
title: "Triển khai Frontend & Backend API"
date: 2026-06-15
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Bài viết này hướng dẫn triển khai tách biệt phần Frontend (S3 + CloudFront CDN + WAF) và phần Backend chạy container trên Amazon ECS Fargate (serverless container) đặt trong Private Subnet, phía sau Application Load Balancer.

---

### 1. Triển khai Web Frontend (S3 + CloudFront CDN + WAF)


#### Bước 1: Tạo S3 Bucket cho Frontend
1. Truy cập **S3** -> **Create bucket**.
2. **Bucket name:** `shopsflow-frontend-static-999`.
3. Giữ tùy chọn **Block all public access**.
4. Build frontend (`npm install && npm run build`), upload toàn bộ nội dung trong thư mục `dist/` lên bucket.

#### Bước 2: Tạo CloudFront Distribution
1. Truy cập **CloudFront** -> **Create distribution**.
2. **Origin domain:** `shopsflow-frontend-static-999.s3.amazonaws.com`.
3. **Origin access:** Chọn **Origin access control settings (recommended)** -> **Create control setting** -> **Sign requests**.
4. **Default cache behavior:** Redirect HTTP to HTTPS.
5. **Default root object:** `index.html`.
6. **Custom error response:** 403/404 -> trả về `/index.html` (HTTP 200) để React Router xử lý client-side routing (SPA fallback).
7. **WAF:** Chọn **Enable security protections** -> **Create Web ACL**.
8. Chọn **Create distribution**, copy **Bucket Policy** do CloudFront cung cấp và dán vào phần **Permissions** -> **Bucket policy** của S3 bucket.

![S3 bucket containing the frontend](/images/5-Workshop/5.4-deploy/s3_bucket.jpg)
![CloudFront distribution](/images/5-Workshop/5.4-deploy/fe_cloudfront.jpg)

---

### 2. Triển khai Backend trên ECS Fargate

**Bước 1: Tạo ECR Repository và push image**
* ECR → Create repository → Private, tên `shopsflow-repo`.
* Build Docker image từ Dockerfile trong repo backend.
* Authenticate Docker với ECR (`aws ecr get-login-password | docker login ...`), tag image, push:
  ```bash
  docker build -t shopsflow-repo .
  aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com
  docker tag shopsflow-repo:latest <account-id>[.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest](https://.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest)
  docker push <account-id>[.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest](https://.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest)
  ```
![ECR Image](/images/5-Workshop/5.4-deploy/ecr_image.jpg)

**Bước 2: Tạo Application Load Balancer**
* EC2 → Load Balancers → Create load balancer → Application Load Balancer.
* Thiết lập:
  * Load balancer name: `shopsflow-backend-alb`
  * Scheme: Internet-facing
  * VPC: `shopsflow-vpc`
  * Mappings: 2 public subnet (`shopsflow-public-1`, `shopsflow-public-2`)
  * Security groups: `alb-sg`
  * Listener HTTP:80 → Create target group:
    * Target type: IP (bắt buộc cho Fargate awsvpc — không dùng "Instances")
    * Target group name: `shopsflow-backend-tg`
    * Protocol/Port: HTTP 8080
    * VPC: `shopsflow-vpc`
    * Health check path: `/api/products`
* Create load balancer, ghi lại DNS Name.

![ALB](/images/5-Workshop/5.4-deploy/alb.jpg)

**Bước 3: Tạo ECS Cluster và Task Definition**
* ECS → Clusters → Create cluster → chọn AWS Fargate (serverless) → tên `shopsflow-cluster`.
* Task definitions → Create new task definition:
  * Task definition family: `shopsflow-backend-task`
  * Launch type: AWS Fargate
  * Network mode: awsvpc (mặc định với Fargate)
  * Task size: ví dụ 0.5 vCPU / 1 GB memory (đủ cho lab)
  * Task execution role: `shopsflow-ecs-task-execution-role` (đã tạo ở 5.2)
  * Task role: `shopsflow-ecs-task-role` (đã tạo ở 5.2)
  * Container definition:
    * Container name: `shopsflow-backend`
    * Image URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/shopsflow-repo:latest`
    * Container port: 8080
    * Logging: bật awslogs, log group `/shopsflow/ecs/backend`, region `ap-southeast-1`
    * Environment variables:
      * `SPRING_DATASOURCE_URL` = `jdbc:postgresql://<rds-endpoint>:5432/shopsflow` (endpoint từ 5.3)
      * `SPRING_DATASOURCE_USERNAME` = `postgres`
    * Secrets (không dùng Environment variables cho dữ liệu nhạy cảm): trỏ tới secret `shopsflow/production/secrets` đã tạo ở 5.2:
      * `SPRING_DATASOURCE_PASSWORD` ← key `SPRING_DATASOURCE_PASSWORD` trong secret
      * `JWT_SECRET` ← key `JWT_SECRET` trong secret
* Chọn Create.



**Bước 4: Tạo ECS Service**
* Vào `shopsflow-cluster` → Create service.
* Launch type: Fargate
* Task definition: `shopsflow-backend-task` (revision mới nhất)
* Service name: `shopsflow-backend-service`
* Desired tasks: 2 (giữ tinh thần chạy song song 2 AZ)
* Networking:
  * VPC: `shopsflow-vpc`
  * Subnets: `shopsflow-private-1`, `shopsflow-private-2`
  * Security group: `ecs-sg`
  * Public IP: Turned off
* Load balancing: chọn Application Load Balancer → target group `shopsflow-backend-tg`.
* (Tùy chọn) Service Auto Scaling: bật, Target tracking, metric `ECSServiceAverageCPUUtilization`, target 70, min 1 / max 4 — thay thế đúng vai trò của Auto Scaling Group trong bản EC2 cũ.
* Create service, đợi task chuyển RUNNING và target ALB chuyển healthy.

![ECS Cluster Service](/images/5-Workshop/5.4-deploy/ecs_cluster_service.jpg)

### 3. Cấu hình định tuyến CloudFront cho API Backend
* Quay lại CloudFront distribution đã tạo ở phần 1.
* Tab Origins → Create origin:
  * Origin domain: DNS Name của `shopsflow-backend-alb`
  * Protocol: HTTP Only
* Tab Behaviors → Create behavior:
  * Path pattern: `/api/*`
  * Target origin: ALB origin vừa tạo
  * Allowed HTTP methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
  * Cache policy: CachingDisabled
  * Origin request policy: AllViewerAndCloudFrontHeaders-2022-06
* Create behavior.
* Sau khi distribution deploy xong, mở domain CloudFront để kiểm tra cả frontend lẫn `/api/*` hoạt động dưới cùng 1 tên miền (tránh lỗi CORS).