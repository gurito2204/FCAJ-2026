---
title: "Thiết lập Mạng, Quyền & Secrets"
date: 2026-06-15
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Bài viết này hướng dẫn chuẩn bị môi trường mạng riêng ảo **Amazon VPC** theo chuẩn có sẵn (VPC Multi-AZ), cấu hình khóa mã hóa **AWS KMS**, lưu trữ thông tin nhạy cảm trên **AWS Secrets Manager**, cấu hình **Security Groups** và phân quyền **IAM**.

---

### 1. Chuẩn bị (Prerequisites)
* **Tài khoản AWS:** Có quyền quản trị.
* **AWS Region:** Chọn Singapore (`ap-southeast-1`).
* **Công cụ cá nhân:** Cài sẵn AWS CLI, Git và SSH Client.

---

### 2. Thiết lập Mạng (Amazon VPC Multi-AZ)

Chúng ta cần xây dựng cấu trúc mạng chia tầng bảo mật bằng cách sử dụng 6 subnets phân bổ trên 2 Availability Zones (AZs) ap-southeast-1a và ap-southeast-1b.

#### Bước 1: Khởi tạo VPC
1. Truy cập **AWS Console** -> **VPC** -> Chọn **Your VPCs** -> **Create VPC**.
2. Thiết lập:
   * **VPC settings:** Chọn **VPC only**.
   * **Name tag:** `shopsflow-vpc`
   * **IPv4 CIDR block:** `10.0.0.0/16`
3. Chọn **Create VPC**.

#### Bước 2: Tạo các Subnets
Truy cập **VPC** -> **Subnets** -> **Create subnet**. Chọn VPC `shopsflow-vpc`. Tạo lần lượt 6 subnets sau:

1. **Public Subnet 1 (AZ1):** CIDR `10.0.1.0/24`, AZ: `ap-southeast-1a`, Name: `shopsflow-public-1`
2. **Public Subnet 2 (AZ2):** CIDR `10.0.2.0/24`, AZ: `ap-southeast-1b`, Name: `shopsflow-public-2`
3. **Private App Subnet 1 (AZ1):** CIDR `10.0.3.0/24`, AZ: `ap-southeast-1a`, Name: `shopsflow-private-app-1`
4. **Private App Subnet 2 (AZ2):** CIDR `10.0.4.0/24`, AZ: `ap-southeast-1b`, Name: `shopsflow-private-app-2`
5. **Private DB Subnet 1 (AZ1):** CIDR `10.0.5.0/24`, AZ: `ap-southeast-1a`, Name: `shopsflow-private-db-1`
6. **Private DB Subnet 2 (AZ2):** CIDR `10.0.6.0/24`, AZ: `ap-southeast-1b`, Name: `shopsflow-private-db-2`

#### Bước 3: Tạo Internet Gateway (IGW) & NAT Gateway
1. **Tạo Internet Gateway:**
   * Truy cập **VPC** -> **Internet gateways** -> **Create internet gateway**.
   * Name: `shopsflow-igw`. Click **Create**.
   * Chọn Action -> **Attach to VPC** -> Chọn `shopsflow-vpc`.
2. **Tạo NAT Gateway:**
   * Truy cập **VPC** -> **NAT gateways** -> **Create NAT gateway**.
   * Name: `shopsflow-nat-gw`.
   * **Subnet:** Chọn `shopsflow-public-1` (Bắt buộc phải nằm ở Public Subnet).
   * **Elastic IP allocation ID:** Chọn **Allocate Elastic IP** để cấp IP tĩnh công cộng cho NAT.
   * Click **Create NAT gateway** và đợi trạng thái chuyển sang *Available*.

#### Bước 4: Thiết lập Bảng định tuyến (Route Tables)
1. **Public Route Table (Cho Public Subnet):**
   * Truy cập **Route tables** -> **Create route table**. Name: `shopsflow-public-rt`, VPC: `shopsflow-vpc`.
   * Tại tab **Routes** -> Chọn **Edit routes** -> Thêm route: Destination `0.0.0.0/0`, Target: **Internet Gateway** (`shopsflow-igw`).
   * Tại tab **Subnet associations** -> Chọn **Edit subnet associations** -> Tick chọn `shopsflow-public-1` và `shopsflow-public-2`.
2. **Private Route Table (Cho EC2 Backend Subnet):**
   * Tạo route table: Name: `shopsflow-private-rt`.
   * Tại tab **Routes** -> Thêm route: Destination `0.0.0.0/0`, Target: **NAT Gateway** (`shopsflow-nat-gw`).
   * Liên kết với Subnets: Tick chọn `shopsflow-private-app-1` và `shopsflow-private-app-2`.
3. **DB Route Table (Cô lập hoàn toàn):**
   * Tạo route table: Name: `shopsflow-db-rt` (Không cấu hình route ra ngoài).
   * Liên kết với Subnets: Tick chọn `shopsflow-private-db-1` và `shopsflow-private-db-2`.

---

### 3. Cấu hình Bảo mật (KMS & Secrets Manager)

Để bảo vệ các chuỗi kết nối và mật khẩu nhạy cảm, chúng ta sử dụng AWS Secrets Manager được mã hóa bởi AWS KMS.

#### Bước 1: Tạo AWS KMS Key
1. Truy cập **AWS Console** -> **Key Management Service (KMS)** -> **Customer managed keys** -> **Create key**.
2. Chọn **Symmetric**, Key usage: **Encrypt and decrypt**.
3. Alias: `shopsflow-kms-key`. Chọn **Create**.

#### Bước 2: Khởi tạo Secret trên Secrets Manager
1. Truy cập **Secrets Manager** -> Chọn **Store a new secret**.
2. **Secret type:** Chọn **Other type of secret**.
3. Điền các cặp key/value lưu thông tin database và JWT:
   * Key: `SPRING_DATASOURCE_PASSWORD` / Value: `ShopsflowPass123!`
   * Key: `JWT_SECRET` / Value: `ThayTheBangChuoiSecretCucKyDaiVaMat1234567890!`
4. **Encryption key:** Chọn đúng KMS key `shopsflow-kms-key` vừa tạo.
5. Đặt tên Secret: `shopsflow/production/secrets`. Chọn **Store**.

---

### 4. Thiết lập Firewalls (Security Groups)

Chúng ta tạo 3 Security Groups để kiểm soát truy cập phân tầng:

1. **ALB Security Group (`shopsflow-alb-sg`):**
   * Inbound: Cho phép `HTTP` (Port 80) và `HTTPS` (Port 443) từ mọi nơi (`0.0.0.0/0`).
2. **EC2 Security Group (`shopsflow-ec2-sg`):**
   * Inbound 1: Cho phép `TCP` (Port 8080) với Source là Security Group `shopsflow-alb-sg` (Chỉ nhận request đi qua Load Balancer).
   * Inbound 2: Cho phép `SSH` (Port 22) từ IP tĩnh của bạn (hoặc Session Manager).
3. **RDS Security Group (`shopsflow-rds-sg`):**
   * Inbound: Cho phép `PostgreSQL` (Port 5432) với Source là Security Group `shopsflow-ec2-sg`.

---

### 5. Tạo IAM Role cho EC2

1. Truy cập **IAM** -> **Roles** -> **Create role** -> Common use case: **EC2**.
2. Gán các Policies sau:
   * `CloudWatchAgentServerPolicy`: Để gửi log và metrics.
   * `AmazonS3FullAccess`: Thao tác lưu trữ backup.
3. Tạo thêm một Inline Policy cho phép đọc Secret:
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
4. Đặt tên role: `ShopsflowEC2Role` và chọn **Create role**.