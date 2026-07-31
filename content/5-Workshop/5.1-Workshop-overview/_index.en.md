---
title: "Introduction & Architecture"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---
 
### 1. Project Idea & Objectives
 
#### Background & Problem Statement
The **Shopsflow** system is a complete full-stack e-commerce application featuring a Storefront for product search, shopping, and online payment via the VNPay gateway, alongside an Admin Portal for product catalog management, order tracking, inventory management, and revenue analytics.
 
The target audience consists of small and medium-sized businesses (SMBs) and traditional retail store owners looking to undergo digital transformation with optimized costs and complete autonomy over source code and databases, without depending on third-party SaaS platforms.
 
The system solves the following challenges:
* **Reducing downtime and deployment risks:** Mitigating environment conflicts (library version discrepancies between local machines and servers) using containerization (Docker), packaging the application into an image, and running it on **Amazon ECS (Fargate)** instead of direct server installations.
* **Data security:** Preventing customer data leaks by placing backend containers and databases in isolated private networks (Private Subnets).
* **Data protection:** Automating periodic PostgreSQL database backups to prevent information loss during incidents.
* **Centralized monitoring:** Consolidating all application logs and system metrics into CloudWatch for streamlined troubleshooting.

#### Specific Objectives
* **Expected Outputs:**
  * **Frontend Web:** React + Vite Single Page Application (SPA) statically deployed on Amazon S3 and distributed via Amazon CloudFront CDN.
  * **Backend API:** Spring Boot RESTful API packaged in Docker, storing images on **Amazon ECR (Elastic Container Registry)**, and running as tasks on **Amazon ECS with the Fargate launch type** (serverless containers, eliminating the need to manage EC2 instances manually) located in a Private Subnet behind an Application Load Balancer (ALB).
  * **Database RDS:** PostgreSQL database on Amazon RDS, located in a Private Subnet with public access completely disabled.
  * **Security & Encryption:** Using AWS Secrets Manager and KMS to store passwords and sensitive configurations.
  * **Monitoring System:** A centralized monitoring dashboard and logging on CloudWatch for both ECS tasks and RDS.

#### Program Alignment
The project utilizes foundational and advanced AWS services including **VPC**, **ECS (Fargate)**, **ECR**, **RDS**, **CloudFront**, **S3**, **Application Load Balancer**, **Secrets Manager**, **KMS**, **CloudWatch**, and **IAM**. The infrastructure design adheres to the AWS Well-Architected Framework, making it ideal as a practical capstone project for participants in the First Cloud Journey (FCJ) program.
 
---
 
### 2. Architecture Diagram & Technical Design
 
#### Architecture Diagram
 
The architecture diagram below illustrates the layered structure and data flow of the Shopsflow application when deployed on AWS infrastructure:
 
![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.png)
 
#### Usecase
| No. | Use Case | API Used | Description |
|---|---|---|---|
| 1 | Register account | POST `/api/auth/register` | Create a new account, returns a JWT for immediate login |
| 2 | Login | POST `/api/auth/login` | Authenticate email/password, returns a JWT for subsequent requests |
| 3 | Search & filter products | GET `/api/products` | View product list, supports `keyword`, `categoryId`, `minPrice`, `maxPrice`, pagination & sorting |
| 4 | View product details | GET `/api/products/{id}` | Retrieve information for a specific product |
| 5 | Create product (Admin) | POST `/api/products` | Admin adds a new product to the catalog |
| 6 | Update product (Admin) | PUT `/api/products/{id}` | Admin updates product information |
| 7 | Delete product (Admin) | DELETE `/api/products/{id}` | Admin removes a product from the catalog |
| 8 | View product reviews | GET `/api/products/{productId}/reviews` | View review list for a product (public) |
| 9 | Write a review | POST `/api/products/{productId}/reviews` | User posts a review (1–5 stars, comment up to 2000 characters), limited to 1 review per user/product |
| 10 | Edit review | PUT `/api/reviews/{reviewId}` | Review author edits their own review |
| 11 | Delete review | DELETE `/api/reviews/{reviewId}` | Review author deletes their own review |
| 12 | View categories | GET `/api/categories` | View all product categories (public) |
| 13 | View category details | GET `/api/categories/{id}` | View a specific category |
| 14 | Create category (Admin) | POST `/api/categories` | Admin adds a new category |
| 15 | Update category (Admin) | PUT `/api/categories/{id}` | Admin updates a category |
| 16 | Delete category (Admin) | DELETE `/api/categories/{id}` | Admin deletes a category |
| 17 | View cart | GET `/api/cart` | User views their current shopping cart |
| 18 | Add to cart | POST `/api/cart/items` | User adds a product to the cart |
| 19 | Update cart quantity | PUT `/api/cart/items/{itemId}` | User updates product quantity in the cart |
| 20 | Remove item from cart | DELETE `/api/cart/items/{itemId}` | User removes an item from the cart |
| 21 | Checkout | POST `/api/orders` | Converts the cart into an order with an initial `PENDING` status |
| 22 | View my orders | GET `/api/orders` | User views their list of orders |
| 23 | View order details | GET `/api/orders/{id}` | User views details of a specific order |
| 24 | Update order status (Admin) | PUT `/api/orders/{id}/status` | Admin updates order status (e.g., delivered, cancelled...) |
| 25 | View all orders (Admin) | GET `/api/orders/all` | Admin views all orders in the system |
| 26 | Initialize VNPay payment | POST `/api/payment/vnpay/checkout/{orderId}` | Generates a VNPay Sandbox payment link (`payUrl`) for the order |
| 27 | Receive payment result (redirect) | GET `VNPAY_RETURN_URL` (`/payment-result` on frontend) | VNPay redirects browser with transaction result query parameters |
| 28 | Confirm payment (IPN webhook) | `VNPAY_IPN_URL` (server-to-server) | VNPay calls the backend asynchronously to update the exact order status |
| 29 | Reconcile order status post-payment | GET `/api/orders/{orderId}` | Frontend calls back to fetch the official status (`PAID`/`PENDING`/`CANCELLED`) from DB |

**Runtime Flow (referenced by numbers in the diagram):**
- 1–2: User → CloudFront → S3 (serve frontend)
- 3–6: User → Internet Gateway → ALB → ECS Cluster → Fargate (API calls)
- 7–8: Backend Container ↔ VNPay: generate Payment URL, receive Payment Status callback

| Service | Application |
|---|---|
| CloudFront + S3 | Static hosting + CDN for frontend |
| Internet Gateway + NAT Gateway | Internet connectivity for public/private subnets |
| VPC (public/private subnets, 2 AZs) | Network isolation, multi-tier |
| ALB | Load balancing into the backend |
| ECS Cluster + Fargate | Container orchestration, runs backend containers |
| RDS (2 AZs) | Managed database |
| ECR | Private registry storing Docker images |
| CloudWatch | Monitoring/logging |
| IAM | Access control |
| GitHub + Docker | Build & push images to ECR |
 
#### Service Selection Rationale
 
* **Amazon CloudFront & Amazon S3 (Frontend):**
  * *Reason for selection:* Hosting static assets (HTML/JS/CSS/Images) on S3 and distributing them via CloudFront offloads traffic completely from the backend, accelerates global page loading speed via caching at Edge Locations, and optimizes costs.
* **Amazon ECS (Fargate) & Amazon ECR (Backend):**
  * *Reason for selection:* Packaging the Spring Boot backend into a Docker image, storing it on ECR, and running it as tasks on ECS Fargate—a serverless container model that eliminates the need to manually manage, patch, or scale underlying EC2 instances. The ALB distributes traffic to ECS tasks placed in Private Subnets, and the ECS service can automatically scale the number of parallel running tasks based on system load.
* **Amazon RDS PostgreSQL:**
  * *Reason for selection:* A managed database service that reduces operational overhead (patching, backups, failover) compared to self-hosting PostgreSQL on a server, while residing entirely within a Private Subnet without exposing it to the Internet.
* **AWS Secrets Manager & KMS:**
  * *Reason for selection:* Centrally stores and manages sensitive information (Database Password, JWT Secret) encrypted using AWS KMS. ECS tasks automatically fetch temporary credentials at runtime instead of embedding raw configuration files inside the image.
* **Application Load Balancer:**
  * *Reason for selection:* Distributes traffic from the Internet Gateway to ECS tasks in Private Subnets, ensuring complete separation between the public tier and the internal backend tier.

#### Security & Access Control (IAM & Network Isolation)
* **IAM Task Role / Execution Role:** Assigns IAM roles to ECS tasks to read secrets from Secrets Manager, pull images from ECR, and send logs/metrics to CloudWatch—following the principle of least privilege (avoiding EC2 instance roles since the backend does not run directly on EC2).
* **Network Isolation:** ECS tasks and RDS reside entirely within Private Subnets without public IP addresses. Security Groups are structured in a chained manner: `Internet` → `CloudFront` → `Internet Gateway` → `ALB SG` → `ECS Task SG` → `RDS SG`.