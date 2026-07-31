---
title: "Triển khai Frontend & Backend API"
date: 2026-06-15
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Bài viết này hướng dẫn triển khai tách biệt phần **Frontend** (S3 + CloudFront CDN + WAF) và phần **Backend** chạy trên nhóm máy chủ EC2 tự động co giãn (**Application Load Balancer + Auto Scaling Group**) đặt trong Private Subnets, kết hợp cấu hình **VPC S3 Gateway Endpoint**.

---

### 1. Triển khai Web Frontend (S3 + CloudFront CDN + WAF)

Chúng ta lưu trữ mã nguồn tĩnh của Web Frontend trên **Amazon S3** và phân phối bảo mật qua **Amazon CloudFront** với kiểm soát truy cập biên **Origin Access Control (OAC)**.

#### Bước 1: Tạo S3 Bucket cho Frontend
1. Truy cập **AWS Console** -> **S3** -> Chọn **Create bucket**.
2. **Bucket name:** `shopsflow-frontend-static-999` (nhập tên duy nhất toàn cầu).
3. Cấu hình mặc định: **Block all public access** (Bảo mật - chúng ta không mở public bucket, mà chỉ cho phép CloudFront đọc thông tin).
4. Chọn **Create bucket**. Sau đó, tải các tệp tĩnh đã build của React Frontend (`dist/` folder chứa `index.html`, `assets/`,...) lên bucket này.

#### Bước 2: Tạo CloudFront Distribution
1. Truy cập **CloudFront** -> Chọn **Create distribution**.
2. **Origin:**
   * **Origin domain:** Chọn bucket S3 `shopsflow-frontend-static-999.s3.amazonaws.com`.
   * **Origin access:** Chọn **Origin access control settings (recommended)** -> Chọn **Create control setting** -> Chọn **Sign requests (recommended)** và click **Create**.
3. **Default cache behavior:**
   * **Viewer protocol policy:** Chọn **Redirect HTTP to HTTPS**.
4. **Web Application Firewall (WAF):**
   * Chọn **Enable security protections** -> Chọn **Create Web ACL** (Đính kèm AWS WAF để bảo vệ website khỏi tấn công mạng phổ biến).
5. Chọn **Create distribution**.
6. **Cập nhật S3 Bucket Policy:**
   * Sau khi tạo xong, CloudFront sẽ hiển thị thông báo yêu cầu cập nhật Bucket Policy. Copy đoạn policy hiển thị.
   * Quay lại S3 Bucket `shopsflow-frontend-static-999` -> Tab **Permissions** -> **Bucket policy** -> Chọn **Edit** -> Dán đoạn policy đã copy (cấp quyền cho dịch vụ CloudFront truy cập đọc đối tượng S3) và lưu lại.

---

### 2. Triển khai Load Balancing & Auto Scaling cho Backend

Chúng ta sẽ thiết lập một **Application Load Balancer (ALB)** công cộng để nhận API requests, sau đó chuyển tiếp cho nhóm máy chủ EC2 đặt trong Private Subnet quản lý bởi **Auto Scaling Group (ASG)**.

#### Bước 1: Tạo Application Load Balancer
1. Truy cập **EC2** -> **Load Balancers** -> Chọn **Create load balancer** -> Chọn **Application Load Balancer**.
2. Thiết lập:
   * **Load balancer name:** `shopsflow-backend-alb`
   * **Scheme:** **Internet-facing** (ALB nhận traffic từ internet chuyển tiếp vào subnet riêng).
   * **Network mapping:** Chọn VPC `shopsflow-vpc`.
   * **Mappings:** Chọn 2 **Public Subnets** (`shopsflow-public-1` và `shopsflow-public-2`) ở 2 AZs khác nhau.
3. **Security groups:** Chọn `shopsflow-alb-sg`.
4. **Listeners and routing:**
   * Protocol: `HTTP`, Port: `80`.
   * **Default action:** Chọn **Create target group** để định cấu hình chuyển tiếp:
     * *Target type:* **Instances**
     * *Target group name:* `shopsflow-backend-tg`
     * *Protocol:* `HTTP`, Port: `8080` (Cổng Spring Boot ứng dụng lắng nghe).
     * *VPC:* Chọn `shopsflow-vpc`.
     * *Health checks:* Path `/api/products` (hoặc endpoint kiểm tra trạng thái của API).
     * Click **Create target group**.
   * Quay lại màn hình tạo ALB, F5 để chọn target group `shopsflow-backend-tg` vừa tạo.
5. Chọn **Create load balancer** và ghi lại DNS Name của ALB.

#### Bước 2: Thiết lập Launch Template cho EC2 Backend
Launch Template định nghĩa cấu hình chi tiết cho các EC2 instances sẽ được Auto Scaling tự động tạo ra.

1. Truy cập **EC2** -> **Launch Templates** -> Chọn **Create launch template**.
2. Thiết lập:
   * **Launch template name:** `shopsflow-backend-template`
   * **OS Image (AMI):** **Ubuntu Server 24.04 LTS**.
   * **Instance type:** `t3.micro`.
   * **Key pair:** Chọn key pair SSH của bạn.
   * **Network settings:** Chọn Security Group `shopsflow-ec2-sg`.
   * **Advanced details:**
     * **IAM instance profile:** Chọn `ShopsflowEC2Role`.
     * **User data (Script tự động cấu hình chạy lúc boot instance):**
       Sao chép đoạn script tự động lấy Secrets từ Secrets Manager và chạy ứng dụng Spring Boot Backend:
       ```bash
       #!/bin/bash
       apt-get update -y
       apt-get install -y docker.io awscli jq git
       systemctl enable --now docker
       usermod -aG docker ubuntu

       # Tạo thư mục và cấu hình CloudWatch Agent
       mkdir -p /opt/aws/amazon-cloudwatch-agent/etc/

       # Tải và cập nhật Secrets từ AWS Secrets Manager bằng IAM Role
       SECRET_RAW=$(aws secretsmanager get-secret-value --secret-id shopsflow/production/secrets --region ap-southeast-1 --query SecretString --output text)
       DB_PASS=$(echo $SECRET_RAW | jq -r .SPRING_DATASOURCE_PASSWORD)
       JWT_KEY=$(echo $SECRET_RAW | jq -r .JWT_SECRET)

       # Clone code và build
       git clone <LINK_GITHUB_CUA_BAN> /home/ubuntu/shopsflow
       cd /home/ubuntu/shopsflow/deploy/aws
       
       # Tạo file cấu hình môi trường
       cat <<EOF > .env.aws
       SPRING_DATASOURCE_URL=jdbc:postgresql://<RDS_ENDPOINT>:5432/shopsflow
       SPRING_DATASOURCE_USERNAME=postgres
       SPRING_DATASOURCE_PASSWORD=$DB_PASS
       JWT_SECRET=$JWT_KEY
       APP_SEED_DEMO_DATA=true
       EOF

       # Cài đặt docker compose
       mkdir -p ~/.docker/cli-plugins/
       curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
       chmod +x ~/.docker/cli-plugins/docker-compose

       # Triển khai backend container
       /usr/local/bin/docker-compose --env-file .env.aws -f docker-compose.aws.yml up -d backend-service
       ```
       *(Thay thế `<RDS_ENDPOINT>` bằng endpoint RDS thật thu được ở chương trước và điền link GitHub của bạn).*
3. Chọn **Create launch template**.

#### Bước 3: Khởi tạo Auto Scaling Group (ASG)
1. Truy cập **EC2** -> **Auto Scaling Groups** -> Chọn **Create Auto Scaling group**.
2. **Name:** `shopsflow-backend-asg`.
3. **Launch template:** Chọn template `shopsflow-backend-template` vừa tạo -> Chọn **Next**.
4. **Network:**
   * **VPC:** Chọn `shopsflow-vpc`.
   * **Subnets:** Tick chọn 2 subnets riêng tư **Private App Subnet 1** (`shopsflow-private-app-1`) và **Private App Subnet 2** (`shopsflow-private-app-2`). Điều này đảm bảo EC2 chỉ chạy trong môi trường mạng riêng cô lập.
5. **Configure advanced options:**
   * Tick chọn **Load balancing** -> Chọn **Attach to an existing load balancer**.
   * Chọn Target group `shopsflow-backend-tg` -> Chọn **Next**.
6. **Group size and scaling policies:**
   * **Desired capacity:** `2` (Chạy mặc định 2 instance ở 2 AZs khác nhau để đảm bảo tính sẵn sàng cao).
   * **Min capacity:** `1`, **Max capacity:** `4`.
   * **Scaling policies:** Chọn **Target tracking scaling policy** -> Metric type: *Average CPU utilization*, Target value: `70` (Tự động scale up khi CPU trung bình quá 70%).
7. Hoàn tất các bước và chọn **Create Auto Scaling group**. Hệ thống sẽ tự động khởi chạy 2 máy chủ EC2 backend đặt trong Private subnets.

---

### 3. Cấu hình định tuyến CloudFront cho API Backend

Để gộp chung giao diện frontend tĩnh và các api backend vào cùng một tên miền duy nhất (tránh lỗi CORS), chúng ta cấu hình định tuyến bổ sung trên CloudFront.

1. Quay lại trang quản lý **CloudFront** -> Chọn distribution của bạn.
2. Tại tab **Origins** -> Chọn **Create origin**:
   * **Origin domain:** Chọn DNS Name của Application Load Balancer `shopsflow-backend-alb`.
   * **Protocol:** Chọn `HTTP Only` (ALB nhận traffic HTTP chuyển tiếp).
   * Chọn **Create origin**.
3. Tại tab **Behaviors** -> Chọn **Create behavior**:
   * **Path pattern:** `/api/*` (Bất kỳ request nào bắt đầu bằng /api/ sẽ được chuyển cho backend xử lý).
   * **Target origin:** Chọn Load Balancer origin vừa tạo.
   * **Allowed HTTP methods:** Chọn `GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE`.
   * **Cache policy:** Chọn **CachingDisabled** (Để dữ liệu động từ API không bị lưu cache).
   * **Origin request policy:** Chọn **AllViewerAndCloudFrontHeaders-2022-06**.
   * Chọn **Create behavior**.

---

### 4. Tạo Gateway VPC Endpoint cho S3

Vì máy chủ EC2 đặt trong Private Subnet, việc kết nối trực tiếp ra S3 qua Internet để backup dữ liệu bình thường phải đi qua NAT Gateway và chịu phí truyền tải dữ liệu rất đắt đỏ. Chúng ta tạo một VPC Endpoint loại Gateway để kết nối an toàn và miễn phí qua mạng nội bộ AWS.

1. Truy cập **VPC** -> **Endpoints** -> Chọn **Create endpoint**.
2. **Service category:** Chọn **AWS services**.
3. **Services:** Tìm kiếm và chọn dịch vụ `com.amazonaws.ap-southeast-1.s3` có Type là **Gateway**.
4. **VPC:** Chọn `shopsflow-vpc`.
5. **Route tables:** Tick chọn bảng định tuyến `shopsflow-private-rt` (Bảng định tuyến đang quản lý Private App Subnets).
   * *Ý nghĩa:* AWS sẽ tự động thêm một rule định tuyến trong route table của EC2: Mọi traffic đến S3 sẽ được định tuyến trực tiếp qua Endpoint này thay vì chuyển ra ngoài Internet.
6. Chọn **Create endpoint**.
