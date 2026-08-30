# BÁO CÁO KỸ THUẬT: ĐIỂM YẾU KIẾN TRÚC CỦA AINOVEL-CLI KHI CHẠY VỚI NGỮ CẢNH ĐẦU VÀO LỚN VÀ MÔ HÌNH LLM HẠN CHẾ

> **Tài liệu phân tích chuyên sâu:** Đánh giá hiệu năng, hành vi máy trạng thái (State Machine), và các rào cản kỹ thuật khi triển khai `ainovel-cli` trên mô hình ngôn ngữ nhỏ cục bộ (Local LLMs 8B – 14B) và môi trường ngữ cảnh lớn (>15.000 tokens).

---

## Executive Summary (Tóm tắt điều hành)

Hệ thống `ainovel-cli` được thiết kế theo mô hình **Multi-Agent phối hợp tự động** (Coordinator, Architect, Writer, Editor) với kỳ vọng vận hành trơn tru các tác phẩm trường thiên (100–200 chương). Tuy nhiên, kiến trúc ban đầu của dự án được xây dựng dựa trên giả định rằng hệ thống luôn chạy cùng các **Frontier Models siêu cấp** (như Claude 3.5 Sonnet, GPT-4o).

Khi chuyển đổi sang chạy trên các **Mô hình Ngôn ngữ cục bộ (Local LLMs 8B / 14B)** hoặc máy chủ GPU miễn phí (Kaggle/Colab) với tài liệu thế giới quan dài (>25KB ~ 15.000 tokens), hệ thống bộc lộ **4 nhóm điểm yếu cốt lõi**:
1. Hiện tượng nghẽn thời gian xử lý Prompt ban đầu (TTFT Bottleneck) dẫn đến vượt ngưỡng Stream Timeout (5 phút).
2. Lỗi bế tắc logic trạng thái (**State Machine Deadlock**) giữa công cụ kiểm tra ranh giới Arc và quyền ghi đè đề cương.
3. Sự thiếu ổn định trong cơ chế gọi hàm (**Tool / Function Calling**) của các mô hình kích thước nhỏ.
4. Vòng lặp nén ngữ cảnh lặp đi lặp lại (**Compaction Loop**), tiêu tốn hàng trăm ngàn tokens vô ích.

---

## 1. Vấn đề khi Ngữ cảnh Đầu vào Quá Lớn (>15.000 Tokens)

```
┌────────────────────────────────────────────────────────────────────────┐
│                        THÀNH PHẦN MỘT PROMPT GỬI ĐI                    │
│ ┌──────────────────────┐ ┌───────────────────┐ ┌─────────────────────┐ │
│ │ World Setting (27KB) │ │ Rules + Guideline │ │ Session History     │ │
│ │ (~10.000 tokens)     │ │ (~3.000 tokens)   │ │ (~15.000+ tokens)   │ │
│ └──────────────────────┘ └───────────────────┘ └─────────────────────┘ │
└───────────────────────────────────┬────────────────────────────────────┘
                                    ▼  (Tổng: 25.000 - 45.000 tokens)
                  ┌──────────────────────────────────┐
                  │ GPU Pre-fill / TTFT Calculation  │
                  └─────────────────┬────────────────┘
                                    ▼
       ┌─────────────────────────────────────────────────────────┐
       │ - Tesla T4 / PCIe Bottleneck: Mất 300s - 600s           │
       │ - Vượt ngưỡng: stream_idle_timeout = 5m0s               │
       │ => TRIGGER TIMEOUT ERROR => BỊ HỦY LUỒNG!               │
       └─────────────────────────────────────────────────────────┘
```

### 1.1. Nghẽn thời gian tính toán Token đầu tiên (Time To First Token - TTFT)
* **Cơ chế**: Thuật toán Attention có độ phức tạp tính toán $O(N^2)$ đối với độ dài chuỗi đầu vào $N$. Khi nạp toàn bộ `world_setting.md` (27KB), `novel_rules.md`, lịch sử hội thoại và đề cương phân lớp, tổng số token đầu vào của mỗi lượt `novel_context` thường dao động từ **25.000 đến 45.000 tokens**.
* **Hậu quả trên GPU nhỏ**: Trên GPU đơn (như Tesla T4 300 GB/s) hoặc cụm 2 GPU không có NVLink, việc tính toán ma trận KV-cache cho 30.000 tokens mất từ **4 đến 8 phút**.
* **Xung đột Timeout**: Trong file Go `internal/bootstrap/models.go`, hệ thống hardcode:
  ```go
  const streamIdleTimeout = 5 * time.Minute
  ```
  Nếu sau 5 phút máy chủ backend chưa bắt đầu sinh ra token đầu tiên (do còn đang tính toán prompt pre-fill), `ainovel-cli` sẽ tự động hủy kết nối (`stream idle timeout`), kích hoạt vòng lặp thử lại vô tận.

### 1.2. Hiện tượng "Lạc trôi ở giữa" (Lost in the Middle) & Giảm chú ý
* Khi kích thước ngữ cảnh phình to vượt quá 16.000 tokens, các mô hình mã nguồn mở 8B/14B bắt đầu suy giảm khả năng chú ý (Attention Degradation).
* Mô hình chỉ ghi nhớ phần đầu prompt (System Prompt) và phần cuối prompt (User Message gần nhất), hoàn toàn bỏ quên các chi tiết nằm ở giữa (như tên nhân vật phụ, nhánh rẽ Cung truyện, hay quy tắc cấm dùng từ sáo rỗng).

### 1.3. Vòng lặp nén ngữ cảnh tốn kém (Compaction Overhead)
* Khi tổng số token vượt ngưỡng cấu hình (`context_window: 16384`), hệ thống kích hoạt cơ chế `light_trim` hoặc `full_summary`.
* Để nén một phiên làm việc 40.000 tokens, Coordinator phải gọi chính LLM đó để viết tóm tắt (mất 40–70 giây và tiêu hao 40.000 tokens mỗi lần tóm tắt).
* Khi mô hình bị kẹt ở khâu lên kế hoạch, chu kỳ **Gửi prompt $\rightarrow$ Vượt ngưỡng $\rightarrow$ Gọi LLM tóm tắt $\rightarrow$ Gửi lại prompt** đã đốt sạch **hơn 800.000 tokens** mà không sinh ra được một dòng văn bản tiểu thuyết nào.

---

## 2. Điểm Yếu Cốt Lõi Khi Chạy Trên Model Nhỏ (8B / 14B)

### 2.1. Độ chính xác gọi hàm không ổn định (Tool / Function Calling Failure)
* **Hiện tượng in văn bản thô (Direct Streaming)**: 
  * Các mô hình 8B thường gặp khó khăn trong việc phân định giữa **"Văn bản hội thoại thông thường"** và **"Tham số truyền vào hàm JSON"**.
  * Thay vì gọi `draft_chapter(content="...")`, mô hình lại in thẳng chương truyện ra màn hình terminal dưới dạng câu trả lời văn bản.
  * Vì `ainovel-cli` là một Event-driven Engine chỉ lưu file khi có Tool Call, đoạn văn bản in ra màn hình bị bỏ trôi và **không có bất kỳ file nháp nào được tạo trên ổ cứng**.
* **Hiện tượng lặp lại Stop Guard**:
  * Khi mô hình kết thúc lượt mà không gọi công cụ, hệ thống gửi thông báo nhắc nhở (`stop_guard.go`): *"Bạn phải gọi draft_chapter / commit_chapter để lưu chương"*.
  * Mô hình 8B không hiểu rõ cơ chế này, tiếp tục suy nghĩ và in text ra màn hình $\rightarrow$ Kích hoạt 6 lần cảnh báo liên tiếp cho đến khi bị hệ thống cưỡng chế đóng luồng (`run terminated`).

### 2.2. Lỗi cú pháp với Schema JSON lồng nhau sâu (Deeply Nested Schema)
* Cấu trúc đề cương phân lớp của `ainovel-cli` có độ lồng nhau rất sâu:
  $$\text{VolumeOutline} \longrightarrow \text{ArcOutline} \longrightarrow \text{OutlineEntry} \longrightarrow \text{Scenes [ ]}$$
* **Hành vi lỗi thực tế trong Log**:
  * Mô hình 8B xuất Object `{ ... }` thay vì Mảng `[ { ... } ]` $\rightarrow$ Lỗi `cannot unmarshal object into Go value of type []domain.VolumeOutline`.
  * Quên escape dấu ngoặc kép bên trong chuỗi văn bản $\rightarrow$ Lỗi `invalid character '\n' in string`.
  * Quên điền các trường bắt buộc như `ending_direction`, `mode`, `tier`.

### 2.3. Chuyển đổi trạng thái vội vã (Premature Phase Transition)
* Khi được yêu cầu lập kế hoạch, mô hình 8B sau khi lưu khung tập rỗng (`arcs: null`) thường tự suy diễn rằng *"Đề cương đã hoàn tất"* và kích hoạt cờ `foundation_ready: true`.
* Nó bỏ qua hoàn toàn việc gọi công cụ `expand_arc` để mở rộng danh sách chi tiết các chương của Cung 1, dẫn đến việc chuyển sang giai đoạn viết khi cơ sở dữ liệu đề cương vẫn đang bị rỗng.

---

## 3. Phân tích Lỗi Khóa Chết Trạng Thái (State Machine Deadlock)

Đây là điểm nghẽn nghiêm trọng nhất trong mã nguồn Go của dự án khi gặp phải sự cố từ mô hình AI nhỏ:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       SƠ ĐỒ KHÓA CHẾT (DEADLOCK CYCLE)                      │
│                                                                             │
│  [1] Architect (8B) lưu Volume rỗng -> Không gọi expand_arc (arcs: null)    │
│                                │                                            │
│                                ▼                                            │
│  [2] Coordinator chuyển Phase = "writing" -> Phái Writer viết Chương 1      │
│                                │                                            │
│                                ▼                                            │
│  [3] Writer viết xong 01.draft.md -> Gọi commit_chapter(1)                  │
│                                │                                            │
│                                ▼                                            │
│  [4] commit_chapter.go: CheckArcBoundary(1) == nil (Do arcs: null)          │
│      => TỪ CHỐI LƯU CHƯƠNG (Precondition Failed)                            │
│                                │                                            │
│                                ▼                                            │
│  [5] Coordinator thấy lỗi -> Gọi lại Architect để bổ sung đề cương          │
│                                │                                            │
│                                ▼                                            │
│  [6] Architect gọi save_foundation(layered_outline)                         │
│      => save_foundation.go chặn: "Đang ở Phase writing, cấm sửa outline!"    │
│                                │                                            │
│                                └─────────► QUAY LẠI BƯỚC [4] (VÒNG LẶP VÔ HẠN)
└─────────────────────────────────────────────────────────────────────────────┘
```

* **Vấn đề thiết kế**: Tác giả đã đặt 2 chốt chặn phòng thủ độc lập nhưng lại **mâu thuẫn logic khi xảy ra sự cố**:
  1. `commit_chapter.go` không cho phép lưu nếu chương nằm ngoài danh sách Arc.
  2. `save_foundation.go` không cho phép cập nhật Arc nếu hệ thống đã bước vào giai đoạn `writing`.
* **Hệ quả**: Nếu một mô hình AI vô tình bước vào giai đoạn `writing` khi danh sách Arc bị rỗng, hệ thống sẽ bị đóng băng hoàn toàn và không có đường thoát nếu không có can thiệp thủ công từ con người.

---

## 4. Đề Xuất Kiến Trúc & Hướng Khắc Phục Triệt Để

### 4.1. Vá Mã Nguồn Go (Engine-level Self-Healing)
Cần bổ sung cơ chế **Tự động phục hồi (Self-Healing Fallback)** vào 2 điểm kiểm tra trong mã nguồn Go:

1. **Tại `internal/tools/commit_chapter.go`**:
   * Nếu `CheckArcBoundary(chapter)` trả về `nil` do đề cương thiếu Arc, hệ thống không được ném lỗi dừng chương trình.
   * *Giải pháp*: Tự động khởi tạo một Cung mặc định (Volume 1, Arc 1) bao bọc các chương từ `1` đến `max(chapter, 10)` và ghi nhận vào Store trước khi hoàn tất commit.
2. **Tại `internal/tools/save_foundation.go`**:
   * Cho phép nạp đè `layered_outline` trong giai đoạn `writing` nếu phát hiện dữ liệu đề cương hiện tại đang bị rỗng hoặc chưa có bất kỳ Arc nào được mở rộng.

### 4.2. Kiến Trúc Phân Tầng Mô Hình Theo Vai Trò (Role-based Model Tiering)
Không nên sử dụng duy nhất một model nhỏ cho toàn bộ 4 vai trò. Thay vào đó, áp dụng chiến lược kết hợp tối ưu:

| Vai trò Agent | Đặc thù tác vụ | Mô hình tối ưu khuyên dùng | Lý do |
|---|---|---|---|
| 🏛️ **Architect** | Cần tuân thủ JSON 100%, tư duy bao quát | **Groq LPU / Mistral / DeepSeek-R1** | Xử lý cấu trúc đa tầng trong 2 giây, không bao giờ lỗi JSON. |
| 🎯 **Coordinator** | Định tuyến nhanh, quản lý phiên | **Groq LPU / Gemini Flash** | Độ trễ cực thấp, không bị nghẽn ngữ cảnh. |
| ✍️ **Writer** | Viết văn xuôi cảm xúc, Show Don't Tell | **Local Qwen (8B/14B) / Colab / Kaggle** | Tập trung 100% sức mạnh cho việc miêu tả nghệ thuật, không vướng bận JSON. |
| 🧐 **Editor** | Kiểm tra mâu thuẫn, tổng hợp Cung | **Local Qwen 14B / DeepSeek** | Đánh giá khách quan, tóm tắt chính xác. |

### 4.3. Tinh Giản Hồ Sơ Nền Tảng (Context Pruning Strategy)
* **Giảm tải `world_setting.md`**: Tách nhỏ file thế giới quan 27KB thành các phân đoạn động (Dynamic Slices). Chỉ nạp thông tin về *Ma pháp* khi nhân vật thi triển phép, chỉ nạp thông tin về *Vũ khí* khi có cảnh chiến đấu.
* **Cố định `context_window`**: Ghim ở mức `16.384` hoặc `8.192` trên các máy chủ GPU miễn phí để triệt tiêu hoàn toàn thời gian chờ Pre-fill.

---

## 5. Kết Luận

`ainovel-cli` là một framework sáng tác tiểu thuyết có thiết kế logic và chia tầng tác vụ rất xuất sắc. Tuy nhiên, để hệ thống có thể vận hành ổn định trên các mô hình mã nguồn mở cục bộ (8B/14B), framework bắt buộc phải được bổ sung các **tuyến phòng thủ mềm dẻo (Graceful Degradation)** và **cơ chế tự sửa sai đề cương**, thay vì dựa dẫm tuyệt đối vào năng lực gọi hàm hoàn hảo của các siêu mô hình độc quyền.
