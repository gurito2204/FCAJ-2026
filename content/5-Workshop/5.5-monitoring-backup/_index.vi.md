---
title: "Giám sát, Sao lưu & Kiểm thử"
date: 2026-06-15
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

Bài viết này hướng dẫn cấu hình giám sát tập trung qua **Amazon CloudWatch**, thiết lập cơ chế sao lưu cơ sở dữ liệu riêng tư qua **VPC S3 Gateway Endpoint**, và các kịch bản kiểm thử (Test Cases) thực tế kiểm chứng khả năng tự động co giãn (Auto Scaling) và phân tải hệ thống.

---

### 1. Giám sát hệ thống (CloudWatch Agent & Alarm)

Vì các máy chủ EC2 nằm trong nhóm Auto Scaling Group, chúng sẽ tự động cài đặt và chạy CloudWatch Agent thông qua script User Data đã thiết lập ở Launch Template.

#### Cấu hình CloudWatch Alarm cho Auto Scaling Group
1. Truy cập **AWS Console** -> **CloudWatch** -> Chọn **Alarms** -> **Create alarm**.
2. Chọn **Select metric** -> Chọn **EC2** -> Chọn **By Auto Scaling Group** -> Chọn chỉ số `CPUUtilization` của group `shopsflow-backend-asg`.
3. Cấu hình điều kiện cảnh báo:
   * **Threshold type:** Static
   * **Condition:** Greater than `80%` (CPU vượt quá 80% trong 2 chu kỳ đánh giá liên tiếp, mỗi chu kỳ 5 phút).
4. **Configure actions (Cấu hình thông báo):**
   * **Alarm state trigger:** Chọn **In alarm**.
   * **Send a notification to:** Tạo mới hoặc chọn một **SNS Topic** có liên kết với Email của bạn (để AWS tự động gửi email cảnh báo khi hệ thống quá tải).
5. Đặt tên Alarm: `shopsflow-asg-high-cpu-alarm` và chọn **Create alarm**.

---

### 2. Thực hiện sao lưu dữ liệu riêng tư qua VPC Endpoint

Với VPC S3 Gateway Endpoint đã thiết lập ở chương trước, traffic upload tệp dump của cơ sở dữ liệu từ EC2 lên S3 sẽ đi hoàn toàn trong mạng nội bộ AWS, giúp bảo mật tuyệt đối dữ liệu nhạy cảm và không phát sinh phí NAT Gateway.

1. Sử dụng AWS System Manager Session Manager hoặc SSH vào một trong các instance EC2 đang chạy trong Private Subnet.
2. Di chuyển đến thư mục chứa script backup của dự án:
   ```bash
   cd ~/shopsflow/deploy/aws
   ```
3. Cấu hình tên S3 bucket backup của bạn trong tệp script `backup_to_s3.sh` (Ví dụ: `S3_BUCKET="shopsflow-db-backup-999"`).
4. Thực thi script backup:
   ```bash
   ./backup_to_s3.sh
   ```
5. **Xác thực kết nối riêng tư (VPC Endpoint Check):**
   * Chạy lệnh kiểm tra tuyến đường mạng (traceroute) hoặc thực hiện lệnh list đối tượng:
     ```bash
     aws s3 ls s3://shopsflow-db-backup-999 --region ap-southeast-1
     ```
   * *Xác thực:* Tệp nén backup database `.sql.gz` xuất hiện trong S3 bucket mà không cần máy chủ EC2 có IP công cộng và không làm tăng thông số data-transfer-out của NAT Gateway.

---

### 3. Các kịch bản kiểm thử hệ thống (Validation Scenarios)

#### 🧪 Kịch bản 1: Kiểm thử Định tuyến Edge-to-Backend
* **Cách thực hiện:** Truy cập vào tên miền **CloudFront Distribution Domain Name** trên trình duyệt (Ví dụ: `https://dxxxxx.cloudfront.net`).
* **Kết quả mong đợi:** 
  * Giao diện trang chủ React Frontend tải thành công từ S3 qua CloudFront CDN.
  * Khi click xem danh mục sản phẩm, CloudFront định tuyến chính xác các request `/api/*` tới Application Load Balancer (ALB). ALB phân tải tròn (Round Robin) đến 2 EC2 instances nằm ở Private Subnet để lấy dữ liệu sản phẩm từ RDS PostgreSQL và trả về hiển thị lên màn hình.

#### 🧪 Kịch bản 2: Kiểm tra Log Streams tập trung trên CloudWatch
* **Cách thực hiện:** Truy cập **CloudWatch** -> **Log groups** -> Chọn nhóm `/shopsflow/ec2/docker`.
* **Kết quả mong đợi:** Xuất hiện đầy đủ các Log Streams tương ứng với các EC2 instances trong Auto Scaling Group (Log được gom tự động từ các container Docker). Khi một máy chủ mới được scale up, log stream của nó sẽ tự động được tạo và đẩy lên đây.

#### 🧪 Kịch bản 3: Kiểm thử Trừ kho đồng thời (Concurrency Checkout)
* **Cách thực hiện:** Sử dụng Apache Benchmark (`ab`) gửi đồng thời nhiều requests mua một sản phẩm duy nhất còn lại:
  ```bash
  ab -n 20 -c 10 -p payload.json -T application/json https://dxxxxx.cloudfront.net/api/checkout
  ```
* **Kết quả mong đợi:** 
  * Chỉ duy nhất 1 request xử lý thành công. Tất cả các request đồng thời khác bị từ chối và trả về mã lỗi `409 Conflict`.
  * Log của Spring Boot ghi nhận ngoại lệ `OptimisticLockingFailureException`. Số lượng sản phẩm trong kho database RDS không bị âm.

#### 🧪 Kịch bản 4: Giả lập lỗi quá tải để Test Auto Scaling & Alarm
* **Cách thực hiện:** SSH vào một máy chủ EC2 đang chạy trong Private Subnet và kích hoạt quá tải CPU bằng công cụ `stress`:
  ```bash
  sudo apt-get install -y stress
  stress --cpu 4 --timeout 300s
  ```
* **Kết quả mong đợi:**
  1. Đồ thị **CPU Utilization** của Auto Scaling Group tăng mạnh.
  2. Trạng thái CloudWatch Alarm `shopsflow-asg-high-cpu-alarm` chuyển từ `OK` sang `ALARM`, hệ thống gửi email cảnh báo về hòm thư của bạn.
  3. Cơ chế **Auto Scaling** phát hiện quá tải -> Kích hoạt scale-out -> Tự động khởi tạo máy chủ EC2 thứ 3 từ Launch Template.
  4. Máy chủ thứ 3 tự động cấu hình ứng dụng, và tự động được đăng ký (Register) vào target group `shopsflow-backend-tg` của Application Load Balancer để chia sẻ tải. Khi hết thời gian stress, hệ thống tự động scale-in giảm về 2 instances.
