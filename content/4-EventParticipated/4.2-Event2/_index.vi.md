---
title: "FCAJ Community Day - June 2026"
date: 2026-06-30
weight: 2
chapter: false
pre: " "
---

Bài thu hoạch “FCAJ Community Day”

#### Mục Đích Của Sự Kiện

- **Kết nối cộng đồng:** Tạo không gian giao lưu, học hỏi và chia sẻ kinh nghiệm thực chiến giữa các thành viên, chuyên gia và những người đam mê công nghệ/AI.
- **Cập nhật xu hướng & Công nghệ mới:** Điểm qua các keyword, kiến thức và mô hình Voice AI để người tham dự có thể nghiên cứu (research), làm dự án, tham gia các cuộc thi hoặc chuẩn bị sẵn sàng cho công việc thực tế.
- **Giải quyết bài toán thực tế:** Lắng nghe các use-case cụ thể từ doanh nghiệp và cách áp dụng công nghệ để giải quyết các "nỗi đau" (pain points) thực tế tại các ngân hàng lớn (VPBank, VIB).
- **Định hướng phát triển nghề nghiệp:** Chia sẻ cơ hội và thách thức trong việc triển khai các mô hình AI giọng nói tại thị trường Việt Nam.

#### Dự Án / Chủ Đề Khiến Mình Chú Ý

Voice AI - Xây dựng cơ chế giọng nói cho AI

#### Danh Sách Diễn Giả

- Anh Hiếu Nghị (Renova Cloud)
- Anh Kiệt (AWS Study Group)
- Anh Trung (CEO của Revve AI / Revve AI)

#### Overview - Tổng quan dự án/chủ đề

#### Bài toán & Nỗi đau - Pain Point:

- Dữ liệu giọng nói tiếng Việt chất lượng cao cho AI hiện tại rất khan hiếm.
- Mô hình Speech-to-Speech (End-to-End) hoạt động chưa tốt với tiếng Việt, rất khó kiểm soát (guardrail) nội dung AI nói real-time và khả năng gọi hàm thực thi tác vụ (Tool Calling) còn rất hạn chế.
- Khó khăn khi triển khai thực tế trong môi trường ngân hàng (BFSI) với các yêu cầu khắt khe về độ trễ, sự chính xác và khả năng thực thi nghiệp vụ tự động (như khóa thẻ, xác minh CCCD).

#### Giải pháp:

- Áp dụng kiến trúc Pipeline 3 thành phần: `STT (Speech-to-Text) -> LLM -> TTS (Text-to-Speech)`.
- Dùng STT chuyển âm thanh sang text, nạp vào LLM với prompt/agent context cụ thể và thực hiện Tool Calling (khóa thẻ, xử lý yêu cầu), sau đó dùng TTS sinh giọng phản hồi.
- Tận dụng ưu điểm xử lý tiếng Việt rất tốt của các LLM hiện nay để vừa làm chủ được nội dung output, vừa đảm bảo tính chính xác và an toàn thông tin.

#### Architecture & Implementation

#### Luồng xử lý chính của hệ thống:

- **Tầng Giao diện / Frontend:** Tích hợp trực tiếp vào hệ thống thoại hoặc ứng dụng ngân hàng/khách hàng.
- **Tầng Logic / Backend & AI:** Sử dụng chuỗi 3 model (STT - LLM - TTS) kết hợp huấn luyện thêm các model phụ trợ chuyên biệt cho tiếng Việt:
  - **Detect giới tính:** Nhận biết nam/nữ ngay từ giọng nói để xưng hô Anh/Chị chuẩn xác, tránh gây phản cảm.
  - **Xử lý ngắt lời & Ngữ cảnh (Barge-in & Turn-taking):** Tránh việc AI cướp lời khi khách chỉ đang dừng ngắt giọng tư duy (như đọc số điện thoại dở chừng), đồng thời tự động dừng phát giọng AI khi người dùng nói đè lên.
- **Quản trị & Bảo mật:** Kiểm soát nội dung câu trả lời ở dạng text trước khi đưa sang TTS và có khả năng chuyển đổi sang nhân viên hỗ trợ (Human-in-the-loop) khi AI gặp sự cố.

#### Cost-efficient Architecture (Tối ưu hóa):

- Ưu tiên cơ chế Streaming real-time ở cả 3 tầng: STT stream text ngay khi nghe -> LLM stream output text -> TTS stream ra âm thanh tức thì (không chờ sinh xong toàn bộ câu) để tối ưu thời gian phản hồi và trải nghiệm thoại.

#### Team's / Speaker's Lessons Learned

- **Context-awareness (Thấu hiểu ngữ cảnh & Ngôn ngữ):** Phải hiểu rõ đối tượng và đặc thù giao tiếp tiếng Việt (nhận diện giới tính để xưng hô Anh/Chị, xử lý khoảng lặng hợp lý để không nhảy vào miệng khách hàng).
- **Xử lý giọng vùng miền & Persona:** Đưa 10% – 20% dữ liệu giọng vùng miền vào tập train cho STT/TTS để AI nghe hiểu tốt các giọng địa phương, nhưng duy trì persona giọng chuẩn chuyên nghiệp chứ không tự động đổi nhại theo giọng khách hàng.
- **Human-in-the-loop:** AI đóng vai trò hỗ trợ. Hệ thống phải được thiết kế để khi gặp tình huống khó hoặc người dùng phản ứng, tổng đài viên con người có thể nhảy vào (jump-in) cứu nguy kịp thời.

#### Key Takeaways

- **Business-driven Tech & Tool Calling:** Công nghệ Voice AI chỉ thực sự hiệu quả khi gắn liền với các kịch bản kinh doanh và thực thi được tác vụ thực tế (khóa thẻ, hỗ trợ tài khoản).
- **Tối ưu hóa nguồn lực:** Kiểm soát độ trễ (latency) bằng Streaming và xử lý ngắt lời tự nhiên là chìa khóa để triển khai thành công AI trong các hệ thống ngân hàng/doanh nghiệp lớn.
- **Giá trị của sự thấu hiểu ngôn ngữ bản địa:** Việc nghiên cứu sâu về ngữ cảnh tiếng Việt (danh xưng Anh/Chị, xử lý ngắt giọng, giọng vùng miền) là lợi thế cạnh tranh cốt lõi cho các giải pháp Voice AI tại Việt Nam.
![Photo at Live FCAJ Community Day - June 2026](/images/event1.png)