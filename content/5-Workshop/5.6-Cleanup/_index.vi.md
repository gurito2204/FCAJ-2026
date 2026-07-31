---
title: "Dọn dẹp tài nguyên"
date: 2026-06-15
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Kiến trúc Doanh nghiệp sẵn sàng cao (Enterprise HA) sử dụng các dịch vụ có phí duy trì theo giờ (như NAT Gateway, Application Load Balancer, Fargate task, RDS). Hãy chắc chắn thực hiện dọn dẹp tài nguyên theo thứ tự dưới đây để tránh các chi phí phát sinh ngoài mong muốn trên tài khoản AWS của bạn.

---

### Các bước dọn dẹp tài nguyên theo thứ tự

#### 1. Xóa CloudFront & WAF
1. **CloudFront** → chọn distribution → **Disable**, đợi chuyển trạng thái xong.
2. Sau khi disabled, tick chọn → **Delete**.
3. **AWS WAF → Web ACLs** → chọn Web ACL đã tạo → **Delete**.

#### 2. Xóa ECS Service, Task và Cluster
1. **ECS → Clusters** → chọn `shopsflow-cluster` → **Services**.
2. Chọn `shopsflow-backend-service` → set **Desired tasks = 0**, hoặc **Delete service**.
3. Đợi toàn bộ Fargate task chuyển **STOPPED**.
4. Khi cluster không còn service/task nào, xóa `shopsflow-cluster`.

#### 3. Xóa Application Load Balancer
1. **EC2 → Load Balancers** → chọn `shopsflow-backend-alb` → **Actions → Delete**.
2. **Target Groups** → chọn `shopsflow-backend-tg` → **Actions → Delete**.

#### 4. Xóa ECR
1. **ECR → Repositories** → chọn `shopsflow-repo`.
2. Nếu cleanup toàn bộ: xóa hết image trong repo trước, sau đó xóa repository. Nếu muốn giữ lại image làm record, có thể bỏ qua bước này.

#### 5. Xóa Database RDS PostgreSQL Multi-AZ
1. **RDS → Databases** → chọn `shopsflow-db` → **Actions → Delete**.
2. Cân nhắc **Create final snapshot?** trước khi xóa hẳn nếu muốn giữ lại dữ liệu.
3. Xác nhận xóa, đợi hoàn tất.
4. **Subnet groups** → chọn `shopsflow-db-subnet-group` → **Delete**.

#### 6. Xóa các S3 Buckets
* Làm rỗng (**Empty**) rồi xóa (**Delete**) bucket frontend `shopsflow-frontend-static-999` nếu không cần giữ lại.

#### 7. Xóa Secrets Manager & KMS Key
1. **Secrets Manager** → chọn `shopsflow/production/secrets` → **Actions → Delete secret** (thời gian chờ tối thiểu 7 ngày).
2. **KMS** → chọn `shopsflow-kms-key` → **Key actions → Schedule key deletion** (tối thiểu 7 ngày).

#### 8. Xóa IAM Roles
* Xóa `shopsflow-ecs-task-execution-role` và `shopsflow-ecs-task-role` — chỉ xóa sau khi chắc chắn ECS không còn tham chiếu tới 2 role này.

#### 9. Giải phóng tài nguyên mạng (NAT Gateway & VPC)
1. **NAT Gateways:** xóa `shopsflow-nat-gw-1` và `shopsflow-nat-gw-2`, đợi chuyển **Deleted**.
2. **Elastic IPs:** release 2 địa chỉ IP đã cấp cho NAT Gateway.
3. **VPC:** sau khi ALB, RDS, ECS network interface và NAT Gateway đã gỡ hết, xóa route table tùy chỉnh, detach & xóa `shopsflow-igw`, xóa subnet + security group tùy chỉnh, cuối cùng xóa `shopsflow-vpc`.

---

### Hướng phát triển trong tương lai

* **Infrastructure as Code (IaC):** dùng Terraform hoặc AWS CDK để định nghĩa toàn bộ hạ tầng dưới dạng mã nguồn, tránh sai sót khi thao tác thủ công trên Console.
* **CI/CD tự động:** dùng GitHub Actions để tự động build & push image lên ECR, đăng ký task-definition revision mới và cập nhật ECS service mỗi khi merge vào `main`, thay cho việc deploy thủ công.