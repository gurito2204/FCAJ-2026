---
title: "Dọn dẹp tài nguyên"
date: 2026-06-15
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Kiến trúc Doanh nghiệp sẵn sàng cao (Enterprise HA) sử dụng các dịch vụ có phí duy trì theo giờ (như NAT Gateway, Application Load Balancer). Hãy chắc chắn thực hiện dọn dẹp tài nguyên theo thứ tự dưới đây để tránh các chi phí phát sinh ngoài mong muốn trên tài khoản AWS của bạn.

---

### Các bước dọn dẹp tài nguyên theo thứ tự

#### 1. Xóa CloudFront & WAF
1. Truy cập **AWS Console** -> **CloudFront**.
2. Chọn distribution của bạn -> Click **Disable** và đợi trạng thái chuyển sang disabled.
3. Sau khi bị vô hiệu hóa, tích chọn distribution và click **Delete**.
4. Truy cập **AWS WAF** -> **Web ACLs** -> Chọn Web ACL đã tạo -> Click **Delete**.

#### 2. Xóa Application Load Balancer (ALB)
1. Truy cập **EC2** -> **Load Balancers**.
2. Chọn Load Balancer `shopsflow-backend-alb` -> Click **Actions** -> **Delete**.
3. Chọn **Target Groups** ở menu trái -> Chọn target group `shopsflow-backend-tg` -> Click **Actions** -> **Delete**.

#### 3. Xóa Auto Scaling Group (ASG) & Launch Template
1. Truy cập **EC2** -> **Auto Scaling Groups**.
2. Chọn `shopsflow-backend-asg` -> Click **Delete**. 
   * *Lưu ý:* Việc xóa ASG sẽ tự động tiến hành tắt máy (Terminate) toàn bộ các EC2 instances trong Private subnets mà nó quản lý. Đợi quá trình kết thúc hoàn toàn.
3. Chọn **Launch Templates** ở menu trái -> Chọn `shopsflow-backend-template` -> Click **Actions** -> **Delete template**.

#### 4. Xóa Database RDS PostgreSQL Multi-AZ
1. Truy cập **RDS** -> **Databases**.
2. Chọn database `shopsflow-db` -> Click **Actions** -> **Delete**.
3. Tại bảng xác nhận:
   * Bỏ chọn *Create final snapshot?*
   * Tích chọn *I acknowledge...*
   * Nhập chữ `delete me` vào ô xác nhận.
   * Chọn **Delete** và đợi database được xóa hoàn toàn.
4. Chọn **Subnet groups** ở menu trái -> Chọn `shopsflow-db-subnet-group` -> Chọn **Delete**.

#### 5. Xóa các S3 Buckets
1. Truy cập **S3**.
2. Thực hiện làm rỗng (**Empty**) và sau đó xóa (**Delete**) lần lượt 2 buckets:
   * Bucket frontend: `shopsflow-frontend-static-999`
   * Bucket backup database: `shopsflow-db-backup-999`

#### 6. Xóa Secrets Manager & KMS Key
1. Truy cập **Secrets Manager** -> Chọn secret `shopsflow/production/secrets` -> Chọn **Actions** -> Chọn **Delete secret** -> Cấu hình thời gian xóa tối thiểu (7 ngày).
2. Truy cập **KMS** -> Chọn key `shopsflow-kms-key` -> Chọn **Key actions** -> Chọn **Schedule key deletion** -> Chọn thời gian tối thiểu 7 ngày.

#### 7. Giải phóng tài nguyên Mạng (NAT Gateway & VPC)
Khi các tài nguyên điện toán và database đã được xóa sạch, chúng ta tiến hành xóa tài nguyên mạng:
1. **Xóa NAT Gateway:**
   * Truy cập **VPC** -> **NAT Gateways**.
   * Chọn NAT `shopsflow-nat-gw` -> Click **Delete NAT gateway** và đợi trạng thái chuyển sang *Deleted*.
2. **Giải phóng Elastic IP:**
   * Chọn **Elastic IPs** ở menu trái -> Chọn địa chỉ IP tĩnh đã cấp cho NAT -> Click **Actions** -> Chọn **Release Elastic IP addresses**.
3. **Xóa VPC Endpoints:**
   * Chọn **Endpoints** ở menu trái -> Chọn S3 Gateway Endpoint -> Click **Actions** -> Chọn **Delete VPC endpoints**.
4. **Xóa Custom VPC:**
   * Chọn **Your VPCs** ở menu trái.
   * Chọn VPC `shopsflow-vpc` -> Click **Actions** -> Chọn **Delete VPC**.
   * *Lưu ý:* Khi xóa VPC, AWS sẽ tự động xóa tất cả các subnets, route tables, network ACLs và internet gateway đi kèm với VPC đó.

---

### Tự đánh giá & Bài học rút ra (Reflection)

#### 1. Khó khăn gặp phải
* **Độ phức tạp của hạ tầng mạng:** Việc chia VPC thành 6 subnets phân bổ trên 2 Availability Zones yêu cầu cấu hình Route Tables rất chính xác. Lúc đầu, các instance trong Private Subnets không kết nối được với internet để cập nhật packages và tải mã nguồn do cấu hình sai Route Table định tuyến đến NAT Gateway.
* **Tích hợp Secrets Manager & KMS:** Quá trình cấu hình User Data cho EC2 tự động kéo database credentials và JWT secret về giải mã rất dễ gặp lỗi bất đối xứng quyền (KMS key policy chưa cho phép IAM Role của EC2 sử dụng khóa để decrypt).
* **Định tuyến CloudFront & CORS:** Cấu hình gộp chung Frontend S3 và Backend ALB dưới một distribution CloudFront với đường dẫn `/api/*` đòi hỏi thiết lập Cache Policy và Origin Request Policy chuẩn xác để tránh lỗi CORS và mất Headers khi gửi về API Backend.

#### 2. Bài học rút ra
* **Thấu hiểu thực tế kiến trúc High Availability (HA):** Bài lab giúp em nắm vững cách hoạt động thực tế của Application Load Balancer và Auto Scaling Group trong việc phân chia lưu lượng, tự động phát hiện và thay thế máy chủ lỗi (Self-healing).
* **Bảo mật chuyên sâu (Defense in Depth):** Hiểu rõ tầm quan trọng của việc cô lập hoàn toàn cơ sở dữ liệu và ứng dụng backend trong Private Subnet, chỉ cho phép traffic đi qua các chốt chặn được kiểm soát (WAF, CloudFront, ALB).
* **Tối ưu hóa chi phí với VPC Endpoint:** Học cách sử dụng Gateway VPC Endpoint để truyền tải dữ liệu dung lượng lớn (Backup database) lên S3 qua mạng nội bộ mà không cần đi qua NAT Gateway, tránh phát sinh chi phí băng thông NAT rất đắt đỏ.

#### 3. Hướng phát triển trong tương lai
* **Infrastructure as Code (IaC):** Sử dụng các công cụ IaC như **Terraform** hoặc **AWS CDK** để định nghĩa toàn bộ hạ tầng này dưới dạng mã nguồn, giúp triển khai tự động, nhất quán và tránh các sai sót thủ công khi click trên AWS Console.
* **Triển khai Serverless/Container nâng cao:** Tìm hiểu cách đưa Spring Boot Backend lên chạy trên **AWS ECS Fargate** để loại bỏ việc quản trị và cấu hình hệ điều hành cho máy chủ EC2, hướng tới kiến trúc Cloud-native hiện đại hơn.