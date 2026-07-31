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
#### Bước 1: Khởi tạo Mạng (VPC & Subnets)

##### 1.1. Khởi tạo VPC
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

![Kết quả tạo VPC](/images/5-Workshop/5.2-Prerequisite/vpc.jpg)

---

##### 1.2. Tạo 4 Subnets (Phân bổ ở 2 Availability Zones)
1. Truy cập **VPC** -> **Subnets** -> Chọn **Create subnet**.
2. **VPC ID:** Chọn `shopsflow-vpc`.
3. Cấu hình thông tin từng Subnet:
   * **Subnet 1 (Public AZ 1):** Name: `shopsflow-public-1` | AZ: `ap-southeast-1a` | CIDR: `10.0.1.0/24`
   * **Subnet 2 (Public AZ 2):** Name: `shopsflow-public-2` | AZ: `ap-southeast-1b` | CIDR: `10.0.2.0/24`
   * **Subnet 3 (Private AZ 1):** Name: `shopsflow-private-1` | AZ: `ap-southeast-1a` | CIDR: `10.0.3.0/24`
   * **Subnet 4 (Private AZ 2):** Name: `shopsflow-private-2` | AZ: `ap-southeast-1b` | CIDR: `10.0.4.0/24`
4. Chọn **Create subnets**.
5. Sau khi tạo xong, tick chọn 2 Public Subnets (`shopsflow-public-1`, `shopsflow-public-2`) -> Chọn **Actions** -> **Edit subnet settings** -> Tick chọn **Enable auto-assign public IPv4 address** để các tài nguyên khởi tạo trong này tự động nhận IP Public.
![Tạo Public Subnet 1](/images/5-Workshop/5.2-Prerequisite/public_subnet1.jpg)
![Tạo Public Subnet 2](/images/5-Workshop/5.2-Prerequisite/public_subnet2.jpg)
![Tạo Private Subnet 1](/images/5-Workshop/5.2-Prerequisite/private_subnet1.jpg)
![Tạo Private Subnet 2](/images/5-Workshop/5.2-Prerequisite/private_subnet2.jpg)


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

![Kết quả tạo ALB Security Group](/images/5-Workshop/5.2-Prerequisite/alb-sg.jpg)
![Cấu hình ALB](/images/5-Workshop/5.2-Prerequisite/alb.jpg)

##### 2.2. Tạo Security Group cho ECS (ecs-sg)
1. Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `ecs-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound Rules**:
   * Type: **Custom TCP** | Port range: `8080` | Source: Chọn `alb-sg`
4. Chọn **Create security group**.

![Kết quả tạo ECS Security Group](/images/5-Workshop/5.2-Prerequisite/ecs-sg.jpg)

##### 2.3. Tạo Security Group cho RDS (rds-sg)
1. Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `rds-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound Rules**:
   * Type: **PostgreSQL** | Port range: `5432` | Source: Chọn `ecs-sg`
4. Chọn **Create security group**.

![Kết quả tạo RDS Security Group](/images/5-Workshop/5.2-Prerequisite/rds-sg.jpg)

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

### 5. Thiết lập Tường lửa ảo (Security Groups)

Truy cập **AWS Console** -> **EC2** -> **Security Groups** và lần lượt tạo 3 Security Groups để kiểm soát truy cập phân tầng theo đúng kiến trúc.

#### 5.1. Tạo ALB Security Group (alb-sg)
1. Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `alb-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound rules**:
   * **Rule 1:** Type: **HTTP** | Port: `80` | Source: `0.0.0.0/0`
   * **Rule 2:** Type: **HTTPS** | Port: `443` | Source: `0.0.0.0/0`
4. Chọn **Create security group**.

![Kết quả tạo ALB Security Group](/images/5-Workshop/5.2-Prerequisite/alb-sg.jpg)

---

#### 5.2. Tạo ECS Security Group (ecs-sg)
1. Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `ecs-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound rules**:
   * Type: **Custom TCP** | Port range: `8080` | Source: Chọn Security Group `alb-sg`
4. Chọn **Create security group**.

*(Lưu ý: Cấu hình này đảm bảo backend chỉ nhận request đi qua Load Balancer).*

![Kết quả tạo ECS Security Group](/images/5-Workshop/5.2-Prerequisite/ecs-sg.jpg)

---

#### 5.3. Tạo RDS Security Group (rds-sg)
1. Chọn **Create security group**.
2. Thiết lập thông tin chung:
   * **Security group name:** `rds-sg`
   * **VPC:** Chọn `shopsflow-vpc`
3. Cấu hình **Inbound rules**:
   * Type: **PostgreSQL** | Port range: `5432` | Source: Chọn Security Group `ecs-sg`
4. Chọn **Create security group**.

![Kết quả tạo RDS Security Group](/images/5-Workshop/5.2-Prerequisite/rds-sg.jpg)

---

#### 5.4. Kiểm tra danh sách Security Groups
Quay lại màn hình danh sách Security Groups chính, kiểm tra 3 nhóm bảo mật `alb-sg`, `ecs-sg` và `rds-sg` đã được khởi tạo thành công và liên kết chính xác với VPC.

![Danh sách tổng hợp Security Groups](/images/5-Workshop/5.2-Prerequisite/sg.jpg)
---

### 6. Thiết lập IAM Roles cho ECS

Để đảm bảo phân quyền bảo mật theo nguyên tắc quyền tối thiểu (Principle of Least Privilege) cho hệ thống container, ta cần khởi tạo 2 IAM Role riêng biệt: một dành cho môi trường thực thi nền tảng ECS Fargate (Task Execution Role) và một dành riêng cho mã nguồn ứng dụng (Task Role).

#### Bước 1: Tạo ECS Task Execution Role (shopsflow-ecs-task-execution-role)
Role này cấp quyền cho môi trường nền tảng ECS (Fargate agent) thực hiện các tác vụ cơ sở hạ tầng như kéo (pull) Docker image từ ECR và đẩy log hệ thống (container logs) lên CloudWatch.

1. Truy cập **AWS Console** -> **IAM Console** -> Chọn **Roles** ở menu bên trái -> Chọn **Create role**.
2. Thiết lập Trusted Entity:
   * **Trusted entity type:** Chọn **AWS service**.
   * **Use case:** Tìm mục **Use cases for other AWS services** -> Chọn **Elastic Container Service** -> Chọn **Elastic Container Service Task** *(Allows ECS tasks to call AWS services on your behalf)*.
   * Chọn **Next**.
3. Cấp quyền Policy:
   * Tại trang **Add permissions**, tìm kiếm từ khóa: `AmazonECSTaskExecutionRolePolicy`.
   * Tick chọn policy `AmazonECSTaskExecutionRolePolicy` (policy do AWS quản lý sẵn).
   * Chọn **Next**.
4. Hoàn tất khởi tạo:
   * **Role name:** `shopsflow-ecs-task-execution-role`
   * **Description:** `Role cho phép Fargate pull image từ ECR và đẩy log lên CloudWatch`
   * Chọn **Create role**.



---

#### Bước 2: Tạo ECS Task Role (shopsflow-ecs-task-role)
Role này cấp quyền cho chính mã nguồn ứng dụng (Application Code) đang chạy bên trong container. Bất kỳ lệnh gọi AWS SDK nào từ code Spring Boot (như ghi file lên S3, đọc biến môi trường từ Secrets Manager, gửi tin nhắn SQS) đều sẽ sử dụng quyền từ role này.

1. Truy cập lại **IAM Console** -> Chọn **Roles** -> Chọn **Create role**.
2. Thiết lập Trusted Entity:
   * **Trusted entity type:** Chọn **AWS service**.
   * **Use case:** Chọn **Elastic Container Service** -> Chọn **Elastic Container Service Task**.
   * Chọn **Next**.
3. Cấp quyền Policy (tùy thuộc vào nhu cầu thực tế của backend Shopsflow):
   * *Ví dụ 1:* Nếu app cần tương tác với S3 bucket để lưu trữ file tĩnh, tìm và tick chọn policy liên quan đến S3 (ví dụ: `AmazonS3FullAccess` hoặc policy tùy chỉnh).
   * *Ví dụ 2:* Nếu app cần lấy thông tin cấu hình từ Parameter Store hoặc Secrets Manager, tick chọn policy cấp quyền `ssm:GetParameters` hoặc `secretsmanager:GetSecretValue`.
   * *(Lưu ý: Nếu ứng dụng hiện tại chỉ tương tác với RDS bằng user/password thông thường và không gọi bất kỳ AWS API nào khác, có thể không tick chọn policy nào và thêm vào sau khi cần).*
   * Chọn **Next**.
4. Hoàn tất khởi tạo:
   * **Role name:** `shopsflow-ecs-task-role`
   * **Description:** `Role cấp quyền cho Application Code bên trong container`
   * Chọn **Create role**.
