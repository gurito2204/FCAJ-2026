---
title: "Khởi tạo Database (Amazon RDS - PostgreSQL)"
date: 2026-06-15
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

Chúng ta sẽ khởi tạo một cơ sở dữ liệu quan hệ Amazon RDS PostgreSQL chạy ở chế độ Multi-AZ. Database nằm hoàn toàn trong phân vùng mạng riêng (Private Subnets) đã tạo ở mục 5.2 và chỉ nhận kết nối nội bộ từ ECS Fargate task.

Để đảm bảo cơ sở dữ liệu hoàn toàn cách ly khỏi Internet và chỉ nhận kết nối từ container ECS, quy trình tạo RDS sẽ được chia thành hai phần: Tạo Subnet Group (để khóa DB vào Private Subnet) và Tạo Database Instance với cấu hình Multi-AZ để đảm bảo High Availability.

---

### 1. Tạo DB Subnet Group cho RDS

Trước khi khởi tạo database RDS, AWS yêu cầu định nghĩa một **DB Subnet Group** chứa các subnets ở ít nhất 2 Availability Zones khác nhau để phục vụ cơ chế HA Multi-AZ. Chúng ta dùng lại đúng 2 private subnet đã tạo ở 5.2 — không tạo thêm subnet mới.

1. Truy cập **AWS Console** -> Tìm kiếm và chọn dịch vụ **RDS**.
2. Ở menu điều hướng trái, chọn **Subnet groups** -> Chọn **Create DB subnet group**.
3. Cấu hình thông tin cơ bản:
   * **Name:** `shopsflow-db-subnet-group`
   * **Description:** DB Subnets Group for Shopsflow Multi-AZ PostgreSQL
   * **VPC:** `shopsflow-vpc`
4. Tại phần **Add subnets**:
   * **Availability Zones:** Chọn `ap-southeast-1a` và `ap-southeast-1b`
   * **Subnets:** Tick chọn `shopsflow-private-1` (AZ1) và `shopsflow-private-2` (AZ2) — đúng 2 private subnet đã tạo ở 5.2, không phải subnet riêng cho DB.
5. Chọn **Create**.

---

### 2. Tạo Cơ sở dữ liệu RDS PostgreSQL Multi-AZ

1. Tại RDS Console, chọn **Databases** -> **Create database**.
2. Cấu hình các thông số sau:
   * **Database creation method:** Standard create
   * **Engine options:** PostgreSQL, phiên bản mặc định khuyên dùng (ví dụ PostgreSQL 16.x)
   * **Templates:** Production hoặc Dev/Test *(không chọn Free Tier vì Free Tier không hỗ trợ Multi-AZ)*
   * **Availability and durability:** Chọn **Multi-AZ DB instance** — AWS tự tạo Primary DB ở AZ1 và Standby DB đồng bộ ở AZ2 để tự động failover.
3. Tại phần **Settings**:
   * **DB instance identifier:** `shopsflow-db`
   * **Master username:** `postgres`
   * **Master password:** Lấy đúng giá trị key `SPRING_DATASOURCE_PASSWORD` đã lưu trong secret `shopsflow/production/secrets` ở mục 5.2 — không gõ tay mật khẩu mới, để đảm bảo backend đọc secret ra sẽ khớp với mật khẩu thật của DB.
4. Tại phần **Instance configuration:** 
   * Chọn cấu hình tiết kiệm chi phí, ví dụ `db.t3.micro` hoặc `db.t4g.micro`.
5. Tại phần **Storage**:
   * **Allocated storage:** Tối thiểu 20 GiB
   * Bỏ tick **Enable storage autoscaling** để tránh phát sinh chi phí ngoài dự kiến.
6. Tại phần **Connectivity**:
   * **VPC:** `shopsflow-vpc`
   * **DB subnet group:** Chọn `shopsflow-db-subnet-group` (vừa tạo ở mục 1)
   * **Public access:** No
   * **Existing VPC security groups:** Chọn `rds-sg` đã tạo ở 5.2 (bỏ tick nhóm default)
7. Tại phần **Database authentication:** Chọn Password authentication.
8. Tại phần **Additional configuration:** 
   * Mở rộng menu -> **Initial database name:** `shopsflow`
9. Chọn **Create database**.

---

### 3. Ghi nhận Endpoint kết nối

Quá trình khởi tạo Multi-AZ mất khoảng 10–15 phút do AWS thiết lập đồng thời 2 instance ở 2 AZ và cấu hình replication.

1. Khi trạng thái chuyển thành **Available**, click vào DB instance `shopsflow-db`.
2. Chuyển sang Tab **Connectivity & security** -> sao chép **Endpoint** (dạng `shopsflow-db.xxxx.ap-southeast-1.rds.amazonaws.com`) và **Port** `5432`.
3. Endpoint này sẽ được truyền cho ECS Task Definition ở mục 5.4 thông qua biến môi trường / secret — không hard-code trong source code.

> **Lưu ý:** Database RDS PostgreSQL hiện đã được cô lập hoàn toàn. Do tắt Public access và inbound rule của `rds-sg` chỉ mở port 5432 cho các thực thể thuộc `ecs-sg`, database chỉ nhận kết nối truy vấn SQL phát ra từ ECS Fargate task đang chạy trong private subnet. Mọi truy cập từ bên ngoài VPC — kể cả từ EC2 khác không thuộc `ecs-sg` — đều bị từ chối.

![Màn hình chi tiết Database Endpoint và Multi-AZ Configuration](rds-endpoint.jpg)

---

### 4. Cấu hình Source Code Spring Boot kết nối RDS

Để mã nguồn hoạt động linh hoạt trên môi trường container hóa (ECS) và đảm bảo an toàn thông tin, toàn bộ thông số kết nối cơ sở dữ liệu sẽ được truyền thông qua biến môi trường (Environment Variables).

#### 4.1. Bổ sung Dependency PostgreSQL
Đảm bảo file `pom.xml` (Maven) đã khai báo driver cho PostgreSQL:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>