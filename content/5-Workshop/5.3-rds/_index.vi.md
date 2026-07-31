---
title: "Tạo cơ sở dữ liệu Amazon RDS PostgreSQL"
date: 2026-06-15
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

Chúng ta sẽ khởi tạo một cơ sở dữ liệu quan hệ **Amazon RDS PostgreSQL** chạy ở chế độ **Multi-AZ** (Nhiều vùng sẵn sàng). Database sẽ được cấu hình nằm hoàn toàn trong phân vùng mạng cô lập riêng biệt (**Private DB Subnets**) và chỉ cho phép nhận kết nối nội bộ từ nhóm máy chủ ứng dụng EC2.

---

### 1. Tạo DB Subnet Group cho RDS

Trước khi khởi tạo database RDS, AWS yêu cầu định nghĩa một **DB Subnet Group** chứa các subnets ở ít nhất 2 Availability Zones khác nhau để phục vụ cơ chế HA Multi-AZ.

1. Truy cập **AWS Console** -> Tìm kiếm và chọn dịch vụ **RDS**.
2. Ở menu điều hướng trái, chọn **Subnet groups** -> Chọn **Create DB subnet group**.
3. Cấu hình thông tin cơ bản:
   * **Name:** `shopsflow-db-subnet-group`
   * **Description:** `DB Subnets Group for Shopsflow Multi-AZ PostgreSQL`
   * **VPC:** Chọn đúng VPC `shopsflow-vpc`.
4. **Add subnets:**
   * **Availability Zones:** Chọn `ap-southeast-1a` và `ap-southeast-1b`.
   * **Subnets:** Tick chọn đúng 2 dải mạng Private DB Subnets đã tạo ở phần trước (`10.0.5.0/24` và `10.0.6.0/24` tương ứng với `shopsflow-private-db-1` và `shopsflow-private-db-2`).
5. Chọn **Create**.

---

### 2. Tạo Cơ sở dữ liệu RDS PostgreSQL Multi-AZ

1. Tại màn hình điều hướng RDS, chọn **Databases** -> Chọn **Create database**.
2. Cấu hình các thông số:
   * **Database creation method:** Chọn **Standard create** (Để có thể kích hoạt tùy chọn Multi-AZ).
   * **Engine options:** Chọn **PostgreSQL**.
   * **Engine version:** Chọn phiên bản mặc định khuyên dùng (ví dụ: *PostgreSQL 16.x*).
3. **Templates:** Chọn **Production** hoặc **Dev/Test** (Không chọn Free Tier vì Free Tier không hỗ trợ Multi-AZ).
4. **Availability and durability (Tính sẵn sàng cao):**
   * Chọn **Multi-AZ DB instance** (Tùy chọn này sẽ tự động tạo một database chính Primary DB ở AZ1 và một database dự phòng Standby DB đồng bộ ở AZ2 để tự động failover khi có sự cố).
5. **Settings:**
   * **DB instance identifier:** `shopsflow-db`
   * **Master username:** `postgres`
   * **Master password:** Nhập mật khẩu quản trị của bạn (Ví dụ: `ShopsflowPass123!`). *Lưu ý: Mật khẩu này phải trùng với giá trị `SPRING_DATASOURCE_PASSWORD` bạn đã khai báo trong Secrets Manager ở chương trước.*
6. **Instance configuration:**
   * Chọn dòng instance tiết kiệm tài nguyên (ví dụ: Burstable classes -> `db.t3.micro` hoặc `db.t4g.micro` để hạn chế tối đa chi phí lab).
7. **Storage:**
   * Chọn Allocated storage tối thiểu: `20 GiB`. Tắt tính năng *Enable storage autoscaling* để tránh phát sinh chi phí tự động tăng dung lượng.
8. **Connectivity:**
   * **Virtual private cloud (VPC):** Chọn `shopsflow-vpc`.
   * **DB subnet group:** Chọn đúng group `shopsflow-db-subnet-group` vừa tạo ở mục 1.
   * **Public access:** Chọn **No** (Ngăn chặn database gán IP công cộng, bảo mật tuyệt đối khỏi Internet).
   * **Existing VPC security groups:** Chọn `shopsflow-rds-sg` đã tạo ở Bước 3. (Bỏ tick nhóm default).
9. **Database authentication:** Chọn **Password authentication**.
10. **Additional configuration (Cấu hình nâng cao):**
    * Nhấp mở rộng menu -> Nhập **Initial database name:** `shopsflow` (Để RDS tự động khởi tạo database trống tên là `shopsflow` lúc startup).
11. Chọn **Create database**.

---

### 3. Ghi nhận Endpoint kết nối

1. Quá trình khởi tạo cơ sở dữ liệu Multi-AZ sẽ diễn ra từ **10 đến 15 phút** do AWS phải đồng thời thiết lập 2 instances ở 2 vùng sẵn sàng vật lý khác nhau và cấu hình replication đồng bộ.
2. Khi trạng thái chuyển sang màu xanh **Available**, click chọn vào DB instance `shopsflow-db`.
3. Tại tab **Connectivity & security**, sao chép lại địa chỉ **Endpoint** (Ví dụ: `shopsflow-db.xxxx.ap-southeast-1.rds.amazonaws.com`).
4. Endpoint này chính là host kết nối database. Chúng ta sẽ lưu Endpoint này vào Secrets Manager hoặc file cấu hình backend.

> [!IMPORTANT]
> Database RDS PostgreSQL hiện tại đã được cô lập hoàn toàn. Do tắt `Public access` và cấu hình inbound của `shopsflow-rds-sg` chỉ mở port 5432 cho các thực thể thuộc `shopsflow-ec2-sg`, database chỉ có thể nhận kết nối truy vấn SQL phát ra từ các máy chủ EC2 đặt trong Private App Subnets. Mọi truy cập từ bên ngoài VPC sẽ bị từ chối hoàn toàn.
