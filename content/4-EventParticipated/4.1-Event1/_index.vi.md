---
title: "Event 1"
date: 2026-07-26
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---


# Bài thu hoạch “FCAJ x Agentic AI Build Week”

### Mục Đích Của Sự Kiện

-Trải nghiệm và thực hành AI: Tạo sân chơi để sinh viên làm quen, học hỏi và ứng dụng công nghệ Agentic AI trong thực tế.
Giải quyết bài toán thực tiễn.
-Rèn luyện kỹ năng xây dựng hệ thống: Hướng dẫn sinh viên cách thiết kế kiến trúc hệ thống trên AWS, từ giai đoạn lên ý tưởng đến triển khai sản phẩm.
-Phát triển kỹ năng mềm: Rèn luyện khả năng làm việc nhóm, quản lý thời gian, xử lý áp lực và kỹ năng pitching sản phẩm trong thời gian ngắn.
-Bài học về tư duy giải quyết vấn đề: Cách từ một ý tưởng sơ khai, họ phát triển thành một dự án giải quyết được pain thực tế cho doanh nghiệp
### Dư Án Khiến Mình Chú Ý
**Signal Scout**
### Danh Sách Diễn Giả

- **Lê Tấn Luc**
- **Đỗ Hoàng Hiếu**
- **Triệu Quốc Hảo**
- **Nguyễn Văn Duy Khiêm**
- **Nguyễn Công Minh**
- **Nguyễn Trần Minh Quân**


#### Overview - Ton quan du an

* **Bai toan & Noi dau - Pain Point:**
    * Các doanh nghiệp, tập đoàn lớn thường gặp khó khăn khi muốn theo dõi và phát hiện sớm các thay đổi chiến lược, tái cấu trúc hoặc hướng đi mới của đối thủ cạnh tranh.
    * Những thông tin hoặc tín hiệu - signals này nằm vô cùng rải rác trên báo cáo tài chính, tài liệu họp cổ đông, thông cáo báo chí, hay các trang web.

* **Giải pháp:**
    * Xây dựng một Multi-Agent System thu thập, xâu chuỗi và phân tích các tín hiệu rời rạc đó thành một câu chuyện bức tranh toàn cảnh trực quan.
    * Đưa ra đánh giá, dự báo chỉ số ROI, rủi ro và khuyến nghị chiến lược Maintain, Adapt, Accelerate giúp các đội ngũ chiến lược và quản trị rủi ro ra quyết định.
    * Cung cấp một giao diện Self-service Executive Dashboard thân thiện, trực quan và minh bạch.

#### Architecture & Implementation

* **Luồng xử lý chính của hệ thống Multi-Agent:**
    * **Frontend & Security:** Dashboard giao diện React UI được lưu trữ trên AWS Amplify Hosting, bảo vệ bởi AWS WAF và xác thực người dùng qua Amazon Cognito.
    * **Supervisor Agent:**
        * Sử dụng AgentCore Runtime / AgentCore Management để điều phối các Agent con theo mô hình Agent-to-Agent - A2A.
        * Tích hợp AgentCore Short-Term Memory để lưu giữ thông tin ngữ cảnh - session context xuyên suốt quá trình hội thoại.
    * **Sub-agents & Công cụ:**
        * **Crawler Subagent:** Gọi các công cụ cào dữ liệu web.
            * Sử dụng Apify cho các trang web tĩnh cần thu thập lượng dữ liệu lớn.
            * Sử dụng TinyFish cho các trang web động có độ sâu cao hoặc có các rào cản truy cập - login wall.
        * **Data Sanitization:** Lọc và làm sạch dữ liệu bằng code thuần Lambda ngay sau khi cào về nhằm giảm chi phí token và ngăn chặn rủi ro Prompt Injection.
        * **Analysis Subagent & Quản trị:** Sử dụng Amazon Bedrock Guardrails để kiểm soát dữ liệu đầu vào và đầu ra. Dữ liệu được gửi tới Langfuse để phân tích và đánh giá chất lượng - evaluation scoring.
    * **Evaluation & Data Persistence:**
        * Nếu điểm đánh giá cao, dữ liệu được lưu vào Amazon S3 và metadata lưu vào Amazon DynamoDB, sau đó hiển thị lên Dashboard.
        * Nếu điểm đánh giá thấp, hệ thống kích hoạt cơ chế retry - tối đa 2 lần để tiết kiệm chi phí. Nếu vẫn thấp sau các lần chạy lại, hệ thống gắn tag yêu cầu con người can thiệp review - Human-in-the-loop.

* **Cost-efficient Architecture:**
    * Nhóm đã tự phân tích chi phí và nhận thấy việc phụ thuộc vào các công cụ bên thứ 3 như Apify, TinyFish, Langfuse khiến chi phí tăng đột biến từ khoảng $35/tháng lên tới khoảng $359/tháng ở mức dùng tối đa.
    * Nhóm đề xuất phương án cải tiến Native AWS bằng cách thay thế công cụ cào web bên thứ 3 bằng AWS Built-in Web/Browser tools, giúp vừa tối ưu chi phí vừa đảm bảo tính tuân thủ bảo mật dữ liệu - Data Residency & Compliance cho doanh nghiệp.

#### Team's Lessons Learned

* **Clear Direction:** Ban đầu nhóm có quá nhiều ý tưởng dẫn đến tranh luận gay gắt. Bài học là cần hạ cái tôi xuống, lắng nghe nhau để chốt một giải pháp khả thi nhất và cùng đi đến cùng.
* **Execution:** Ý tưởng hay đến đâu nếu chỉ nằm trên slide thì không có giá trị trong Hackathon. Phải ưu tiên làm ra sản phẩm hoặc demo chạy được thực tế để chứng minh giải quyết được bài toán - Pain point của người dùng.
* **Teamwork:** Sự tin tưởng, phân chia công việc đúng thế mạnh và tinh thần gạt bỏ xung đột cá nhân là chìa khóa để đưa cả đội vượt qua áp lực làm việc xuyên đêm.

#### Key Takeaways

* **Business-driven Tech:** Kỹ thuật hoặc kiến trúc phức tạp hay xịn đến đâu cũng phải phục vụ một bài toán cụ thể và mang lại giá trị thực tế cho doanh nghiệp.
* **FinOps trong AI:** Khi thiết kế kiến trúc Agentic AI, việc tính toán chi phí token, chọn dịch vụ Native AWS hay Third-party, và áp dụng cơ chế giới hạn số lần retry là vô cùng quan trọng để tránh bùng nổ chi phí.
* **Kỹ năng thi Hackathon & Pitching:**
    * Không cần hệ thống phải hoàn thiện 100%, nhưng demo phải phản ánh đúng điểm đau - pain point và quy trình MVP.
    * Chuẩn bị kỹ cho phần Q&A vì sự quan tâm và câu hỏi đào sâu từ Ban giám khảo chứng tỏ ý tưởng đã gây ấn tượng thành công.
![Chụp cùng anh Nguyễn Công Minh](/images/event.jpg)