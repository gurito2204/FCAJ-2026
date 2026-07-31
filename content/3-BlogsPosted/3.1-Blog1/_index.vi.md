---
title: "Blog 1"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# 3 CHI TIẾT "NGÁCH" TRÊN AWS ÍT AI NÓI CHO BẠN BIẾT, NHƯNG ĐỤNG VÀO LÀ DÍNH SỰ CỐ

Thay vì bàn về những khái niệm vĩ mô, bài này mình tổng hợp 3 chi tiết kỹ thuật rất nhỏ trên AWS. Chúng ít khi xuất hiện trên các slide bài giảng nhưng lại là nguyên nhân gây mất tiền vô lý hoặc tốn vài ngày debug.

---

### 1. File "tàng hình" trên S3: Tiền vẫn mất nhưng file không thấy (Incomplete Multipart Uploads)

Khi ứng dụng upload một file dung lượng lớn lên S3 và bị ngắt mạng giữa chừng, tiến trình bị hủy.

*   **Sự thật ngầm:** Phần dữ liệu đã truyền lên trước khi đứt mạng vẫn nằm lại trên hạ tầng của S3, và AWS vẫn âm thầm tính tiền lưu trữ cho phần dữ liệu dở dang đó hàng tháng.
*   **Điểm oái ăm:** Bạn không thể nhìn thấy các mảnh dữ liệu này trên giao diện S3 Console hay qua câu lệnh `aws s3 ls` thông thường. Nếu upload hàng ngàn file lớn bị lỗi, dung lượng tiền phạt ẩn này là không nhỏ.
*   **Xử lý:** Luôn bật tính năng **S3 Lifecycle Rule** và tích chọn *Delete incomplete multipart uploads* sau 1 đến 2 ngày để tự động dọn sạch rác ẩn.

---

### 2. Bẫy IMDSv2 Hop Limit khi chạy ứng dụng trong Container

Nâng cấp từ IMDSv1 lên IMDSv2 là chuẩn bảo mật bắt buộc để chống rò rỉ IAM Role credentials trên EC2. Nhưng nếu ứng dụng của bạn chạy bên trong Docker container đặt trên EC2 đó, ứng dụng sẽ lập tức bị mất quyền truy cập AWS SDK.

*   **Nguyên nhân ngầm:** IMDSv2 sử dụng chỉ số TTL (Time to Live) của gói tin IP để chặn hành vi Session Hijacking. Mặc định, AWS đặt tham số `PutResponseHopLimit = 1`.
*   **Sự cố:** Request đi từ ứng dụng bên trong Container, đi qua card mạng ảo của Docker (`veth bridge`) để tới IP metadata (`169.254.169.254`) đã nhảy qua 2 lớp mạng (2 hops). Gói tin lập tức bị hủy vì vượt quá giới hạn Hop Limit = 1.
*   **Xử lý:** Tăng thông số **Metadata Response Hop Limit** của EC2 instance đó từ 1 lên 2.

---

### 3. Thư mục /tmp của AWS Lambda không sạch như bạn nghĩ

Nhiều lập trình viên mặc định rằng mỗi lần Lambda function được kích hoạt, nó sẽ chạy trong một môi trường hoàn toàn isolated và sạch sẽ.

*   **Sự thật ngầm:** Nhờ cơ chế **Warm Start**, AWS sẽ tái sử dụng lại container cho các lần gọi tiếp theo để tối ưu tốc độ. Điều này đồng nghĩa với việc các file bạn ghi vào thư mục `/tmp` ở request trước vẫn nằm nguyên ở đó cho request sau.
*   **Hậu quả:**
    *   *Rủi ro bảo mật:* Nếu ghi file tạm chứa thông tin cá nhân của User A mà quên xóa, User B ở request tiếp theo (dùng chung container warm) có thể đọc được file đó.
    *   *Rủi ro tràn ổ đĩa:* Thư mục `/tmp` sẽ bị đầy dần qua các lượt gọi, gây ra lỗi *No space left on device* một cách ngẫu nhiên và khó bắt vết.
*   **Xử lý:** Luôn bọc đoạn code ghi file tạm trong khối `try...finally` để đảm bảo file luôn được xóa chủ động ngay sau khi xử lý xong, không phụ thuộc vào vòng đời của Lambda.

---

### Thông tin bài đăng
*   **Nhóm Facebook:** AWS Study Group
*   **Link bài đăng gốc:** [Facebook Post Link](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2229330564498570/)