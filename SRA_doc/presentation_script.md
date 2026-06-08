# 📝 Kịch bản Báo cáo SRS - Ephurin (Cân bằng & Chuyên nghiệp)

## 1. Mở đầu & Tổng quan (Thái - Nhóm trưởng)

**Thái:** 
"Dạ em chào Thầy. Em là Thái, đại diện nhóm xin trình bày tóm tắt về tài liệu Đặc tả Yêu cầu (SRS) cho dự án Ephurin. 

Thay vì chỉ làm một website đọc truyện thông thường, điểm nhấn của dự án là việc nhóm em tích hợp trợ lý AI để hỗ trợ trực tiếp cho tác giả. Để xây dựng bộ tài liệu này một cách chuẩn xác, nhóm em đã áp dụng kết hợp nhiều phương pháp và kỹ thuật cốt lõi trong môn SRA:
*   **Requirements Elicitation & Benchmarking (Thu thập yêu cầu & Đối chuẩn):** Khảo sát các nền tảng truyện hiện có (như Wattpad, Webnovel) để xác định điểm khác biệt và phân tích lợi thế cạnh tranh.
*   **Functional Decomposition (Phân rã chức năng):** Xây dựng sơ đồ cây (Functional Tree) để khoanh vùng và làm rõ phạm vi chức năng của toàn hệ thống.
*   **UML & Data Modeling (Mô hình hóa):** Sử dụng Use Case, Activity Diagram để chuẩn hóa các luồng nghiệp vụ phức tạp và dùng ERD để thiết kế cấu trúc dữ liệu.
*   **Prototyping / Wireframing (Thiết kế bản mẫu):** Xây dựng các UI Mockups cho cả giao diện Web và Mobile để trực quan hóa yêu cầu của người dùng trước khi code.
*   **CRUD Analysis (Ma trận phân quyền):** Xây dựng Permission Matrix để bóc tách chi tiết quyền hạn (Create, Read, Update, Delete) cho từng nhóm đối tượng (Actor).
*   **NFR Analysis (Phân tích yêu cầu phi chức năng):** Đặt ra các ràng buộc cụ thể về Hiệu năng (Performance), Bảo mật (Security) và Độ tin cậy (Reliability) cho hệ thống Microservices.

Về **công cụ thực hiện tài liệu**:
*   Nhóm thống nhất sử dụng định dạng **Markdown** làm chuẩn để dễ dàng quản lý phiên bản (Version Control) trên Git.
*   Các biểu đồ luồng được nhóm sử dụng **Mermaid.js** để nhúng trực tiếp vào file, kết hợp với **Draw.io** cho các sơ đồ có độ phức tạp cao.
*   Ngoài ra, nhóm cũng tận dụng tối đa **Google Search, ChatGPT và Gemini** để tra cứu nhanh các tiêu chuẩn viết Use Case và thu thập dữ liệu phục vụ phần Benchmarking.

Dạ tiếp theo, để tối ưu thời gian, mỗi thành viên sẽ chia sẻ nhanh về phần đặc tả thử thách nhất hoặc thú vị nhất mà mình phụ trách ạ."

---

## 2. Phần của Từng Thành Viên

### 2.1. Thái: Hệ thống, ERD, Module AI & Đối chuẩn (Benchmarking)
**Thái:** 
"Về phần em, em phụ trách kiến trúc tổng thể, module AI và phân tích Đối chuẩn (Benchmarking). Ở phần Benchmarking, em đã tập trung khảo sát 2 nhóm nền tảng lớn: **nền tảng đăng truyện truyền thống** (như Wattpad, Webnovel) và **nền tảng release sản phẩm nội dung lớn** (như Netflix, Spotify) để chốt lại lợi thế cạnh tranh cho Ephurin.

**Điểm khó khăn lớn nhất** của em trong quá trình làm tài liệu là việc **đặc tả yêu cầu cho tính năng AI**. Nó không giống các chức năng thêm/sửa/xóa thông thường, đòi hỏi phải định nghĩa rất rõ đầu vào và đầu ra. Khó khăn thứ hai là phải **thiết kế ERD kết hợp song song** cả PostgreSQL (dữ liệu có cấu trúc) và MongoDB (dữ liệu phi cấu trúc), khiến mô hình dữ liệu trở nên phức tạp. Để giải quyết, em đã thiết kế luồng 'Kiểm duyệt lai' (Hybrid Moderation) nhằm phân định rõ ràng các tác vụ."

### 2.2. Thắng: Trải nghiệm Độc giả, Tương tác & App Mobile
**Thắng:**
"Dạ phần của em là phân tích toàn bộ **Module Trải nghiệm Độc giả và Tương tác Cộng đồng** trên nền tảng Mobile App. Để đặc tả nghiệp vụ phần này, em đã tiến hành qua 3 bước chính:
*   **Bước 1:** Vẽ Use Case tổng quát phía Độc giả, xác định rõ hai luồng nghiệp vụ cốt lõi là: *Đọc truyện cá nhân hóa* (tùy chỉnh phông chữ, chế độ Dark/Light mode) và *Tương tác xã hội* (Thích, Lưu trữ, Theo dõi tác giả).
*   **Bước 2:** Dùng Activity Diagram để mô hình hóa các luồng nghiệp vụ phức tạp, ví dụ như luồng Tìm kiếm lọc truyện nâng cao có kết nối với Backend, hay luồng nhận Push Notification.
*   **Bước 3:** Thiết kế hệ thống UI Mockups cho Mobile App (từ màn hình Khám phá đến màn hình Đọc) để trực quan hóa các tính năng ngay trên giao diện di động.

Trong toàn bộ quá trình đó, **khó khăn lớn nhất** đối với em là **đặc tả kỹ thuật cho tính năng Bình luận theo đoạn văn (Inline Commenting)**. Để mô tả rành mạch kịch bản: *người dùng bôi đen đoạn chữ -> app phải tính toán và lưu lại mốc tọa độ (anchor) -> hiển thị bình luận khớp đúng với ngữ cảnh câu văn ngay cả khi độc giả phóng to/thu nhỏ phông chữ* là một bài toán rất hóc búa. Em đã phải vẽ các luồng rẽ nhánh (Exception Flow) cực kỳ chi tiết để xử lý lỗi đồng bộ giao diện hoặc khi mất kết nối mạng."

### 2.3. Trung: Luồng Sáng tác & Quản lý Tác phẩm
**Trung:**
"Dạ em phụ trách luồng Đăng truyện cho Tác giả. **Điểm khó khăn nhất** của em là phải **mô hình hóa một luồng nghiệp vụ xuất bản có sự can thiệp liên tục của AI**. Em phải suy nghĩ cách diễn đạt luồng đồng bộ: *Tác giả viết -> AI tự động quét lỗi -> Hệ thống lưu nháp -> Đủ 5 chương mới cho phép gửi duyệt -> Admin kiểm tra*. Việc phải liệt kê không thiếu sót bất kỳ một **điều kiện ràng buộc** hay **luồng thay thế** nào trong Activity Diagram là một thử thách lớn, nhưng nó giúp team dev sau này không bị bối rối khi code."

### 2.4. Tùng: Phân quyền & Admin
**Tùng:**
"Em đảm nhận phần Account, Phân quyền và Admin. **Khó khăn lớn nhất** của em là **xử lý độ phức tạp của việc phân quyền và bảo mật**. Em đã tốn rất nhiều thời gian để rà soát và xây dựng **Bảng Phân Quyền (Permission Matrix)** sao cho không bị xung đột giữa 5 nhóm đối tượng (Guest, Reader, Author, Admin, AI). Bên cạnh đó, việc phải mô tả quy trình cấp phát Token (Access/Refresh Token) qua OAuth 2.0 bằng sơ đồ luồng cũng rất khó khăn vì đòi hỏi sự chính xác tuyệt đối về mặt kỹ thuật."

---

## 3. Tổng kết (Thái)
**Thái:**
"Dạ tóm lại, việc hoàn thiện bộ tài liệu SRS này giúp nhóm em biết cách ứng dụng các công cụ như Markdown, Mermaid để diễn đạt ý tưởng hệ thống và luồng AI một cách chuyên nghiệp và rõ ràng hơn.
Dạ phần trình bày tóm tắt của nhóm đến đây là hết. Nhóm em cảm ơn Thầy đã lắng nghe và rất mong nhận được góp ý từ Thầy ạ!"
