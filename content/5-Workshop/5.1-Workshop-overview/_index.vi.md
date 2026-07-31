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
* **Giảm downtime và rủi ro triển khai:** Khắc phục tình trạng xung đột môi trường (lỗi phiên bản thư viện giữa máy local và máy chủ) bằng công nghệ container hóa (Docker), đóng gói ứng dụng thành image và chạy trên **Amazon ECS (Fargate)** thay vì cài đặt trực tiếp trên máy chủ.
* **Bảo mật dữ liệu:** Ngăn ngừa việc rò rỉ dữ liệu khách hàng bằng cách đưa tác vụ container backend và cơ sở dữ liệu vào vùng mạng riêng cô lập (Private Subnets).
* **Bảo toàn dữ liệu:** Tự động hóa quy trình sao lưu (backup) cơ sở dữ liệu PostgreSQL định kỳ, tránh mất mát thông tin khi có sự cố.
* **Khả năng giám sát tập trung:** Tập trung hóa toàn bộ log ứng dụng và các thông số hệ thống lên CloudWatch để dễ dàng xử lý sự cố.
#### Mục tiêu cụ thể
* **Đầu ra mong muốn (Outputs):**
  * **Frontend Web:** React + Vite Single Page Application (SPA) được deploy tĩnh trên Amazon S3 và phân phối qua Amazon CloudFront CDN.
  * **Backend API:** Spring Boot RESTful API được đóng gói bằng Docker, lưu image trên **Amazon ECR (Elastic Container Registry)** và chạy dưới dạng task trên **Amazon ECS với launch type Fargate** (serverless container, không cần tự quản lý EC2), đặt trong Private Subnet, phía sau Application Load Balancer (ALB).
  * **Database RDS:** PostgreSQL Database trên Amazon RDS, đặt trong Private Subnet, tắt hoàn toàn khả năng truy cập công cộng.
  * **Security & Encryption:** Sử dụng AWS Secrets Manager và KMS để lưu mật khẩu và cấu hình nhạy cảm.
  * **Monitoring System:** Dashboard giám sát và logs tập trung trên CloudWatch cho cả ECS task lẫn RDS.
#### Phù hợp chương trình
Dự án sử dụng các dịch vụ nền tảng cơ bản và nâng cao của AWS bao gồm: **VPC**, **ECS (Fargate)**, **ECR**, **RDS**, **CloudFront**, **S3**, **Application Load Balancer**, **Secrets Manager**, **KMS**, **CloudWatch**, và **IAM**. Cấu trúc hạ tầng tuân thủ nguyên tắc thiết kế bảo mật và sẵn sàng cao của AWS (Well-Architected Framework), rất phù hợp làm đề tài thực hành thực tế cho học viên trong chương trình First Cloud Journey (FCJ).
 
---
 
### 2. Sơ đồ kiến trúc & Thiết kế kỹ thuật
 
#### Sơ đồ kiến trúc (Architecture Diagram)
 
Dưới đây là sơ đồ kiến trúc mô tả cấu trúc phân tầng và luồng dữ liệu của ứng dụng Shopsflow khi triển khai trên hạ tầng AWS:
 
![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.png)
 
#### Usecase
| STT | Use case | API sử dụng | Công dụng |
|---|---|---|---|
| 1 | Đăng ký tài khoản | POST `/api/auth/register` | Tạo tài khoản mới, trả về JWT để đăng nhập luôn |
| 2 | Đăng nhập | POST `/api/auth/login` | Xác thực email/password, trả về JWT dùng cho các request sau |
| 3 | Tìm kiếm & lọc sản phẩm | GET `/api/products` | Xem danh sách sản phẩm, hỗ trợ `keyword`, `categoryId`, `minPrice`, `maxPrice`, phân trang & sắp xếp |
| 4 | Xem chi tiết sản phẩm | GET `/api/products/{id}` | Lấy thông tin 1 sản phẩm cụ thể |
| 5 | Tạo sản phẩm (Admin) | POST `/api/products` | Admin thêm sản phẩm mới vào catalog |
| 6 | Cập nhật sản phẩm (Admin) | PUT `/api/products/{id}` | Admin chỉnh sửa thông tin sản phẩm |
| 7 | Xoá sản phẩm (Admin) | DELETE `/api/products/{id}` | Admin gỡ sản phẩm khỏi catalog |
| 8 | Xem đánh giá sản phẩm | GET `/api/products/{productId}/reviews` | Xem danh sách review của 1 sản phẩm (public) |
| 9 | Viết đánh giá | POST `/api/products/{productId}/reviews` | User đăng review (sao 1–5, comment tối đa 2000 ký tự), mỗi user chỉ review 1 lần/sản phẩm |
| 10 | Sửa đánh giá | PUT `/api/reviews/{reviewId}` | Chủ review chỉnh sửa review của chính mình |
| 11 | Xoá đánh giá | DELETE `/api/reviews/{reviewId}` | Chủ review xoá review của chính mình |
| 12 | Xem danh mục | GET `/api/categories` | Xem toàn bộ danh mục sản phẩm (public) |
| 13 | Xem chi tiết danh mục | GET `/api/categories/{id}` | Xem 1 danh mục cụ thể |
| 14 | Tạo danh mục (Admin) | POST `/api/categories` | Admin thêm danh mục mới |
| 15 | Cập nhật danh mục (Admin) | PUT `/api/categories/{id}` | Admin sửa danh mục |
| 16 | Xoá danh mục (Admin) | DELETE `/api/categories/{id}` | Admin xoá danh mục |
| 17 | Xem giỏ hàng | GET `/api/cart` | User xem giỏ hàng hiện tại của mình |
| 18 | Thêm vào giỏ hàng | POST `/api/cart/items` | User thêm sản phẩm vào giỏ |
| 19 | Cập nhật số lượng trong giỏ | PUT `/api/cart/items/{itemId}` | User chỉnh số lượng sản phẩm trong giỏ |
| 20 | Xoá sản phẩm khỏi giỏ | DELETE `/api/cart/items/{itemId}` | User xoá 1 item khỏi giỏ hàng |
| 21 | Đặt hàng (checkout) | POST `/api/orders` | Chuyển giỏ hàng thành đơn hàng, trạng thái ban đầu `PENDING` |
| 22 | Xem đơn hàng của tôi | GET `/api/orders` | User xem danh sách đơn hàng của chính mình |
| 23 | Xem chi tiết đơn hàng | GET `/api/orders/{id}` | User xem chi tiết 1 đơn hàng của mình |
| 24 | Cập nhật trạng thái đơn (Admin) | PUT `/api/orders/{id}/status` | Admin cập nhật trạng thái đơn hàng (vd: đã giao, huỷ...) |
| 25 | Xem tất cả đơn hàng (Admin) | GET `/api/orders/all` | Admin xem toàn bộ đơn hàng trong hệ thống |
| 26 | Khởi tạo thanh toán VNPay | POST `/api/payment/vnpay/checkout/{orderId}` | Tạo link thanh toán VNPay Sandbox (`payUrl`) cho đơn hàng |
| 27 | Nhận kết quả thanh toán (redirect) | GET `VNPAY_RETURN_URL` (`/payment-result`, phía frontend) | VNPay redirect trình duyệt về kèm query params kết quả giao dịch |
| 28 | Xác nhận thanh toán (IPN webhook) | `VNPAY_IPN_URL` (server-to-server) | VNPay gọi ngầm về backend để cập nhật trạng thái đơn hàng chính xác, bất đồng bộ |
| 29 | Đối soát trạng thái đơn sau thanh toán | GET `/api/orders/{orderId}` | Frontend gọi lại để lấy trạng thái chính thức (`PAID`/`PENDING`/`CANCELLED`) từ DB |
**Luồng runtime (theo số trong sơ đồ):**
- 1–2: User → CloudFront → S3 (serve frontend)
- 3–6: User → Internet Gateway → ALB → ECS Cluster → Fargate (gọi API)
- 7–8: BackEnd Container ↔ VNPay: tạo Payment URL, nhận Payment Status callback

| Service | Ứng dụng |
|---|---|
| CloudFront + S3 | Static hosting + CDN cho frontend |
| Internet Gateway + NAT Gateway | Kết nối internet cho public/private subnet |
| VPC (public/private subnet, 2 AZ) | Network isolation, multi-tier |
| ALB | Load balancing vào backend |
| ECS Cluster + Fargate | Container orchestration, chạy BackEnd container |
| RDS (2 AZ) | Managed database |
| ECR | Private registry chứa Docker image |
| CloudWatch | Monitoring/logging |
| IAM | Access control |
| GitHub + Docker | Build & push image lên ECR |
 
#### Lựa chọn dịch vụ (Service Selection Rationale)
 
* **Amazon CloudFront & Amazon S3 (Frontend):**
  * *Lý do chọn:* Đưa web tĩnh (HTML/JS/CSS/Ảnh) lên S3 và phân phối qua CloudFront giúp giảm tải hoàn toàn cho backend, tăng tốc độ tải trang toàn cầu nhờ bộ nhớ đệm (Caching) tại các Edge Location, và tối ưu chi phí.
* **Amazon ECS (Fargate) & Amazon ECR (Backend):**
  * *Lý do chọn:* Đóng gói backend Spring Boot thành Docker image, lưu trữ trên ECR và chạy dưới dạng task trên ECS Fargate — mô hình serverless container giúp không phải tự quản lý, vá lỗi hay scale thủ công các EC2 instance bên dưới. ALB phân phối traffic vào các ECS task đặt trong Private Subnet, và ECS Service có thể tự động điều chỉnh số lượng task chạy song song dựa trên tải hệ thống.
* **Amazon RDS PostgreSQL:**
  * *Lý do chọn:* Dịch vụ database quản lý (managed service) giúp giảm gánh nặng vận hành (patching, backup, failover) so với tự cài PostgreSQL trên máy chủ, đồng thời đặt hoàn toàn trong Private Subnet để không expose ra Internet.
* **AWS Secrets Manager & KMS:**
  * *Lý do chọn:* Lưu trữ và quản lý tập trung các thông tin nhạy cảm (Database Password, JWT Secret) được mã hóa bằng AWS KMS. ECS Task sẽ tự động lấy credentials tạm thời lúc runtime thay vì lưu file cấu hình thô trong image.
* **Application Load Balancer:**
  * *Lý do chọn:* Phân phối lưu lượng truy cập từ Internet Gateway tới các ECS task trong Private Subnet, đảm bảo tách biệt hoàn toàn giữa tầng public và tầng backend nội bộ.
#### Bảo mật & IAM (Security & Access Control)
* **IAM Task Role / Execution Role:** Gán IAM Role cho ECS Task, cho phép đọc Secrets từ Secrets Manager, kéo image từ ECR, và gửi log/metric lên CloudWatch — theo nguyên tắc least privilege (không dùng quyền của EC2 instance vì backend không chạy trực tiếp trên EC2).
* **Mạng cô lập (Network Isolation):** ECS Task và RDS nằm hoàn toàn trong Private Subnets, không có địa chỉ IP công cộng. Security Groups được thiết lập theo dạng chuỗi: `Internet` → `CloudFront` → `Internet Gateway` → `ALB SG` → `ECS Task SG` → `RDS SG`.