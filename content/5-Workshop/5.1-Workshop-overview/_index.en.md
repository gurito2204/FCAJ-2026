---
title: "Introduction & Architecture"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---
 
### 1. Project Idea & Objectives
 
#### Background & Problem Statement
The **Shopsflow** system is a complete full-stack e-commerce application including a Customer interface (Storefront) for searching, shopping for products, and paying online via the VNPay gateway, combined with an Admin Portal to manage product catalogs, track orders, manage inventory, and view revenue analytics.
 
Target customers are small and medium-sized businesses (SMBs), and traditional retail store owners who have a need for digital transformation to the online environment with optimal costs, complete autonomy over source code and databases without relying on third-party SaaS platforms.
 
The system solves the following problems:
* **Reduce downtime and deployment risks:** Overcome environmental conflicts (library version errors between local machines and servers) by using containerization technology (Docker), packaging the application into an image and running it on **Amazon ECS (Fargate)** instead of installing it directly on the server.
* **Data security:** Prevent customer data leaks by placing backend container tasks and the database in an isolated virtual private network zone (Private Subnets).
* **Data preservation:** Automate the periodic PostgreSQL database backup process, avoiding information loss in case of incidents.
* **Centralized monitoring capability:** Centralize all application logs and system metrics to CloudWatch for easy troubleshooting.

#### Specific Objectives
* **Desired Outputs:**
  * **Frontend Web:** React + Vite Single Page Application (SPA) statically deployed on Amazon S3 and distributed via Amazon CloudFront CDN.
  * **Backend API:** Spring Boot RESTful API packaged with Docker, image stored on **Amazon ECR (Elastic Container Registry)** and run as a task on **Amazon ECS with Fargate launch type** (serverless container, no need to self-manage EC2), placed in a Private Subnet, behind an Application Load Balancer (ALB).
  * **Database RDS:** PostgreSQL Database on Amazon RDS, placed in a Private Subnet, completely disabling public access.
  * **Security & Encryption:** Use AWS Secrets Manager and KMS to store passwords and sensitive configurations.
  * **Monitoring System:** Centralized monitoring dashboard and logs on CloudWatch for both ECS tasks and RDS.

#### Program Suitability
The project utilizes basic and advanced AWS platform services including: **VPC**, **ECS (Fargate)**, **ECR**, **RDS**, **CloudFront**, **S3**, **Application Load Balancer**, **Secrets Manager**, **KMS**, **CloudWatch**, and **IAM**. The infrastructure structure adheres to AWS's secure and high-availability design principles (Well-Architected Framework), making it highly suitable as a practical hands-on topic for students in the First Cloud Journey (FCJ) program.
 
---
 
### 2. Architecture Diagram & Technical Design
 
#### Architecture Diagram
 
Below is the architecture diagram describing the tiered structure and data flow of the Shopsflow application when deployed on AWS infrastructure:
 
![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.png)
 
#### Usecases
| No. | Use case | API Used | Function |
|---|---|---|---|
| 1 | Register account | POST `/api/auth/register` | Create a new account, return JWT for immediate login |
| 2 | Login | POST `/api/auth/login` | Authenticate email/password, return JWT used for subsequent requests |
| 3 | Search & filter products | GET `/api/products` | View product list, supports `keyword`, `categoryId`, `minPrice`, `maxPrice`, pagination & sorting |
| 4 | View product details | GET `/api/products/{id}` | Get specific information of 1 product |
| 5 | Create product (Admin) | POST `/api/products` | Admin adds a new product to the catalog |
| 6 | Update product (Admin) | PUT `/api/products/{id}` | Admin edits product information |
| 7 | Delete product (Admin) | DELETE `/api/products/{id}` | Admin removes a product from the catalog |
| 8 | View product reviews | GET `/api/products/{productId}/reviews` | View review list of 1 product (public) |
| 9 | Write a review | POST `/api/products/{productId}/reviews` | User posts a review (1-5 stars, max 2000 characters comment), each user can only review once per product |
| 10 | Edit review | PUT `/api/reviews/{reviewId}` | Review owner edits their own review |
| 11 | Delete review | DELETE `/api/reviews/{reviewId}` | Review owner deletes their own review |
| 12 | View categories | GET `/api/categories` | View all product categories (public) |
| 13 | View category details | GET `/api/categories/{id}` | View a specific category |
| 14 | Create category (Admin) | POST `/api/categories` | Admin adds a new category |
| 15 | Update category (Admin) | PUT `/api/categories/{id}` | Admin edits a category |
| 16 | Delete category (Admin) | DELETE `/api/categories/{id}` | Admin deletes a category |
| 17 | View cart | GET `/api/cart` | User views their current cart |
| 18 | Add to cart | POST `/api/cart/items` | User adds a product to the cart |
| 19 | Update quantity in cart | PUT `/api/cart/items/{itemId}` | User adjusts product quantity in the cart |
| 20 | Remove product from cart | DELETE `/api/cart/items/{itemId}` | User removes 1 item from the cart |
| 21 | Place order (checkout) | POST `/api/orders` | Convert cart into an order, initial status `PENDING` |
| 22 | View my orders | GET `/api/orders` | User views their own order list |
| 23 | View order details | GET `/api/orders/{id}` | User views details of 1 of their orders |
| 24 | Update order status (Admin) | PUT `/api/orders/{id}/status` | Admin updates order status (e.g., delivered, canceled...) |
| 25 | View all orders (Admin) | GET `/api/orders/all` | Admin views all orders in the system |
| 26 | Initiate VNPay payment | POST `/api/payment/vnpay/checkout/{orderId}` | Create VNPay Sandbox payment link (`payUrl`) for the order |
| 27 | Receive payment result (redirect) | GET `VNPAY_RETURN_URL` (`/payment-result`, frontend side) | VNPay redirects the browser back with transaction result query params |
| 28 | Confirm payment (IPN webhook) | `VNPAY_IPN_URL` (server-to-server) | VNPay implicitly calls back to the backend to accurately update the order status, asynchronously |
| 29 | Reconcile order status after payment | GET `/api/orders/{orderId}` | Frontend calls back to fetch the official status (`PAID`/`PENDING`/`CANCELLED`) from DB |


**Runtime flow (numbered according to the diagram):**
- 1–2: User → CloudFront → S3 (serve frontend)
- 3–6: User → Internet Gateway → ALB → ECS Cluster → Fargate (call API)
- 7–8: BackEnd Container ↔ VNPay: create Payment URL, receive Payment Status callback

| Service | Application |
|---|---|
| CloudFront + S3 | Static hosting + CDN for frontend |
| Internet Gateway + NAT Gateway | Internet connection for public/private subnet |
| VPC (public/private subnet, 2 AZ) | Network isolation, multi-tier |
| ALB | Load balancing into backend |
| ECS Cluster + Fargate | Container orchestration, running BackEnd container |
| RDS (2 AZ) | Managed database |
| ECR | Private registry containing Docker images |
| CloudWatch | Monitoring/logging |
| IAM | Access control |
| GitHub + Docker | Build & push image to ECR |
 
#### Service Selection Rationale
 
* **Amazon CloudFront & Amazon S3 (Frontend):**
  * *Reason for selection:* Hosting static web assets (HTML/JS/CSS/Images) on S3 and distributing them via CloudFront completely offloads the backend, accelerates global page load times through Edge Location caching, and optimizes costs.
* **Amazon ECS (Fargate) & Amazon ECR (Backend):**
  * *Reason for selection:* Packaging the Spring Boot backend into a Docker image, storing it on ECR, and running it as a task on ECS Fargate — a serverless container model that eliminates the need to self-manage, patch, or manually scale the underlying EC2 instances. ALB distributes traffic to ECS tasks located in Private Subnets, and the ECS Service can automatically adjust the number of concurrently running tasks based on system load.
* **Amazon RDS PostgreSQL:**
  * *Reason for selection:* A managed database service reduces the operational burden (patching, backup, failover) compared to self-installing PostgreSQL on a server, while placing it entirely in a Private Subnet to prevent public internet exposure.
* **AWS Secrets Manager & KMS:**
  * *Reason for selection:* Centralized storage and management of sensitive information (Database Password, JWT Secret) encrypted using AWS KMS. ECS Tasks will automatically fetch temporary credentials at runtime instead of storing raw configuration files in the image.
* **Application Load Balancer:**
  * *Reason for selection:* Distributes internet traffic from the Internet Gateway to ECS tasks in Private Subnets, ensuring complete isolation between the public tier and the internal backend tier.

#### Security & Access Control
* **IAM Task Role / Execution Role:** Assign IAM Roles to ECS Tasks, allowing them to read Secrets from Secrets Manager, pull images from ECR, and send logs/metrics to CloudWatch — following the principle of least privilege (not using EC2 instance permissions because the backend does not run directly on EC2).
* **Network Isolation:** ECS Tasks and RDS are located entirely in Private Subnets, with no public IP addresses. Security Groups are set up in a chained manner: `Internet` → `CloudFront` → `Internet Gateway` → `ALB SG` → `ECS Task SG` → `RDS SG`.