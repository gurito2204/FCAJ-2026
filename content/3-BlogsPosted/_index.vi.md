---
title: "Blog 3"
date: 2026-06-30
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# NHỮNG KỸ THUẬT "NGẦM" TRÊN AWS: TỪ TIỀN PHẠT ẨN ĐẾN NHỮNG SỰ CỐ MẠNG VÔ HÌNH

Khi làm việc với AWS đủ lâu, chúng ta sẽ bắt đầu nhận ra rằng có một khoảng cách rất lớn giữa những gì được giảng dạy trong Certificate và những gì thực sự diễn ra ở môi trường Production. Có những sự cố không phát ra cảnh báo rõ ràng, không có trong các lỗi phổ biến trên StackOverflow, nhưng lại khiến hệ thống chập chờn hoặc ngốn hàng ngàn USD tiền phí dịch vụ.

Dưới đây là 4 vấn đề kỹ thuật chuyên sâu và ít được chú ý trên AWS mà mình từng mất ăn mất ngủ mới có thể tự rút ra bài học.

---

### 1. Cross-AZ Data Transfer: "Cú sốc" chi phí mạng ngay trong cùng một Region

Hầu hết mọi người đều biết data transfer out Internet sẽ tốn tiền. Nhưng rất nhiều ae lại lầm tưởng rằng: *"Chỉ cần dữ liệu chạy nội bộ bên trong cùng một Region thì sẽ hoàn toàn free"*.

*   **Sự thật ngầm định:** AWS tính phí truyền dữ liệu giữa các Availability Zone (AZ) khác nhau trong cùng một Region. Mức phí thông thường là $0.01 cho mỗi GB ở chiều gửi và $0.01 cho mỗi GB ở chiều nhận (tổng cộng là $0.02/GB cho một chu trình dữ liệu).
*   **Kịch bản thực tế:** Bạn triển khai một EKS cluster hoặc dàn EC2 chạy microservices phân bổ trên cả `AZ-a` và `AZ-b` để đảm bảo tính High Availability (HA). Các service này liên tục gọi API lẫn nhau hoặc truy vấn chung một con Redis Cache / DB nằm cố định ở `AZ-a`.
*   **Hậu quả:** Hàng nghìn lượt gọi inter-service cross-AZ mỗi giây sẽ âm thầm tạo ra hàng Terabyte dữ liệu đi qua lại giữa các AZ. Kết quả là hóa đơn "Data Transfer" cuối tháng cao vượt cả tiền thuê server.
*   **Giải pháp:** Áp dụng tư duy **AZ-Awareness**. Sử dụng các cơ chế điều hướng như *Topology Aware Hints* trong Kubernetes để ưu tiên định tuyến traffic giữa các service nằm trong cùng một AZ. Chỉ cho traffic vượt AZ khi thật sự xảy ra sự cố.

---

### 2. Bẫy MTU 9001 và sự cố kết nối chập chờn qua VPC Peering / VPN

Đây là một trong những ca debug kinh điển khiến nhiều DevOps mới vào nghề mất nhiều ngày bế tắc vì hệ thống không hề quăng ra log lỗi cụ thể.

*   **Bản chất vấn đề:** Mặc định, các máy ảo EC2 trong cùng một VPC truyền tải dữ liệu với kích thước khung mạng (MTU) là 9001 bytes (Jumbo Frames). Tuy nhiên, khi kết nối đi qua VPC Peering giữa các Region, kết nối VPN, hoặc Direct Connect, giới hạn MTU sẽ bị kéo tụt xuống chuẩn thông thường là 1500 bytes.
*   **Hiện tượng lạ:** Lệnh ping giữa 2 máy ảo ở 2 VPC vẫn phản hồi mượt mà. Kết nối SSH vẫn bình thường. Nhưng khi ứng dụng truyền một file lớn hoặc gửi một chuỗi JSON API dài qua kết nối đó, request sẽ bị treo vô tận cho đến khi timeout.
*   **Nguyên nhân ngầm:** Do gói tin quá lớn (> 1500 bytes) cần phải phân mảnh (fragmentation), nhưng router trung gian không thể thực hiện và phải gửi lại một thông điệp ICMP thông báo. Nếu đội ngũ Sysadmin vô tình cài đặt Security Group chặn tiệt toàn bộ giao thức ICMP, gói tin thông báo này sẽ bị nuốt chửng (sự cố **Path MTU Discovery Black Hole**).
*   **Giải pháp:** Luôn cho phép giao thức ICMP (cụ thể là *Custom ICMP - IPv4: Type 3, Code 4 - Destination Unreachable*) trong Security Group, hoặc hạ MTU trên card mạng của EC2 xuống 1500 nếu hệ thống phụ thuộc nhiều vào các đường truyền liên VPC/VPN.

---

### 3. "Trần giới hạn" ngầm của DynamoDB On-Demand khi gặp Flash Sale

Mô hình DynamoDB On-Demand được quảng cáo là giải pháp tự động co giãn tuyệt vời, không cần tính toán trước năng lượng xử lý (WCU/RCU). Nhưng từ "tự động" này có một điều khoản ràng buộc rất quan trọng.

*   **Cơ chế hoạt động thực tế:** DynamoDB On-Demand không hề mở rộng đến mức vô tận ngay lập tức. Nó chỉ đáp ứng ngay lập tức được gấp đôi mức traffic đỉnh cao nhất mà bạn từng đạt được trong quá khứ.
*   **Kịch bản thảm họa:** Hệ thống của bạn bình thường chạy rất êm đềm ở mức 1,000 requests/giây. Đúng 12h đêm diễn ra sự kiện Flash Sale, lượng truy cập tăng vọt lên 20,000 requests/giây chỉ trong vòng vài giây.
*   **Hậu quả:** DynamoDB lập tức trả về hàng loạt lỗi `ProvisionedThroughputExceededException` và đánh rớt request của khách hàng. Cơ chế tự động chia tách partition ngầm của DynamoDB cần khoảng thời gian vài phút để đáp ứng mức load mới này, nhưng lúc đó khách hàng của bạn đã bỏ đi rồi.
*   **Giải pháp:** Nếu biết trước có sự kiện tăng bùng nổ traffic trong thời gian ngắn, hãy chủ động chuyển bàn cờ sang chế độ **Provisioned Capacity** trước vài tiếng, đặt con số WCU/RCU cao tương ứng để DynamoDB chuẩn bị sẵn tài nguyên hạ tầng, sau đó mới bật lại chế độ On-Demand nếu muốn.

---

### 4. Cú click chuột đắt giá trên CloudWatch Logs Insights

CloudWatch Logs Insights là một công cụ truy vấn log bằng cú pháp dạng SQL rất tiện lợi. Nhưng cách AWS tính phí công cụ này lại khiến chúng ta bất ngờ.

*   **Cách AWS tính tiền:** Phí của CloudWatch Logs Insights không tính trên số lượng dòng log trả về, mà tính trên tổng dung lượng log thô bị quét, với mức giá khoảng $0.005 cho mỗi GB quét được.
*   **Sai lầm phổ biến:** Bạn mở công cụ này lên, chọn một Log Group tích tụ log trong 6 tháng qua (dung lượng khoảng 500GB), gõ một câu lệnh tìm kiếm chung chung không chọn khoảng thời gian cụ thể và nhấn Run.
*   **Hậu quả:** Chỉ bằng một cú click chuột đơn giản để chờ vài giây lấy kết quả, bạn vừa tốn $2.5. Nếu một dev thiết lập cronjob chạy tự động câu truy vấn đó mỗi 5 phút để làm Dashboard, bạn sẽ nhận về một hóa đơn hàng ngàn USD chỉ riêng cho tiền scan log.
*   **Giải pháp:** Luôn giới hạn khoảng thời gian truy vấn nhỏ nhất có thể (ví dụ: chỉ quét trong 15 phút hay 1 giờ gần nhất). Đối với nhu cầu phân tích log lịch sử dung lượng lớn, hãy xuất log ra S3 và dùng **AWS Athena** để truy vấn với chi phí rẻ hơn nhiều.

---

### Lời kết

Làm việc trên Cloud không chỉ là việc lắp ghép các dịch vụ theo sơ đồ kiến trúc, mà là quá trình hiểu rõ các thông số vận hành bên dưới tầng hạ tầng. Những chi tiết như kích thước gói tin MTU, cơ chế phân tách partition hay cước phí cross-AZ thường bị bỏ qua vì chúng không xuất hiện ở các bài lab cơ bản.

Hi vọng những chia sẻ thực tế này sẽ giúp mọi người tránh được những sự cố "vô hình" và tối ưu hệ thống tốt hơn. Nếu ae từng gặp phải những trường hợp oái ăm nào khác trên AWS, hãy để lại cmt để ae cùng thảo luận.

---

### Thông tin bài đăng
*   **Nhóm Facebook:** AWS Study Group
*   **Link bài đăng gốc:** [Facebook Post Link](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2230167921081501/?rdid=OWE359AjcB0vTUf2#)