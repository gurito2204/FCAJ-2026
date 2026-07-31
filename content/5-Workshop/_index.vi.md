---
title: "Thực hành (Workshop)"
date: 2026-06-15
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Triển khai hệ thống thương mại điện tử sử dụng AWS Cloud

#### Tổng quan

Trong phần workshop này, chúng tôi sẽ thực hiện deploy hệ thống thương mại điện tử Business-to-Customer quy mô nhỏ **Shopsflow** (Java 21, Spring Boot 4.0.6, Spring Data JPA/Hibernate, PostgreSQL 16, Spring Security + JJWT, Swagger UI, JUnit5 + Mockito, Docker/Docker Compose, CI bằng GitHub Actions sử dụng các **Amazon Web Services (AWS)**.

Mục tiêu cốt lõi của bài thực hành này bao gồm:
* **Tính sẵn sàng cao (High Availability):** Thiết lập hạ tầng mạng VPC trên nhiều Availability Zones. Máy chủ ứng dụng EC2 được quản lý tự động bằng Auto Scaling Group (ASG) đặt sau Application Load Balancer (ALB). Cơ sở dữ liệu RDS PostgreSQL chạy ở chế độ Multi-AZ Standby.
* **Tính bảo mật:** Cô lập hoàn toàn máy chủ Backend EC2 và Database RDS trong các Private Subnets. Khách hàng chỉ truy cập Frontend thông qua Amazon CloudFront CDN kết hợp tường lửa AWS WAF. Quản lý thông tin mật và khóa mã hóa thông qua AWS Secrets Manager và KMS.
* **Tối ưu hóa kết nối & Giám sát:** EC2 kết nối tới S3 thông qua VPC Gateway Endpoint để sao lưu dữ liệu riêng tư mà không cần đi qua mạng Internet. Toàn bộ logs của container và metrics hệ điều hành được thu thập tập trung lên Amazon CloudWatch.

#### Nội dung bài lab

Quy trình thực hành được chia thành các bước chi tiết sau:

1. [Giới thiệu & Sơ đồ kiến trúc](5.1-workshop-overview/)
2. [Thiết lập Mạng (VPC Multi-AZ), Quyền (IAM) & Quản lý Secrets](5.2-prerequiste/)
3. [Tạo cơ sở dữ liệu Amazon RDS PostgreSQL Multi-AZ](5.3-rds/)
4. [Triển khai Frontend (S3+CloudFront) & Backend (ALB+ASG)](5.4-ec2/)
5. [Cấu hình Giám sát CloudWatch & Tự động sao lưu Database](5.5-monitoring-backup/)
6. [Dọn dẹp tài nguyên](5.6-cleanup/)