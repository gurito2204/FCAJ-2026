---
title: "Bản đề xuất"
date: 2026-06-15
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Nabathico - Nắm bắt thiên cơ qua xem bói hỗn hợp
Giải pháp AWS Serverless tối giản cho ứng dụng chiêm tinh học đa văn hóa

### 1. Tóm tắt điều hành
Nabathico là một ứng dụng xem bói kết hợp hệ thống triết học phương Đông (Tử Vi) và phương Tây (Bản đồ sao Chiêm tinh) nhằm mang lại những luận giải toàn diện, đa chiều. Nền tảng hỗ trợ người dùng tra cứu nhanh gọn thông qua giao diện di động/web mượt mà. Dự án tận dụng tối đa các dịch vụ AWS Serverless cơ bản, kết hợp AI để cung cấp các bài phân tích cá nhân hóa với chi phí duy trì hệ thống cực thấp, dễ dàng mở rộng từ một dự án cá nhân lên hàng ngàn người dùng thực tế.

### 2. Tuyên bố vấn đề
**Vấn đề hiện tại**
Thị trường hiện nay phân mảnh: người dùng phải cài đặt riêng biệt các ứng dụng Tử Vi và ứng dụng Bản đồ sao. Việc tự đối chiếu, tổng hợp thông tin giữa hai hệ thống lý luận (ví dụ: mâu thuẫn giữa Cung Mệnh và Sun Sign) là vô cùng phức tạp đối với người dùng phổ thông. Các hệ thống tính toán lá số hiện tại thường rời rạc, khó bảo trì.

**Giải pháp**
Nền tảng sử dụng một kiến trúc tối giản: ứng dụng người dùng giao tiếp với Amazon API Gateway, kích hoạt AWS Lambda để chạy các thuật toán lõi bằng Python (sử dụng thư viện `lasotuvi` và `pyswisseph`). Dữ liệu người dùng được lưu trữ đơn giản trên Amazon DynamoDB. Sau khi lập xong lá số, hệ thống gọi API của LLM (như OpenAI) để tổng hợp và xuất ra lời bình giải. Amazon Cognito được dùng để quản lý đăng nhập an toàn. Các tính năng chính bao gồm: lập lá số kép tức thời, phân tích tương quan Đông - Tây bằng AI và lưu trữ hồ sơ sinh thần.

**Lợi ích và hoàn vốn đầu tư (ROI)**
Giải pháp tạo ra một sản phẩm công nghệ ngách độc đáo, có tiềm năng thu hút lượng lớn người dùng Gen Z. Kiến trúc Serverless giúp loại bỏ chi phí máy chủ cố định (pay-as-you-go). Thời gian phát triển được rút ngắn tối đa do không phải thiết kế cơ sở hạ tầng phức tạp. Ứng dụng có thể sinh lời thông qua mô hình Freemium (miễn phí lá số cơ bản, thu phí cho luận giải AI chuyên sâu).

### 3. Kiến trúc giải pháp (Tối giản)
Nền tảng áp dụng kiến trúc AWS Serverless một luồng (single-flow) để xử lý yêu cầu tính toán từ thiết bị người dùng. Yêu cầu được gửi qua API, tính toán trực tiếp trên RAM của Lambda và trả về kết quả ngay lập tức, không qua các bước trung gian phức tạp.

**Dịch vụ AWS sử dụng**
*   **Amazon API Gateway:** Tiếp nhận request (thông tin ngày giờ sinh) từ ứng dụng di động/web.
*   **AWS Lambda:** Đảm nhiệm toàn bộ logic cốt lõi (tính toán vị trí hành tinh, an sao Tử Vi, gọi AI API).
*   **Amazon DynamoDB:** Lưu trữ NoSQL siêu tốc độ cho thông tin người dùng và lịch sử lá số đã tạo.
*   **AWS Amplify:** Lưu trữ và triển khai liên tục giao diện Frontend.
*   **Amazon Cognito:** Quản lý đăng ký/đăng nhập của người dùng.

**Thiết kế thành phần**
*   **Frontend (Giao diện):** Ứng dụng (React Native hoặc Next.js) được thiết kế UI/UX trên Figma trước khi lập trình, gửi thông tin sinh thần lên hệ thống.
*   **Core Engine (Lõi tính toán):** Các script Python xử lý thư viện lịch thiên văn.
*   **AI Synthesizer:** Khối giao tiếp với LLM API để "dịch" các thông số kỹ thuật (ví dụ: góc hợp, sao chiếu mệnh) thành văn bản dễ hiểu.

### 4. Triển khai kỹ thuật
**Các giai đoạn triển khai**
Dự án được chia thành 4 giai đoạn tinh gọn:
1.  **Nghiên cứu và thiết kế:** Phác thảo UI/UX trên Figma, kiểm tra các thư viện tính toán mã nguồn mở.
2.  **Phát triển Lõi Engine (Backend):** Đóng gói thư viện `pyswisseph` và `lasotuvi` vào AWS Lambda.
3.  **Tích hợp AI & Cơ sở dữ liệu:** Xây dựng Prompt Engineering để AI đọc hiểu dữ liệu lá số; kết nối DynamoDB.
4.  **Kiểm thử và Ra mắt:** Lập trình Frontend, kết nối API, kiểm thử độ chính xác của múi giờ (Timezone) và đưa lên store.

**Yêu cầu kỹ thuật**
*   **Backend:** Logic xử lý viết bằng Python. Khi chạy test và cấu hình môi trường cục bộ trên Windows, sử dụng lệnh `python` (thay vì `python3`) để kích hoạt các script tính toán nhằm tránh lỗi terminal.
*   **Thư viện:** `pyswisseph` (Bản đồ sao) và thuật toán chuyển đổi âm dương lịch (Tử Vi).
*   **Database:** Cấu trúc JSON linh hoạt trên DynamoDB để chứa các mảng thông tin phức tạp của 12 cung hoàng đạo và 12 cung Tử Vi.

### 5. Lộ trình & Mốc triển khai
*   **Tháng 0:** Thiết kế quy trình người dùng, vẽ wireframe, thử nghiệm độ chính xác của các thư viện tính toán.
*   **Tháng 1 (Lõi hệ thống):** Thiết lập AWS (Cognito, DynamoDB, API Gateway). Viết các hàm Lambda xử lý chuyển đổi ngày giờ và tọa độ.
*   **Tháng 2 (AI & Trải nghiệm):** Thiết kế cấu trúc Prompt, tích hợp OpenAI API. Bắt đầu code giao diện người dùng.
*   **Tháng 3 (Hoàn thiện):** Kết nối Frontend với Backend, kiểm thử bảo mật dữ liệu, vá lỗi hiển thị và ra mắt phiên bản v1.0.

### 6. Ước tính ngân sách (MVP)
Chi phí được tối ưu cực thấp nhờ tận dụng Free Tier của AWS cho giai đoạn đầu.
**Chi phí hạ tầng (Ước tính hàng tháng cho ~1.000 user active):**
*   **AWS Lambda:** 0,00 USD (nằm trong giới hạn 1 triệu request miễn phí).
*   **Amazon DynamoDB:** 0,00 USD (nằm trong giới hạn 25 GB lưu trữ miễn phí).
*   **Amazon API Gateway:** ~0,01 USD.
*   **AWS Amplify:** ~0,35 USD (lưu trữ và băng thông frontend).
*   **LLM API (OpenAI/Claude):** ~5,00 USD - 10,00 USD (trả theo lượng token thực tế tạo ra từ việc luận giải lá số).
*   **Tổng chi phí AWS/Cloud:** < 1 USD/tháng. Tổng phí vận hành (bao gồm AI): ~6-11 USD/tháng.

### 7. Đánh giá rủi ro
**Ma trận rủi ro**
*   **Sai số Timezone (Múi giờ):** Ảnh hưởng cao, xác suất trung bình (giờ sinh là yếu tố sống còn của lá số).
*   **Độ trễ API của AI (Timeout):** Ảnh hưởng trung bình, xác suất cao (LLM đôi khi mất 10-15s để phản hồi).
*   **Vượt chi phí token AI:** Ảnh hưởng trung bình, xác suất thấp (nếu bị lạm dụng spam request).

**Chiến lược giảm thiểu**
*   **Múi giờ:** Sử dụng thư viện cơ sở dữ liệu múi giờ chuẩn quốc tế (TZ database) kết hợp Geo-coding từ Google Maps để lấy tọa độ và múi giờ chính xác tuyệt đối.
*   **Độ trễ:** Thiết kế UI hiển thị hiệu ứng "Đang chiêm tinh..." hoặc "Đang kết nối các vì sao..." để giữ chân người dùng trong lúc đợi Lambda gọi AI.
*   **Chi phí:** Đặt quota (hạn mức) request trên API Gateway và thiết lập cảnh báo ngân sách (Billing Alarm).

### 8. Kết quả kỳ vọng
*   **Cải tiến kỹ thuật:** Tự động hóa hoàn toàn việc an sao, lập bản đồ và tổng hợp thông tin, thay thế việc tra cứu chéo thủ công giữa các sách hay nền tảng riêng lẻ.
*   **Giá trị dài hạn:** Xây dựng được một Engine cốt lõi về huyền học Đông - Tây, có thể đóng gói bán dưới dạng API SaaS (Software as a Service) cho các bên thứ ba, hoặc mở rộng thêm Tarot, Bát Tự trong tương lai.