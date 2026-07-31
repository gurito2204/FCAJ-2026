---
title: "Đề xuất"
date: 2026-07-30
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## Shopsflow

### Đề xuất triển khai ứng dụng thương mại điện tử container trên AWS

#### 1. Tóm tắt điều hành

Shopsflow là ứng dụng thương mại điện tử full-stack gồm React/Vite frontend, Spring Boot backend, PostgreSQL database và payment flow trực tuyến. Đề xuất này tách frontend, application runtime và database để mỗi tầng có trách nhiệm, security boundary và deployment path rõ ràng.

Frontend được phân phối từ Amazon S3 qua Amazon CloudFront. Backend chạy container trên Amazon ECS Fargate phía sau Application Load Balancer (ALB), trong khi Amazon RDS for PostgreSQL lưu dữ liệu ở private tier. Amazon ECR lưu backend image theo version; Amazon CloudWatch hỗ trợ log và kiểm tra vận hành.

#### 2. Vấn đề và giải pháp đề xuất

Việc chạy toàn bộ ứng dụng trên một public server sẽ gắn chặt static delivery, API runtime và database access. Điều này cũng làm lộ nhiều hạ tầng hơn cần thiết, đồng thời khiến deployment, scaling, troubleshoot và cost review khó hơn.

Giải pháp đề xuất sử dụng kiến trúc AWS ba tầng:

- **Frontend delivery:** CloudFront phục vụ React/Vite build từ S3 bucket private.
- **Application tier:** Internet-facing ALB route API request đến Spring Boot container trên ECS Fargate trong private subnet.
- **Data tier:** RDS PostgreSQL ở private tier và chỉ nhận port `5432` từ ECS security group.

Cách tiếp cận này giới hạn public entry point ở CloudFront và ALB, còn backend task cùng database không có public IP.

#### 3. Kiến trúc giải pháp

![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.png)
_Hình 1. Kiến trúc logic của Shopsflow. Bản cấu hình triển khai sử dụng Region `us-east-1`._

**Request flow**

1. User truy cập website qua CloudFront.
2. CloudFront lấy React/Vite static artifact từ S3.
3. Request đến `/api/*` được CloudFront forward đến ALB.
4. ALB route request healthy đến ECS Fargate task trong private subnet.
5. Spring Boot backend đọc và ghi dữ liệu trên RDS PostgreSQL.

**Payment flow**

Backend tạo payment parameter, gọi payment provider qua NAT Gateway và trả Payment URL về cho user. Sau khi provider phản hồi, backend xác minh signature cùng payment status trước khi cập nhật Order.

#### 4. Dịch vụ AWS và cấu hình dự án

| Phạm vi            | Dịch vụ và cấu hình                                                                             |
| ------------------ | ----------------------------------------------------------------------------------------------- |
| Region             | `us-east-1`                                                                                     |
| Network            | `shopsflow-vpc` với hai public subnet và hai private subnet trên `us-east-1a` cùng `us-east-1b` |
| Frontend           | Amazon S3 private origin và CloudFront distribution `fe cloudfront`                             |
| Public domain      | `d2m34udjfc5fxq.cloudfront.net`                                                                 |
| API entry point    | Internet-facing ALB `shopsflow-alb`                                                             |
| Container registry | Amazon ECR repository `shopsflow-repo`                                                          |
| Compute            | ECS cluster `shopsflow-cluster`, Fargate, `awsvpc`, private subnet, public IP disabled          |
| Container port     | `8080`, sử dụng IP target group                                                                 |
| Database           | RDS PostgreSQL `database-shopsflow`, private DB subnet group, port `5432`                       |
| Observability      | CloudWatch log group `/shopsflow/ecs/backend` qua `awslogs` driver                              |

#### 5. Bảo mật và vận hành

Security group thực thi traffic path `CloudFront → ALB → ECS → RDS`:

- `alb-sg` nhận HTTP `80` và HTTPS `443`, sau đó chỉ gửi backend traffic đến `ecs-sg` qua port `8080`.
- `ecs-sg` chỉ nhận port `8080` từ `alb-sg`.
- `rds-sg` chỉ nhận PostgreSQL port `5432` từ `ecs-sg`.

ECS sử dụng task execution role và task role riêng. Execution role pull image từ ECR và gửi log đến CloudWatch; task role chỉ chứa application permission. Database credential được truyền tại thời điểm triển khai thay vì lưu trong source code.

CloudWatch được sử dụng để theo dõi ECS task start và stop, Spring Boot error, database connectivity, payment failure, ALB target health và deployment revision.

#### 6. Kế hoạch triển khai

1. Tạo VPC, subnet, Internet Gateway, NAT Gateway, route table và security group.
2. Tạo ECS task execution role và task role với least-privilege access.
3. Tạo RDS PostgreSQL instance cùng private DB subnet group.
4. Build backend image, push lên ECR và tạo ECS Fargate service với ALB health check.
5. Build React/Vite frontend, upload `dist` artifact lên S3 và cấu hình CloudFront với ALB behavior `/api/*`.
6. Cấu hình CloudWatch logging, xác thực payment cùng API flow, sau đó review chi phí và resource usage.

Mỗi lần phát hành backend tạo ECR image và ECS task-definition revision mới. ECS service rollout có health check thay vì thay đổi file trong container đang chạy.

#### 7. Rủi ro và giảm thiểu

| Rủi ro                             | Giảm thiểu                                                                                               |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------- |
| ECS task unhealthy                 | Dùng ALB health check, ECS stopped reason và CloudWatch Logs để tách lỗi startup, routing hoặc database. |
| Database bị lộ                     | Giữ RDS ở private tier và giới hạn port `5432` cho `ecs-sg`.                                             |
| Payment status không chính xác     | Xác minh signature và status của provider tại backend; không chỉ tin client redirect.                    |
| Chi phí còn phát sinh sau khi test | Theo dõi NAT Gateway, ALB, Fargate, RDS, CloudWatch và data transfer; cleanup theo dependency order.     |
| Redeployment lỗi                   | Lưu ECR image theo version và dùng task-definition revision mới cho mỗi lần release.                     |

#### 8. Kết quả kỳ vọng

Đề xuất tạo ra kiến trúc thương mại điện tử có thể triển khai với frontend qua CDN, backend container private, PostgreSQL database private, API routing có health check, log có cấu trúc và payment flow được kiểm soát. Đồng thời, kiến trúc cung cấp mô hình vận hành rõ ràng để xác thực release, troubleshoot failure và cleanup resource nhằm tránh chi phí không cần thiết.
