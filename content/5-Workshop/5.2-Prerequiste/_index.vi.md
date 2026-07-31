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
* **AWS Region:** Chọn `us-east-1an`.
* **Công cụ cá nhân:** Cài sẵn AWS CLI, Git và SSH Client.

---

### 2. Thiết lập Mạng (Amazon VPC Multi-AZ)

Chúng ta cần xây dựng cấu trúc mạng chia tầng bảo mật bằng cách sử dụng 6 subnets phân bổ trên 2 Availability Zones.

#### Bước 1: Khởi tạo Mạng (VPC & Subnets)

1. Truy cập **AWS Console** -> **VPC** -> Chọn **Create VPC**.
2. Thiết lập:
   * **VPC settings:** Chọn **VPC and more** (tự động tạo Subnet và Routing).
   * **Name tag auto-generation:** `shopsflow-vpc`
   * **IPv4 CIDR block:** `10.0.0.0/16`
   * **Number of Availability Zones (AZs):** `2`
   * **Number of Public subnets:** `2`
   * **Number of Private subnets:** `2`
   * **NAT gateways:** Chọn **In 1 AZ** (hoặc **1 per AZ** nếu cần tính sẵn sàng cao).
3. Chọn **Create VPC**.

![Kết quả tạo VPC](vpc.jpg)

---

#### Bước 2: Cấu hình Security Groups

##### 2.1. Tạo Security Group cho ALB (alb-sg)
1. Truy cập **EC2 Console** -> **Security Groups** -> Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `alb-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound Rules**:
   * **Rule 1:** Type: **HTTP (80)** | Source: `0.0.0.0/0` (Anywhere)
   * **Rule 2:** Type: **HTTPS (443)** | Source: `0.0.0.0/0` (Anywhere)
4. Chọn **Create security group**.

![Kết quả tạo ALB Security Group](alb-sg.jpg)
![Cấu hình ALB](alb.jpg)

##### 2.2. Tạo Security Group cho ECS (ecs-sg)
1. Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `ecs-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound Rules**:
   * Type: **Custom TCP** | Port range: `8080` | Source: Chọn `alb-sg`
4. Chọn **Create security group**.

![Kết quả tạo ECS Security Group](ecs-sg.jpg)

##### 2.3. Tạo Security Group cho RDS (rds-sg)
1. Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `rds-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound Rules**:
   * Type: **PostgreSQL** | Port range: `5432` | Source: Chọn `ecs-sg`
4. Chọn **Create security group**.

![Kết quả tạo RDS Security Group](rds-sg.jpg)

#### Bước 3: Tạo Internet Gateway (IGW), NAT Gateway & Định tuyến mạng

##### 3.1. Khởi tạo và gắn kết Internet Gateway (IGW)
1. Truy cập **AWS Console** -> **VPC** -> **Internet gateways** -> Chọn **Create internet gateway**.
2. Thiết lập:
   * **Name tag:** `shopsflow-igw`
3. Chọn **Create internet gateway**.
4. Tại màn hình chi tiết của IGW vừa tạo, chọn **Actions** -> **Attach to VPC**.
5. Cấu hình gắn kết:
   * **VPC:** Chọn `shopsflow-vpc`
6. Chọn **Attach internet gateway**.

![Kết quả tạo và đính kèm Internet Gateway](igw.jpg)

---

##### 3.2. Khởi tạo 02 NAT Gateways (Đảm bảo High Availability)
* **Tạo NAT Gateway cho AZ 1:**
  1. Truy cập **VPC** -> **NAT gateways** -> Chọn **Create NAT gateway**.
  2. Thiết lập:
     * **Name:** `shopsflow-nat-gw-1`
     * **Subnet:** Chọn `shopsflow-public-1` (Public Subnet)
     * **Elastic IP allocation ID:** Chọn **Allocate Elastic IP**
  3. Chọn **Create NAT gateway**.

* **Tạo NAT Gateway cho AZ 2:**
  1. Chọn **Create NAT gateway**.
  2. Thiết lập:
     * **Name:** `shopsflow-nat-gw-2`
     * **Subnet:** Chọn `shopsflow-public-2` (Public Subnet)
     * **Elastic IP allocation ID:** Chọn **Allocate Elastic IP**
  3. Chọn **Create NAT gateway**.

*(Lưu ý: Quá trình khởi tạo NAT Gateway có thể mất từ 2–3 phút để chuyển sang trạng thái Available).*

![Kết quả tạo NAT Gateways](nat-gw.jpg)

---

##### 3.3. Cấu hình Bảng định tuyến (Route Tables)
* **Cấu hình Route cho Public Subnets (Trỏ ra IGW):**
  1. Truy cập **VPC** -> **Route tables** -> Chọn Route Table liên kết với các Public Subnets (`shopsflow-public-1`, `shopsflow-public-2`).
  2. Mở tab **Routes** -> Chọn **Edit routes** -> Chọn **Add route**.
  3. Thiết lập:
     * **Destination:** `0.0.0.0/0`
     * **Target:** Chọn **Internet Gateway** -> Trỏ vào `shopsflow-igw`
  4. Chọn **Save changes**.

* **Cấu hình Route cho Private Subnet 1 (Trỏ ra NAT 1):**
  1. Chọn Route Table của `shopsflow-private-1` -> Mở tab **Routes** -> Chọn **Edit routes** -> **Add route**.
  2. Thiết lập:
     * **Destination:** `0.0.0.0/0`
     * **Target:** Chọn **NAT Gateway** -> Trỏ vào `shopsflow-nat-gw-1`
  3. Chọn **Save changes**.

* **Cấu hình Route cho Private Subnet 2 (Trỏ ra NAT 2):**
  1. Chọn Route Table của `shopsflow-private-2` -> Mở tab **Routes** -> Chọn **Edit routes** -> **Add route**.
  2. Thiết lập:
     * **Destination:** `0.0.0.0/0`
     * **Target:** Chọn **NAT Gateway** -> Trỏ vào `shopsflow-nat-gw-2`
  3. Chọn **Save changes**.


#### Bước 4: Thiết lập Bảng định tuyến (Route Tables)

##### 4.1. Public Route Table (Dành cho các Public Subnets)
1. Truy cập **AWS Console** -> **VPC** -> **Route tables** -> Chọn **Create route table**.
2. Thiết lập:
   * **Name:** `shopsflow-public-rt`
   * **VPC:** Chọn `shopsflow-vpc`
3. Chọn **Create route table**.
4. Cấu hình định tuyến:
   * Mở tab **Routes** -> Chọn **Edit routes** -> Chọn **Add route**.
   * **Destination:** `0.0.0.0/0` | **Target:** Chọn **Internet Gateway** (`shopsflow-igw`)
   * Chọn **Save changes**.
5. Gán Subnet:
   * Mở tab **Subnet associations** -> Chọn **Edit subnet associations**.
   * Tick chọn `shopsflow-public-1` và `shopsflow-public-2`.
   * Chọn **Save associations**.

![Kết quả cấu hình Public Route Table](public-rt.jpg)

---

##### 4.2. Private Route Table 1 (Dành cho Private Subnet tại AZ 1)
1. Chọn **Create route table**.
2. Thiết lập:
   * **Name:** `shopsflow-private-rt-1`
   * **VPC:** Chọn `shopsflow-vpc`
3. Chọn **Create route table**.
4. Cấu hình định tuyến:
   * Mở tab **Routes** -> Chọn **Edit routes** -> Chọn **Add route**.
   * **Destination:** `0.0.0.0/0` | **Target:** Chọn **NAT Gateway** (`shopsflow-nat-gw-1`)
   * Chọn **Save changes**.
5. Gán Subnet:
   * Mở tab **Subnet associations** -> Chọn **Edit subnet associations**.
   * Tick chọn `shopsflow-private-1` *(chứa ECS Backend và RDS tại AZ 1)*.
   * Chọn **Save associations**.

![Kết quả cấu hình Private Route Table 1](private-rt-1.jpg)

---

##### 4.3. Private Route Table 2 (Dành cho Private Subnet tại AZ 2)
1. Chọn **Create route table**.
2. Thiết lập:
   * **Name:** `shopsflow-private-rt-2`
   * **VPC:** Chọn `shopsflow-vpc`
3. Chọn **Create route table**.
4. Cấu hình định tuyến:
   * Mở tab **Routes** -> Chọn **Edit routes** -> Chọn **Add route**.
   * **Destination:** `0.0.0.0/0` | **Target:** Chọn **NAT Gateway** (`shopsflow-nat-gw-2`)
   * Chọn **Save changes**.
5. Gán Subnet:
   * Mở tab **Subnet associations** -> Chọn **Edit subnet associations**.
   * Tick chọn `shopsflow-private-2` *(chứa ECS Backend và RDS tại AZ 2)*.
   * Chọn **Save associations**.

![Kết quả cấu hình Private Route Table 2](private-rt-2.jpg)
---

### 4. Cấu hình Bảo mật (KMS & Secrets Manager)

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

### 5. Thiết lập Firewalls (Security Groups)

Chúng ta tạo 3 Security Groups để kiểm soát truy cập phân tầng:

1. **ALB Security Group (`shopsflow-alb-sg`):**
   * Inbound: Cho phép `HTTP` (Port 80) và `HTTPS` (Port 443) từ mọi nơi (`0.0.0.0/0`).
2. **EC2 Security Group (`shopsflow-ec2-sg`):**
   * Inbound 1: Cho phép `TCP` (Port 8080) với Source là Security Group `shopsflow-alb-sg` (Chỉ nhận request đi qua Load Balancer).
   * Inbound 2: Cho phép `SSH` (Port 22) từ IP tĩnh của bạn (hoặc Session Manager).
3. **RDS Security Group (`shopsflow-rds-sg`):**
   * Inbound: Cho phép `PostgreSQL` (Port 5432) với Source là Security Group `shopsflow-ec2-sg`.

---

### 6. Tạo IAM Role cho EC2

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