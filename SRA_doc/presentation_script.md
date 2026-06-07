# 📝 Kịch bản Báo cáo SRS - Ephurin (Cân bằng & Chuyên nghiệp)

## 1. Mở đầu & Tổng quan (Thái - Nhóm trưởng)

**Thái:** 
"Dạ em chào Thầy. Em là Thái, đại diện nhóm xin trình bày tóm tắt về tài liệu Đặc tả Yêu cầu (SRS) cho dự án Ephurin. 

Thay vì chỉ làm một website đọc truyện thông thường, điểm nhấn của dự án là việc nhóm em tích hợp trợ lý AI để hỗ trợ trực tiếp cho tác giả. Để xây dựng bộ tài liệu này một cách chuẩn xác, nhóm em đã áp dụng một số phương pháp chính:
*   **Benchmarking (Đối chuẩn):** Khảo sát các nền tảng truyện hiện có để xác định rõ điểm khác biệt và lợi thế cạnh tranh của Ephurin.
*   **Phân rã chức năng & Mô hình hóa:** Nhóm chia nhỏ tính năng bằng sơ đồ cây, sau đó sử dụng Use Case, Activity Diagram và ERD để làm rõ luồng hoạt động của hệ thống.

Về **công cụ thực hiện tài liệu**:
*   Nhóm thống nhất sử dụng định dạng **Markdown** làm chuẩn để dễ dàng quản lý phiên bản (Version Control) trên Git.
*   Các biểu đồ luồng được nhóm sử dụng **Mermaid.js** để nhúng trực tiếp vào file, kết hợp với **Draw.io** cho các sơ đồ có độ phức tạp cao.
*   Ngoài ra, nhóm cũng tận dụng tối đa **Google Search, ChatGPT và Gemini** để tra cứu nhanh các tiêu chuẩn viết Use Case và thu thập dữ liệu phục vụ phần Benchmarking.

Dạ tiếp theo, để tối ưu thời gian, mỗi thành viên sẽ chia sẻ nhanh về phần đặc tả thử thách nhất hoặc thú vị nhất mà mình phụ trách ạ."

---

## 2. Phần của Từng Thành Viên

### 2.1. Thái: Hệ thống, ERD & Module AI
**Thái:** 
"Về phần em, em phụ trách kiến trúc tổng thể và module AI. Điểm khó nhất khi viết SRS cho tính năng AI là nó không giống các chức năng thêm/sửa/xóa thông thường. Mình phải định nghĩa rất rõ đầu vào và đầu ra. Em đã thiết kế luồng 'Kiểm duyệt lai' (Hybrid Moderation) để phân định rõ khi nào AI quét tự động, khi nào Admin cần can thiệp thủ công. Ngoài ra, việc thiết kế ERD kết hợp song song cả PostgreSQL và MongoDB cũng là một thử thách khá thú vị trong khâu phân tích dữ liệu."

### 2.2. Thắng: Trải nghiệm Độc giả & App Mobile
**Thắng:**
"Dạ phần của em là phân tích tính năng cho Độc giả. Phần em thấy phức tạp nhất lúc làm tài liệu là đặc tả tính năng **Bình luận theo đoạn văn (Inline Commenting)**. Viết Use Case cho các luồng cơ bản thì khá đơn giản, nhưng để mô tả kịch bản người dùng bôi đen đoạn chữ, app lưu lại vị trí (anchor) rồi hiển thị đúng bình luận lên đó thì đòi hỏi sự logic cao. Em đã phải vẽ các luồng rẽ nhánh (Exception Flow) rất chi tiết để lường trước các trường hợp rớt mạng hoặc khi người dùng thay đổi phông chữ."

### 2.3. Trung: Luồng Sáng tác & Quản lý Tác phẩm
**Trung:**
"Dạ em phụ trách luồng Đăng truyện cho Tác giả. Điểm mới mẻ nhất tụi em đưa vào là cho phép AI kiểm tra nội dung ngay lúc tác giả đang soạn thảo. Vì vậy, khi vẽ sơ đồ luồng (Activity Diagram), em phải mô tả rất kỹ trình tự: *Tác giả viết -> AI quét lỗi -> Lưu nháp -> Đủ 5 chương mới cho phép gửi duyệt -> Admin duyệt*. Em cố gắng liệt kê đầy đủ các điều kiện ràng buộc để team dev sau này đọc tài liệu là có thể nắm bắt và lập trình được ngay."

### 2.4. Tùng: Phân quyền & Admin
**Tùng:**
"Em đảm nhận phần Account, Phân quyền và Admin. Điểm tốn nhiều thời gian nhất là xây dựng **Bảng Phân Quyền (Permission Matrix)** chi tiết cho 5 nhóm đối tượng người dùng khác nhau. Kế đó, em phải mô tả quy trình cấp Token (Access/Refresh Token) lúc người dùng đăng nhập thông qua sơ đồ luồng dữ liệu sao cho rõ ràng và mang tính kỹ thuật nhất. Em cũng phụ trách khâu phân tích số liệu từ các nền tảng khác để hoàn thiện phần Tổng quan dự án."

---

## 3. Tổng kết (Thái)
**Thái:**
"Dạ tóm lại, việc hoàn thiện bộ tài liệu SRS này giúp nhóm em biết cách ứng dụng các công cụ như Markdown, Mermaid để diễn đạt ý tưởng hệ thống và luồng AI một cách chuyên nghiệp và rõ ràng hơn.
Dạ phần trình bày tóm tắt của nhóm đến đây là hết. Nhóm em cảm ơn Thầy đã lắng nghe và rất mong nhận được góp ý từ Thầy ạ!"
