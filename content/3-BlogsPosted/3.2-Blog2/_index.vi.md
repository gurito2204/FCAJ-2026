---
title: "Blog 2"
date: 2026-06-25
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# NHỮNG "CÁI BẪY" ẨN KỸ TRONG AWS MÀ TÀI LIỆU CHÍNH THỨC ÍT KHI CẢNH BÁO BẠN

Khi mới bắt đầu tiếp cận AWS, chúng ta thường nghe nhiều về các khái niệm quen thuộc như EC2, S3, RDS hay các mô hình kiến trúc chuẩn chỉnh trên slide. Nhưng chỉ đến khi trực tiếp vận hành hệ thống thực tế và tự tay trả giá bằng tiền thật hoặc những sự cố lúc rạng sáng, mình mới nhận ra có những góc khuất rất ít khi được nhắc tới trong các khóa học cơ bản.

Dưới đây là 4 bài học thực chiến, liên quan đến các cơ chế vận hành ngầm của AWS mà mình rút ra được sau nhiều lần "trả phí bản quyền cho sự thiếu hiểu biết".

---

### 1. NAT Gateway và "cú lừa" chi phí khi làm việc với S3

Rất nhiều tài liệu khuyên bạn đặt EC2 trong Private Subnet để đảm bảo an toàn, sau đó dùng NAT Gateway để EC2 có thể đi ra Internet. Kiến trúc này hoàn toàn đúng, cho đến khi ứng dụng của bạn bắt đầu đọc và ghi dữ liệu liên tục với S3 (lưu log, backup, upload file media).

*   **Vấn đề ngầm:** Theo mặc định, traffic từ EC2 đi tới S3 sẽ chạy vòng qua NAT Gateway. NAT Gateway không chỉ tính phí theo giờ hoạt động mà còn thu phí trên mỗi GB dữ liệu xử lý đi qua (Data Processing Fee).
*   **Hậu quả:** Hóa đơn cuối tháng bùng nổ chỉ vì tiền xử lý dữ liệu của NAT Gateway, dù cả EC2 và S3 của bạn đều nằm chung một Region.
*   **Điểm ít người để ý:** Bạn chỉ cần bật **S3 Gateway VPC Endpoint**. Tính năng này hoàn toàn miễn phí. Dữ liệu sẽ chạy theo tuyến đường mạng nội bộ của AWS mà không cần đi qua NAT Gateway. Thao tác này vừa giúp tăng tốc độ truyền tải, vừa cắt giảm 100% chi phí xử lý dữ liệu phát sinh vô lý kia.

---

### 2. Glacier hóa file nhỏ: Khi giải pháp tiết kiệm trở thành thảm họa chi phí

S3 Glacier hay Glacier Deep Archive nổi tiếng là nơi lưu trữ dữ liệu lâu dài với mức giá siêu rẻ. Mình từng tự tin thiết lập một S3 Lifecycle Rule: *"Cứ dữ liệu sau 30 ngày sẽ tự động chuyển sang Glacier Deep Archive"*. Tuy nhiên, hệ thống lúc đó lại chứa hàng triệu file log nhỏ (mỗi file chỉ vài KB).

*   **Chi phí API ẩn:** AWS tính phí yêu cầu chuyển đổi (Lifecycle Transition Request) theo số lượng file ($0.03 - $0.05 cho mỗi 1.000 requests). Nếu bạn chuyển 10 triệu file nhỏ, riêng tiền phí gửi request chuyển đổi đã đắt gấp nhiều lần tổng tiền lưu trữ bạn tiết kiệm được trong cả năm.
*   **Phí định dạng metadata:** Với mỗi object đưa lên Glacier, AWS sẽ tự động cộng thêm 32KB dữ liệu quản lý (index và metadata). Một file 2KB khi lên Glacier sẽ bị tính tiền lưu trữ tương đương một file 34KB.
*   **Lời khuyên:** Trước khi tạo Lifecycle Rule sang Glacier, hãy đảm bảo bạn đã đóng gói (zip/tar) các file nhỏ thành file lớn, hoặc cài đặt điều kiện chỉ chuyển đổi các object có kích thước tối thiểu từ 128KB trở lên.

---

### 3. Lỗ hổng "leo thang quyền" từ iam:PassRole

Trong quá trình phân quyền IAM, để tiện cho công việc, nhiều người có thói quen cấp quyền `iam:PassRole` đi kèm với ký tự đại diện (`*`) cho tài khoản của Developer. Lí do là để họ có thể gắn IAM Role vào các dịch vụ như Lambda hay EC2 khi deploy.

*   **Kịch bản nguy hiểm:** Một tài khoản chỉ có quyền tạo Lambda function và quyền `iam:PassRole` đối với `*` hoàn toàn có thể tự nâng quyền của mình lên ngang hàng với Administrator.
*   **Cách thức:** Họ chỉ cần tạo một Lambda function mới, sau đó gắn cho Lambda đó một IAM Role có sẵn quyền `AdministratorAccess` trong hệ thống. Tiếp theo, viết một đoạn code ngắn trong Lambda để thực thi bất kỳ thao tác nào họ muốn dưới danh nghĩa Admin.
*   **Lời khuyên:** Quyền `iam:PassRole` nguy hiểm không kém gì quyền Admin. Luôn chỉ định rõ ràng chính xác Role nào được phép pass (Resource ARN cụ thể) thay vì dùng wildcard `*`.

---

### 4. Giới hạn 6 tiếng ngầm định của EBS Volume

AWS cung cấp tính năng Elastic Volumes cho phép bạn mở rộng dung lượng hoặc thay đổi loại ổ đĩa (từ gp2 sang gp3) trực tiếp trên một EC2 instance đang chạy mà không cần dừng hệ thống. Điều này rất tiện, nhưng nó có một "bẫy thời gian" mà ít người đọc kỹ docs.

*   **Cooldown Period:** Sau khi bạn nhấn xác nhận chỉnh sửa một EBS Volume, AWS sẽ khóa tính năng chỉnh sửa trên ổ đĩa đó trong đúng 6h.
*   **Tình huống thực tế:** Trong một đợt sự cố hệ thống bị đầy ổ đĩa, vì quá vội vã, bạn nâng tạm dung lượng từ 100GB lên 120GB để xử lý trước. Ngay sau đó bạn nhận ra 120GB vẫn không đủ và cần lên 500GB. Lúc này, AWS sẽ từ chối thao tác và bắt bạn chờ đủ 6 tiếng mới được chỉnh sửa tiếp.
*   **Lời khuyên:** Khi hạ lệnh chỉnh sửa EBS Volume, hãy tính toán con số dư dả ngay từ lần bấm đầu tiên, vì bạn sẽ không có cơ hội sửa sai trong 6h tiếp theo.

---

### Lời kết

AWS rất rộng lớn và tài liệu chính thức thường tập trung mô tả các trường hợp sử dụng lý tưởng. Nhưng trong thực tế vận hành, chính những chi tiết kỹ thuật nhỏ và các giới hạn ẩn mới là thứ quyết định độ ổn định của hệ thống cũng như độ "an toàn" cho ví tiền của bạn.

Những trải nghiệm này đều là những bài học thực tế mình phải trả giá mới rút ra được. Nếu bạn từng dính phải những cú "sập bẫy" tương tự trên AWS, hãy share bên dưới để ae trong group cùng né.

---

### Thông tin bài đăng
*   **Nhóm Facebook:** AWS Study Group
*   **Link bài đăng gốc:** [Facebook Post Link](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2229330564498570/)