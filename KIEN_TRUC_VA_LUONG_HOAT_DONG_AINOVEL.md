# BÁO CÁO PHÂN TÍCH CHUYÊN SÂU: MỔ XẺ KIẾN TRÚC, LUỒNG HOẠT ĐỘNG VÀ ÁNH XẠ ĐIỂM YẾU CỦA AINOVEL-CLI

> **Tài liệu Kỹ thuật Toàn diện:** Mổ xẻ chi tiết từng tầng kiến trúc, máy trạng thái (State Machine), chu kỳ vòng đời của 4 Subagent và ánh xạ trực tiếp từng thành phần vào các lỗ hổng/điểm nghẽn hiệu năng khi vận hành thực tế.

---

## MỤC LỤC
1. [Tổng quan Kiến trúc Hệ thống & Mô hình Phân lớp](#1-tổng-quan-kiến-trúc-hệ-thống--mô-hình-phân-lớp)
2. [Mổ xẻ 4 Vai trò Agent & Cơ chế Hoạt động](#2-mổ-xẻ-4-vai-trò-agent--cơ-chế-hoạt-động)
3. [Luồng Vận Hành Toàn Trình (End-to-End Lifecycle Flow)](#3-luồng-vận-hành-toàn-trình-end-to-end-lifecycle-flow)
4. [Cơ chế Ngữ cảnh & Hệ thống Nén Token (Context Engine)](#4-cơ-chế-ngữ-cảnh--hệ-thống-nén-token-context-engine)
5. [Tuyến Phòng Thủ: Stop Guard, Flow Router & Timeout Watchdog](#5-tuyến-phòng-thủ-stop-guard-flow-router--timeout-watchdog)
6. [MA TRẬN ÁNH XẠ: Kiến trúc $\longleftrightarrow$ Điểm Yếu Thực Tế](#6-ma-trận-ánh-xạ-kiến-trúc--điểm-yếu-thực-tế)
7. [Bản Thiết Kế Tái Cấu Trúc (Refactoring Blueprint)](#7-bản-thiết-kế-tái-cấu-trúc-refactoring-blueprint)

---

## 1. Tổng quan Kiến trúc Hệ thống & Mô hình Phân lớp

`ainovel-cli` là một **Hệ điều hành Sáng tác Đa Tác nhân (Multi-Agent Novel Engine)** được xây dựng bằng ngôn ngữ Go, áp dụng mô hình phân tách trách nhiệm nghiêm ngặt (Separation of Concerns).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          1. GIAO DIỆN & ĐIỀU KHIỂN                          │
│        Terminal UI (BubbleTea / Lipgloss)  │  Headless Automation CLI       │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   2. TẦNG ĐIỀU PHỐI TRUNG TÂM (COORDINATOR)                 │
│      Flow Router  │  State Machine  │  Stop Guard  │  Context Compactor     │
└───────────────────┬──────────────────┬───────────────────┬──────────────────┘
                    │                  │                   │
                    ▼                  ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      3. TẦNG CÁC AGENT CHUYÊN TRÁCH                         │
│  🏛️ Architect (Lập đề cương) │  ✍️ Writer (Viết văn)  │  🧐 Editor (Biên tập)│
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      4. TẦNG CÔNG CỤ & STORE DỮ LIỆU                        │
│  novel_context │ plan_chapter │ draft_chapter │ check_consistency │ commit  │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Store: layered_outline.json │ drafts/ │ chapters/ │ progress.json │ meta/  │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     5. TẦNG KẾT NỐI MÔ HÌNH (LLM ADAPTER)                   │
│      LiteLLM Go Wrapper  │  SSE Stream Watchdog (5m)  │  Role Failover      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Các nguyên tắc thiết kế bất di bất dịch của Engine:
1. **Tool-Driven Persistence (Chỉ lưu khi có Tool Call):** AI không thể lưu dữ liệu bằng văn bản tự do. Mọi thao tác ghi đĩa bắt buộc phải thông qua Tool Call có schema JSON xác thực.
2. **State Isolation (Cách ly trạng thái):** Mỗi Agent chạy trong một phiên (Session) độc lập. Agent phụ không thể giao tiếp trực tiếp với nhau mà phải thông qua Coordinator và dữ liệu dùng chung trong `Store`.
3. **Dual-Representation (Biểu diễn kép):** Mọi cấu trúc dữ liệu nền tảng đều được lưu song song:
   - Dạng máy đọc: `JSON` (để query, validate, duyệt trạng thái).
   - Dạng người đọc: `Markdown` (để hiển thị và chèn vào prompt).

---

## 2. Mổ xẻ 4 Vai trò Agent & Cơ chế Hoạt động

```
┌─────────────────┐       dispatch       ┌──────────────────┐
│                 ├─────────────────────►│  ARCHITECT_LONG  │ (Lập đề cương phân lớp)
│                 │                      └─────────┬────────┘
│                 │       dispatch                 │ save_foundation
│                 ├─────────────────────┐          ▼
│   COORDINATOR   │                     │   ┌──────────────┐
│  (Host Driver)  │       dispatch      ├──►│ STORE / DISK │
│                 ├──────────────┐      │   └──────────────┘
│                 │              │      │          ▲
│                 │              ▼      │          │ commit_chapter
│                 │       ┌─────────────┴┐         │
│                 ├──────►│    WRITER    ├─────────┘ (Thực thi 5 bước viết)
│                 │       └──────────────┘
│                 │       dispatch       ┌──────────────────┐
│                 └─────────────────────►│      EDITOR      │ (Đánh giá Cung 10 chương)
└─────────────────┘                      └──────────────────┘
```

### 2.1. 🎯 Điều Phối Viên (`Coordinator`)
* **Trách nhiệm:** "Tổng đạo diễn" của cuốn sách. Đọc `progress.json`, phân tích tiến độ hiện tại để quyết định:
  - Khi nào cần lập kế hoạch (`dispatch architect`).
  - Khi nào cần viết chương tiếp theo (`dispatch writer`).
  - Khi nào cần đánh giá Cung truyện (`dispatch editor`).
* **Đặc điểm:** Không trực tiếp viết văn xuôi. Chỉ nhận nhiệm vụ từ người dùng và phái sinh subagent.

### 2.2. 🏛️ Kiến Trúc Sư (`Architect_Long` & `Architect_Short`)
* **Trách nhiệm:** Xây dựng nền móng tác phẩm (`save_foundation`):
  1. `premise`: Tên truyện, tiền đề cốt truyện, phong cách nghệ thuật.
  2. `characters`: Hồ sơ nhân vật (tên, vai trò, tier, đặc điểm).
  3. `world_rules`: Quy tắc thế giới, hệ thống sức mạnh, cấm kỵ.
  4. `layered_outline` / `outline`: Phân tầng Volume $\rightarrow$ Arc $\rightarrow$ Chapter Skeleton.
  5. `update_compass`: La bàn định hướng kết cục.

### 2.3. ✍️ Người Viết (`Writer`)
* **Trách nhiệm:** Viết từng chương truyện theo **Quy trình 5 bước nghiêm ngặt**:
  ```
  [1] plan_chapter    --> Lập dàn ý xung đột, mục tiêu, nhịp điệu chương
  [2] draft_chapter   --> Viết bản nháp thô (lưu tạm vào drafts/XX.draft.md)
  [3] read_chapter    --> Tự đọc lại bản nháp từ đĩa
  [4] check_consistency -> Đối chiếu mâu thuẫn nhân vật và world building
  [5] commit_chapter  --> Chốt lưu chính thức vào chapters/XX.md
  ```

### 2.4. 🧐 Biên Tập Viên (`Editor`)
* **Trách nhiệm:** Kích hoạt định kỳ (mỗi khi hết 1 Arc hoặc mỗi 10 chương):
  - Đọc lại toàn bộ các chương trong Arc vừa qua.
  - Phân tích 7 chiều chất lượng (Nhịp độ, Tính cách nhân vật, Phục bút, Văn phong).
  - Xuất file tóm tắt Cung `save_arc_summary` để nén bộ nhớ cho các chương sau.

---

## 3. Luồng Vận Hành Toàn Trình (End-to-End Lifecycle Flow)

Dưới đây là sơ đồ chuyển trạng thái chi tiết của một cuốn tiểu thuyết:

```
[BẮT ĐẦU] ──► Nhập Prompt Người dùng
   │
   ▼
[GIAI ĐOẠN LẬP KẾ HOẠCH (Phase: "outline")]
   │
   ├──► Coordinator phái Architect_Long
   │       │
   │       ├── 1. Lưu Premise (Tên truyện)
   │       ├── 2. Lưu Characters & World Rules
   │       ├── 3. Lưu Layered Outline (Volume 1, 2...)
   │       ├── 4. Expand Arc (Danh sách Chương 1 -> 10)  ◄── [ĐIỂM NGHẼN 1]
   │       └── 5. Update Compass
   │
   ▼
[GIAI ĐOẠN VIẾT CHƯƠNG (Phase: "writing")]
   │
   ├──► Coordinator phái Writer (Chương N)
   │       │
   │       ├── 1. novel_context(chương N)
   │       ├── 2. plan_chapter(chương N)
   │       ├── 3. draft_chapter(mode="write", chapter N) ◄── [ĐIỂM NGHẼN 2]
   │       ├── 4. read_chapter(source="draft")
   │       ├── 5. check_consistency(chương N)
   │       └── 6. commit_chapter(chương N)               ◄── [ĐIỂM NGHẼN 3]
   │                 │
   │                 ├── Kiểm tra CheckArcBoundary(N)
   │                 └── Ghi file chapters/NN.md
   │
   ├──► Kiểm tra Ranh giới Arc:
   │       │
   │       ├── N < 10: Lặp lại viết chương tiếp theo (N+1)
   │       └── N == 10 (Cuối Arc): Chuyển sang Editor
   │
   ▼
[GIAI ĐOẠN BIÊN TẬP & MỞ RỘNG (Phase: "reviewing")]
   │
   ├──► Editor đánh giá Cung & lưu arc_summary
   ├──► Coordinator gọi Architect mở rộng Arc tiếp theo (Chương 11 -> 20)
   └──► Quay lại Giai đoạn Viết Chương
```

---

## 4. Cơ chế Ngữ cảnh & Hệ thống Nén Token (Context Engine)

Để phục vụ cho việc viết truyện dài vô tận, `ainovel-cli` trang bị một bộ nén ngữ cảnh tự động (**Context Compactor**):

```
                     ┌──────────────────────────────────┐
                     │ Tổng Token phiên > context_window│
                     └─────────────────┬────────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    ▼                                     ▼
        ┌───────────────────────┐             ┌───────────────────────┐
        │ Chiến lược LIGHT_TRIM │             │ Chiến lược FULL_SUMM  │
        │ (Cắt tỉa nhẹ)         │             │ (Tóm tắt toàn diện)   │
        └───────────┬───────────┘             └───────────┬───────────┘
                    │                                     │
         Xóa bỏ các Tool Call cũ              Gọi chính LLM đó để viết
         và kết quả trung gian                tóm tắt toàn bộ lịch sử
                    │                                     │
                    ▼                                     ▼
         Giảm 30% - 50% Token                  Giảm 60% - 75% Token
                                               (Tốn thêm 1 lượt gọi LLM)
```

---

## 5. Tuyến Phòng Thủ: Stop Guard, Flow Router & Timeout Watchdog

1. **Stop Guard (`stop_guard.go`):**
   - Chặn không cho AI tự ý kết thúc hội thoại (`end_turn`) khi chưa hoàn thành nhiệm vụ (ví dụ: chưa gọi `commit_chapter`).
   - Cho phép nhắc nhở tối đa 5 lần liên tiếp. Nếu lần thứ 6 vẫn vi phạm $\rightarrow$ **Cưỡng chế ngắt luồng (`terminate`)**.
2. **Flow Router (`host/flow`):**
   - Bộ định tuyến thông minh xác định trạng thái tiếp theo dựa trên sự kiện vừa xảy ra mà không cần chờ người dùng gõ lệnh.
3. **SSE Stream Watchdog (`models.go`):**
   - Đồng hồ đếm ngược **5 phút (`streamIdleTimeout = 5m0s`)**. Nếu trong 5 phút không nhận được byte nào từ máy chủ LLM $\rightarrow$ Tự động hủy kết nối.

---

## 6. MA TRẬN ÁNH XẠ: Kiến trúc $\longleftrightarrow$ Điểm Yếu Thực Tế

Bảng dưới đây ánh xạ trực tiếp **từng khối kiến trúc** trong mã nguồn Go của `ainovel-cli` với các **triệu chứng lỗi cụ thể** đã được phân tích trong [BAO_CAO_DIEM_YEU_KIEN_TRUC_LLM.md](file:///c:/Users/Laptop/OneDrive/Documents/Notebooks/New%20folder%20(4)/ainovel-cli/BAO_CAO_DIEM_YEU_KIEN_TRUC_LLM.md):

| Khối Kiến Trúc / File Mã Nguồn | Chức năng Thiết kế | Hành vi Lỗi Thực tế khi chạy Model 8B/Context Lớn | Ánh xạ mục trong Báo Cáo Điểm Yếu |
|---|---|---|---|
| 📡 **`internal/bootstrap/models.go`** (`streamIdleTimeout = 5m0s`) | Hủy kết nối nếu mạng chết | Prompt 30k tokens trên GPU T4 tính toán mất 6 phút $\rightarrow$ **Bị ngắt nhầm kết nối (Stream Idle Timeout)** liên tục. | **Mục 1.1** *(TTFT Bottleneck & Timeout)* |
| 🔄 **`internal/host/context_compactor.go`** (`light_trim / full_summary`) | Nén bộ nhớ phiên tự động | Model 8B bị kẹt ở khâu lập kế hoạch $\rightarrow$ Kích hoạt nén tóm tắt liên tục $\rightarrow$ **Đốt sạch 800.000 tokens vô ích**. | **Mục 1.3** *(Compaction Loop Overhead)* |
| 🛡️ **`internal/host/reminder/stop_guard.go`** (`max consecutive = 5`) | Ép AI phải gọi công cụ lưu đĩa | Model 8B in văn bản trực tiếp thay vì gọi hàm $\rightarrow$ Stop guard nhắc 5 lần $\rightarrow$ **Cưỡng chế hủy luồng (`run terminated`)**. | **Mục 2.1** *(Tool Calling Failure & Stop Guard Escalation)* |
| 🏛️ **`internal/tools/save_foundation.go`** (`decodeFoundationJSON`) | Xác thực cấu trúc đề cương đa tầng | Cấu trúc 4 tầng quá sâu $\rightarrow$ Model 8B sinh sai JSON $\rightarrow$ **Lỗi `cannot unmarshal object into []VolumeOutline`**. | **Mục 2.2** *(Deeply Nested JSON Schema Failure)* |
| 🔒 **`internal/tools/commit_chapter.go`** (Dòng 153–161: `CheckArcBoundary`) | Chặn Writer viết vượt quá Cung | Model 8B chưa gọi `expand_arc` (`arcs: null`) $\rightarrow$ **Chặn lưu chương 1 (`Precondition Failed`)**. | **Mục 3** *(Deadlock Khóa Chết - Vế 1)* |
| 🚫 **`internal/tools/save_foundation.go`** (Dòng 74–77: `isWriting()`) | Cấm phá hỏng đề cương cũ | Cấm sửa `layered_outline` khi ở Phase Writing $\rightarrow$ **Architect không thể bổ sung Arc bị thiếu**. | **Mục 3** *(Deadlock Khóa Chết - Vế 2)* |

---

## 7. Bản Thiết Kế Tái Cấu Trúc (Refactoring Blueprint)

Dành cho các nhà phát triển xây dựng **Writing Copilot** hoặc nâng cấp `ainovel-cli` trong tương lai:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   CHIẾN LƯỢC TÁI CẤU TRÚC ĐỘC LẬP MODEL                     │
│                                                                             │
│  [1] ENGINE-LEVEL RESILIENCE (Tự phục hồi cấp Động cơ):                     │
│      - Tự động bù đắp Default Arc khi arcs: null (Zero Precondition Crash). │
│      - Mở khóa quyền cứu hộ đề cương trong Phase Writing.                   │
│                                                                             │
│  [2] ASYMMETRIC MODEL ROUTING (Định tuyến Mô hình Bất đối xứng):             │
│      - Fast LPU / Frontier API (Groq / Mistral / DeepSeek) cho Architect.   │
│      - Local GPU / Creative LLM (Qwen / Colab) cho Writer.                  │
│                                                                             │
│  [3] DYNAMIC CONTEXT SLICING (Cắt tỉa ngữ cảnh theo nhu cầu):               │
│      - Thay vì nạp toàn bộ 27KB World Setting vào mọi prompt, chỉ nạp đoạn  │
│        Setting có liên quan đến các Entity xuất hiện trong cảnh đó.         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Kết Luận

Kiến trúc của `ainovel-cli` là một hệ thống xuất sắc và hoàn chỉnh bậc nhất về mặt phân chia tác vụ văn học. Các lỗi "chạy mãi không ra chương" hay "đốt hàng trăm ngàn tokens" hoàn toàn không phải do lỗi tư duy câu chuyện, mà là do **sự thiếu đồng bộ giữa các giả định thiết kế lý tưởng và năng lực thực tế của các mô hình AI nhỏ**. 

Bằng cách nắm vững ma trận ánh xạ kiến trúc này, chúng ta có thể làm chủ hoàn toàn hệ thống, tự tin vận hành trên mọi nền tảng phần cứng từ laptop 4GB RAM đến các siêu máy chủ GPU đám mây.
