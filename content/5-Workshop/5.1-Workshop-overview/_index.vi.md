---
title: "Giới thiệu & Kiến trúc"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

### 1. Ý tưởng & Mục tiêu dự án

#### Bối cảnh & Bài toán
Hệ thống **Shopsflow** là một ứng dụng thương mại điện tử full-stack hoàn chỉnh bao gồm giao diện Khách hàng (Storefront) để tìm kiếm, mua sắm sản phẩm và thanh toán trực tuyến qua cổng VNPay, kết hợp với giao diện Quản trị viên (Admin Portal) nhằm quản lý danh mục sản phẩm, theo dõi đơn hàng, quản lý kho và xem phân tích doanh thu.

Khách hàng mục tiêu là các doanh nghiệp vừa và nhỏ (SMBs), các chủ cửa hàng bán lẻ truyền thống đang có nhu cầu chuyển đổi số lên môi trường trực tuyến với chi phí tối ưu, tự chủ hoàn toàn về mã nguồn và cơ sở dữ liệu mà không bị phụ thuộc vào các nền tảng SaaS bên thứ ba.

Hệ thống giải quyết các vấn đề sau:
* **Giảm downtime và rủi ro triển khai:** Khắc phục tình trạng xung đột môi trường (lỗi phiên bản thư viện giữa máy local và máy chủ) bằng công nghệ container hóa (Docker).
* **Bảo mật dữ liệu:** Ngăn ngừa việc rò rỉ dữ liệu khách hàng bằng cách đưa cơ sở dữ liệu và máy chủ ứng dụng vào vùng mạng riêng cô lập (Private Subnets).
* **Bảo toàn dữ liệu:** Tự động hóa quy trình sao lưu (backup) cơ sở dữ liệu PostgreSQL định kỳ qua kết nối mạng nội bộ, tránh mất mát thông tin khi máy chủ gặp sự cố phần cứng.
* **Khả năng giám sát tập trung:** Tập trung hóa toàn bộ log ứng dụng và các thông số phần cứng lên Cloud để dễ dàng xử lý sự cố.

#### Mục tiêu cụ thể
* **Đầu ra mong muốn (Outputs):**
  * **Frontend Web:** React + Vite Single Page Application (SPA) được deploy tĩnh trên Amazon S3 và phân phối qua Amazon CloudFront CDN.
  * **Backend API:** Spring Boot RESTful API chạy trên EC2 đặt trong Private Subnet, quản lý tự động bằng Auto Scaling Group (ASG) phía sau Application Load Balancer (ALB).
  * **Database RDS:** PostgreSQL Database chạy chế độ Multi-AZ Standby, tắt hoàn toàn khả năng truy cập công cộng.
  * **Security & Encryption:** Sử dụng AWS Secrets Manager và KMS để lưu mật khẩu và cấu hình nhạy cảm. Sử dụng AWS WAF bảo vệ CloudFront khỏi tấn công web.
  * **Monitoring & Backup System:** Dashboard giám sát và logs tập trung trên CloudWatch. Backup RDS PostgreSQL nén đẩy lên S3 bảo mật qua VPC Gateway Endpoint.

#### Phù hợp chương trình
Dự án sử dụng các dịch vụ nền tảng cơ bản và nâng cao của AWS bao gồm: **VPC**, **EC2**, **RDS**, **CloudFront**, **WAF**, **S3**, **Secrets Manager**, **KMS**, **CloudWatch**, và **IAM**. Cấu trúc hạ tầng tuân thủ nguyên tắc thiết kế bảo mật và sẵn sàng cao của AWS (Well-Architected Framework), rất phù hợp làm đề tài thực hành thực tế cho học viên trong chương trình First Cloud Journey (FCJ).

---

### 2. Sơ đồ kiến trúc & Thiết kế kỹ thuật

#### Sơ đồ kiến trúc (Architecture Diagram)

Dưới đây là sơ đồ kiến trúc mô tả cấu trúc phân tầng và luồng dữ liệu của ứng dụng Shopsflow khi triển khai trên hạ tầng AWS:

![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.jpg)

#### Lựa chọn dịch vụ (Service Selection Rationale)

* **Amazon CloudFront & Amazon S3 (Frontend):**
  * *Lý do chọn:* Đưa web tĩnh (HTML/JS/CSS/Ảnh) lên S3 và phân phối qua CloudFront giúp giảm tải hoàn toàn cho máy chủ EC2, tăng tốc độ tải trang toàn cầu nhờ bộ nhớ đệm (Caching) tại các Edge Location, và tối ưu chi phí. Bảo vệ bằng AWS WAF ngăn chặn tấn công DDoS và SQL Injection ở tầng biên.
* **AWS ALB & Auto Scaling Group (EC2 Backend):**
  * *Lý do chọn:* ALB phân phối lưu lượng truy cập tới các EC2 instances trong Private Subnet của 2 Availability Zones (AZ), đảm bảo tính sẵn sàng cao. Auto Scaling Group tự động thêm/bớt EC2 dựa trên mức độ sử dụng CPU, ngăn ngừa sập hệ thống khi lượng tải tăng đột biến.
* **Amazon RDS PostgreSQL Multi-AZ:**
  * *Lý do chọn:* Chế độ Multi-AZ giúp tự động nhân bản dữ liệu sang một vùng dự phòng độc lập. Khi Availability Zone chính gặp sự cố, RDS sẽ tự động chuyển hướng kết nối sang Standby database mà không làm gián đoạn ứng dụng.
* **AWS Secrets Manager & KMS:**
  * *Lý do chọn:* Lưu trữ và quản lý tập trung các thông tin nhạy cảm (Database Password, JWT Secret) được mã hóa bằng AWS KMS. EC2 sẽ tự động gọi API lấy credentials tạm thời lúc runtime thay vì lưu file cấu hình thô.
* **VPC Gateway Endpoint (S3):**
  * *Lý do chọn:* Cho phép các EC2 instances trong Private Subnet kết nối trực tiếp đến S3 để đẩy tệp backup cơ sở dữ liệu qua mạng riêng của AWS, không cần đi qua NAT Gateway giúp tiết kiệm đáng kể chi phí băng thông NAT.
* **NAT Gateway:**
  * *Lý do chọn:* Cho phép các tài nguyên trong Private Subnets gửi gói tin ra ngoài internet (outbound) để cập nhật phần mềm hoặc giao tiếp với cổng thanh toán VNPay nhưng chặn mọi kết nối không mong muốn từ ngoài vào (inbound).

#### Bảo mật & IAM (Security & Access Control)
* **IAM Instance Profile:** Gán IAM Role cho EC2 cho phép đọc Secrets từ Secrets Manager và gửi dữ liệu logs/metrics lên CloudWatch.
* **Mạng Cô Lập (Network Isolation):** EC2 và RDS nằm hoàn toàn trong Private Subnets. Không có địa chỉ IP công cộng cho các máy chủ này. Security Groups được thiết lập theo dạng chuỗi: `Internet` -> `CloudFront` -> `ALB SG` -> `EC2 SG` -> `RDS SG`.
