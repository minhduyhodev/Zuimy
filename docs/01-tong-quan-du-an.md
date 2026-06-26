# Tổng quan dự án — Nền tảng học trực tuyến (Zuimy)

> Nền tảng bán khóa học online dạng video, lấy cảm hứng từ Udemy nhưng đơn giản hóa cho dự án cá nhân  
> Định hướng sản phẩm thực tế — Full-stack (Spring Boot 3.x + ReactJS Vite)

---

## 1. Ý tưởng dự án

### Vấn đề thực tế

Việt Nam hiện là quốc gia đứng thứ 2 thế giới về số người dùng nền tảng học online miễn phí, nhưng phần lớn nội dung là khóa học phổ thông (Toán, Văn, tiếng Anh). Mảng **kỹ năng nghề** (lập trình, thiết kế, kỹ năng mềm) do người Việt tạo và bán bằng tiếng Việt vẫn còn ít nền tảng làm tốt, đa số phải dùng Udemy quốc tế (giá USD, không hỗ trợ thanh toán nội địa).

### Giải pháp

Xây dựng nền tảng cho phép:

- **Khách viếng thăm (Guest):** Người dùng chưa đăng nhập. Có quyền xem trang chủ, tìm kiếm khóa học, xem trang chi tiết (mô tả, đề cương, đánh giá công khai), xem các bài học thử (Preview) miễn phí. Bắt buộc phải đăng nhập nếu muốn vào bài học chính thức hoặc thực hiện thanh toán (Checkout).
- **Giảng viên (Instructor):** Tự tạo khóa học, quản lý cấu trúc bài học, upload video bài học thông qua cơ chế tối ưu (dán link YouTube Unlisted hoặc tải lên trực tiếp Cloudflare R2), theo dõi số dư thu nhập (chia sẻ theo tỷ lệ cấu hình động), gửi yêu cầu rút tiền về tài khoản ngân hàng nội địa.
- **Học viên (Student):** Tìm kiếm và mua các khóa học kỹ năng nghề bằng VNĐ qua quét mã VietQR (được Staff đối soát phê duyệt) hoặc thanh toán tự động qua Stripe, học theo tiến độ cá nhân, làm bài tập trắc nghiệm (Quiz) đánh giá và nhận chứng chỉ (Certificate) tự động.
  - _Tính năng tương tác nâng cao:_ Gửi câu hỏi cho AI Chatbot trợ giảng, thảo luận/bình luận dưới mỗi bài học, đánh giá (1 - 5 sao) cho khóa học đã mua (`Enrollment` hợp lệ).
- **Nhân viên vận hành (Staff):** Đóng vai trò đối soát thanh toán và hỗ trợ vận hành:
  - Kiểm tra biến động số dư ngân hàng và đối chiếu với hình ảnh minh chứng chuyển khoản (Bill) do Học viên tải lên từ luồng VietQR để phê duyệt hoặc từ chối đơn hàng.
- **Quản trị viên (Admin / Owner):** Đóng vai trò kiểm duyệt và quản trị cấp cao:
  - Kiểm duyệt nội dung khóa học (Phê duyệt xuất bản hoặc Từ chối kèm lý do).
  - Quản lý danh mục ngành học, cấu hình mã giảm giá (Voucher) toàn sàn.
  - Quản lý tài chính (Cấu hình tỷ lệ chia sẻ doanh thu sàn, trực tiếp duyệt yêu cầu rút tiền thủ công của Giảng viên).
  - Quản lý tài khoản người dùng (Khóa/mở khóa tài khoản Staff, Instructor, Student) và cấu hình cài đặt hệ thống.

### Định hướng khác biệt

Tập trung **thanh toán quét mã VietQR tự động**, định giá VNĐ, tích hợp AI trợ giảng thông minh và tối ưu hóa chi phí hạ tầng video cho giảng viên Việt Nam.

---

## 2. Web tham khảo

| Web           | URL           | Học gì từ đây                                                 |
| ------------- | ------------- | ------------------------------------------------------------- |
| **Udemy**     | udemy.com     | Benchmark chính — giỏ hàng, voucher, trang học video, tiến độ |
| **Kyna.vn**   | kyna.vn       | UX người Việt, flow mua khóa học kỹ năng nghề                 |
| **Unica.vn**  | unica.vn      | Cách bundle nhiều khóa học vào 1 đơn hàng                     |
| **Coursera**  | coursera.org  | Progress tracking, certificate sau khi hoàn thành             |
| **Teachable** | teachable.com | Dashboard giảng viên — tạo và quản lý khóa học                |

### Điểm khác biệt so với web tham khảo

- **Đa dạng phương thức thanh toán (Stripe & VietQR):** Tích hợp cả thanh toán quốc tế tự động qua **Stripe** (nhận thẻ Visa/Mastercard và tự động mở khóa học qua Webhook) lẫn phương thức chuyển khoản nội địa bằng **quét mã VietQR** (Học viên tải bill lên hệ thống để Staff đối soát phê duyệt thủ công), đem lại sự tiện lợi tối đa cho thị trường Việt Nam.
- **Tăng quyền tự chủ cho Giảng viên:** Cung cấp mô hình hiển thị số dư doanh thu thời gian thực (được cộng sau khi áp dụng tỷ lệ chia sẻ và khấu trừ voucher). Giảng viên hoàn toàn chủ động cấu hình giá bán, gửi yêu cầu rút tiền về bất kỳ ngân hàng nội địa nào.
- **Tối ưu hóa chi phí hạ tầng Video:** Áp dụng **kiến trúc lưu trữ Hybrid (YouTube Unlisted + Cloudflare R2 Presigned URL)** giúp hệ thống vận hành mượt mà với chi phí hạ tầng tối thiểu nhờ tận dụng 10GB lưu trữ miễn phí và không tốn phí băng thông tải ra (Egress cost) của Cloudflare R2.
- **Trợ lý học tập thông minh (AI Assistant):** Tích hợp AI chatbot hỗ trợ học tập sử dụng **Gemini API (gói Free tier)** cực kỳ thông minh. AI đóng vai trò làm trợ giảng, đọc ngữ cảnh thông tin bài học hiện tại để giải đáp ngay lập tức các thắc mắc chuyên môn của học viên.

---

## 3. Tính năng chi tiết

### 👤 Học viên (Student)

- Đăng ký / đăng nhập (Email/Password hoặc Google OAuth)
- Duyệt và tìm kiếm khóa học theo danh mục, cấp độ
- Xem preview bài học miễn phí trước khi mua
- Thêm khóa học vào **giỏ hàng** (quản lý giỏ hàng bằng **Redis** hoặc lưu DB, reset TTL 48h khi có tương tác)
- Áp **mã voucher** (mã giảm giá toàn sàn hoặc mã giảm giá của riêng khóa học)
- Thanh toán qua quét mã **VietQR** (tải minh chứng chuyển khoản để Staff duyệt) hoặc tự động qua **Stripe** (thanh toán quốc tế)
- Xem bài học video, hệ thống tự lưu **tiến độ học**
- Tương tác qua khung **Thảo luận / Hỏi đáp (Q&A) dạng Bình luận** dưới mỗi video bài học và Chat với **AI Trợ giảng** bên cạnh video
- Làm **bài tập trắc nghiệm (Quiz)** cuối mỗi chương để củng cố kiến thức
- Nhận **certificate (PDF)** khi hoàn thành 100% tiến độ và đạt điểm Quiz yêu cầu
- Đánh giá và để lại nhận xét (Review) khóa học sau khi sở hữu khóa học thành công

### 🎓 Giảng viên (Instructor)

- Tạo khóa học: nhập tên, mô tả, danh mục, giá và cấp độ của khóa học
- Tạo chương (module), bài học (lesson) và **bài tập trắc nghiệm (Quiz)**
- Thêm video bài học qua cơ chế Hybrid Storage: dán link YouTube Unlisted hoặc upload file `.mp4` trực tiếp (Frontend xin Backend Presigned URL rồi tải thẳng lên Cloudflare R2 để tối ưu tài nguyên Server)
- Quản lý trạng thái khóa học: Yêu cầu phê duyệt xuất bản khóa học (`DRAFT` -> `PENDING_APPROVAL`)
- Dashboard tổng quan: Theo dõi số lượng học viên, tiến độ học trung bình, tổng doanh thu và quản lý số dư thu nhập
- Tạo yêu cầu rút tiền về tài khoản ngân hàng nội địa

### 👥 Nhân viên vận hành (Staff)

- **Đối soát thanh toán:** Xem danh sách các đơn hàng chọn thanh toán qua VietQR kèm hình ảnh minh chứng chuyển khoản (Bill). Đối chiếu biến động số dư tài khoản ngân hàng và thực hiện phê duyệt đơn hàng (`PAID`) hoặc từ chối duyệt đơn hàng (`REJECTED_PROOF`).

### 👑 Quản trị viên (Admin / Owner)

- **Kiểm duyệt khóa học (Course Approval):** Xem nội dung các khóa học đang chờ duyệt, phê duyệt (`PUBLISHED`) hoặc từ chối (`REJECTED` kèm lý do).
- **Quản lý danh mục & Voucher:** Tạo, sửa, xóa các danh mục ngành học; cấu hình mã giảm giá (Voucher) toàn sàn.
- **Quản lý tài chính & Duyệt rút tiền:** Theo dõi tổng doanh thu của hệ thống, cấu hình tỷ lệ chia sẻ doanh thu (phí sàn). Xem các yêu cầu rút tiền của Giảng viên, thực hiện chuyển khoản ngân hàng thủ công và nhấn "Xác nhận đã chuyển" để trừ số dư của Giảng viên trong DB.
- **Quản lý người dùng:** Khóa/mở khóa tài khoản của các Staff, Giảng viên và Học viên.
- **Cấu hình hệ thống:** Mail server, cấu hình API Stripe và thông tin tài khoản nhận chuyển khoản ngân hàng.

---

## 4. Luồng hệ thống chính

### Luồng 1 — Mua khóa học và xử lý thanh toán (VietQR & Stripe)

Hệ thống hỗ trợ 2 phương thức thanh toán chính:

#### 1. Thanh toán qua VietQR (Xác nhận thủ công bởi Staff/Admin)
*   **Happy Path (Thành công):**
    Học viên chọn phương thức VietQR -> Hệ thống tạo Đơn hàng (Trạng thái: `PENDING`) -> Giao diện hiển thị mã QR động chứa thông tin chuyển khoản (Số tài khoản, Số tiền, Nội dung chuyển khoản duy nhất) -> Học viên chuyển khoản qua App ngân hàng và tải lên hình ảnh minh chứng chuyển khoản (Bill) -> Đơn hàng chuyển sang trạng thái `SUBMITTED_PROOF` -> Nhân viên vận hành (Staff) kiểm tra đối soát biến động số dư ngân hàng thực tế:
    *   Nếu khớp giao dịch -> Staff nhấn "Phê duyệt" -> Đơn hàng chuyển sang `PAID` -> Tự động tạo bản ghi `Enrollment` -> Cộng tiền vào ví Giảng viên (sau khi áp dụng tỷ lệ chia sẻ và khấu trừ voucher) -> Xóa giỏ hàng Redis -> Đẩy sự kiện sang Kafka để gửi email xác nhận hóa đơn qua Spring Mail.
*   **Sad Path (Thất bại / Hủy):**
    Học viên chủ động hủy đơn hàng hoặc đơn hàng hết hạn thanh toán (sau 30 phút) mà không tải minh chứng -> Tự động chuyển thành `CANCELLED` và giải phóng voucher.
    Nếu minh chứng chuyển khoản tải lên bị sai lệch thông tin -> Staff nhấn "Từ chối duyệt" -> Đơn hàng chuyển sang `REJECTED_PROOF` kèm lý do từ chối gửi về email của Học viên để yêu cầu thực hiện lại hoặc liên hệ hỗ trợ.

#### 2. Thanh toán tự động qua Stripe (Thẻ quốc tế Visa/Mastercard)
*   **Happy Path (Thành công):**
    Học viên chọn phương thức Stripe -> Tạo Đơn hàng (Trạng thái: `PENDING`) -> Hệ thống gọi Stripe API khởi tạo Stripe Checkout Session -> Chuyển hướng Học viên sang trang thanh toán bảo mật của Stripe -> Học viên thanh toán thành công -> Stripe bắn Webhook về Backend (`payment_intent.succeeded`) -> Backend thực hiện xác thực chữ ký Webhook (Stripe Signature Verification) -> Thực hiện Database Transaction với khóa dòng (`SELECT ... FOR UPDATE`):
    *   Nếu đơn hàng đang ở trạng thái `PENDING` -> Cập nhật thành `PAID` -> Tự động tạo bản ghi `Enrollment` -> Cộng tiền vào ví Giảng viên -> Xóa giỏ hàng trong Redis -> Đẩy sự kiện sang Kafka để gửi email xác nhận.
    *   Nếu đơn hàng đã là `PAID` (do xử lý trùng) -> Dừng xử lý và trả về HTTP 200 ngay lập tức để tránh xử lý lặp lại (Idempotent API).
*   **Sad Path (Thất bại):**
    Thanh toán thất bại trên cổng Stripe -> Stripe trả về webhook lỗi -> Hệ thống cập nhật trạng thái Đơn hàng thành `FAILED`, hoàn trả số lượng mã voucher.

---

### Luồng 1.1 — Quy mô Voucher & Phân chia doanh thu (Revenue Split)
Hệ thống áp dụng nguyên tắc: "Chia sẻ theo tỷ lệ trên số tiền thực thu". Cách giải quyết cụ thể gồm 3 quy tắc:
1.  **Phân loại Voucher:** Hệ thống phân định rõ **Voucher Toàn sàn (Admin)** và **Voucher Khóa học (Giảng viên)**. Hai loại này có thể áp dụng đồng thời hoặc độc lập tùy thuộc vào cấu hình chiến dịch của hệ thống.
2.  **Quy tắc khấu trừ:** Khi tính toán số dư cho Giảng viên, hệ thống không dựa trên giá gốc ban đầu của khóa học mà căn cứ trên số tiền thực tế học viên thanh toán sau khi đã áp dụng toàn bộ các mã Voucher hợp lệ.
3.  **Công thức phân bổ động (Dynamic Revenue Split):**
    Gọi $R_{\text{rate}}$ là Tỷ lệ chia sẻ cho Giảng viên (%) và $P_{\text{rate}}$ là Tỷ lệ phí sàn của Admin (%) được cấu hình động trong hệ thống quản trị (Admin Dashboard), với điều kiện: $R_{\text{rate}} + P_{\text{rate}} = 100\%$.
    $$\text{Số tiền Giảng viên nhận} = (\text{Giá gốc} - \text{Tổng giảm giá Voucher}) \times R_{\text{rate}}$$
    $$\text{Số tiền Phí sàn (Admin)} = (\text{Giá gốc} - \text{Tổng giảm giá Voucher}) \times P_{\text{rate}}$$

### Luồng 2 — Học bài, bảo mật video & lấy chứng chỉ

```
Học viên vào "Khóa học của tôi" → Chọn khóa học đã mua → Hệ thống kiểm tra quyền truy cập (Enrollment) 
→ Học viên bấm vào bài học:
    1. Bảo mật API: API lấy đề cương ẩn hoàn toàn video_id. Frontend gọi API riêng /api/v1/lessons/{id}/video.
    2. Backend lấy user_id trực tiếp từ Spring Security Context (JWT), xác thực Enrollment hợp lệ.
    3. Nếu video nguồn R2: Backend sinh Presigned GET URL (hạn 15 phút) trả về cho Frontend phát.
    4. Nếu video nguồn YouTube: Backend trả về video_id Youtube, Frontend dùng ReactPlayer kết hợp 
       CSS overlay chặn chuột phải/click đúp để giấu link gốc Youtube.
    5. Cập nhật tiến độ: Frontend tự động gửi request cập nhật tiến độ mỗi 30 giây.
       Backend thực hiện Server-side Validation: Lấy thời gian hiện tại trừ thời gian cập nhật cuối cùng trong Redis Hash.
       Nếu khoảng tiến độ tăng thêm hợp lệ (<= thời gian trôi qua thực tế + trễ mạng) → lưu tạm vào Redis Hash.
       Cuối phiên học hoặc chuyển bài → lưu dồn (flush) dữ liệu từ Redis Hash xuống bảng lesson_progress trong PostgreSQL.
→ Học viên học hết 100% video và vượt qua các bài tập Quiz 
→ Đẩy sự kiện qua Kafka để tiến hành sinh file Certificate (PDF) ngầm và gửi email chúc mừng qua Spring Mail.
```

### Luồng 3 — Giảng viên tạo khóa học & Tải video trực tiếp (Direct Upload R2)

```
Giảng viên khởi tạo Khóa học (Trạng thái: DRAFT) → Thêm cấu trúc Chương (Module) → Thêm Bài học (Lesson):
- Nhánh dán link YouTube: Giảng viên dán URL → Frontend trích xuất video_id → Backend lưu.
- Nhánh upload video Cloudflare R2 (Direct Upload):
    Giảng viên chọn file video từ máy → Frontend gọi API xin link upload 
    → Backend dùng S3 SDK sinh Presigned PUT URL và gửi về Frontend 
    → Frontend gửi file video trực tiếp lên Cloudflare R2 qua Presigned PUT URL (không đi qua Backend)
    → Upload thành công, Frontend gọi API lưu bài học kèm theo video_id (tên file trên R2) → Backend lưu thông tin vào DB.
→ Giảng viên bấm "Yêu cầu xuất bản" → Trạng thái chuyển sang PENDING_APPROVAL.
→ Admin duyệt → Trạng thái chuyển sang PUBLISHED (Hiển thị công khai trên sàn).
```

---

## 5. Kiến trúc kỹ thuật (Tech Stack)

### Backend

- **Runtime:** Java (Spring Boot 3.x) + Spring Security (JWT) + Spring Data JPA + Stripe Java SDK
- **Database:** PostgreSQL
- **Cache:** Redis — Quản lý giỏ hàng tạm (TTL 48h sliding expiration), cache đề cương/chi tiết khóa học, cache tạm tiến độ học tập (Redis Hash) và rate limiting.
- **Message Queue:** Kafka — xử lý event bất đồng bộ (`order.paid`, `course.completed`) để gửi email và sinh chứng chỉ.
- **Email:** Spring Boot Starter Mail (JavaMailSender) + Thymeleaf.
- **Cloud Storage SDK:** AWS Java SDK (tương thích với Cloudflare R2).
- **Payment APIs:** Stripe REST API & VietQR Handler (Upload image minh chứng).

### Frontend

- **Framework:** ReactJS (Vite) + Tailwind CSS + Shadcn/ui.
- **Video player:** ReactPlayer hoặc Video.js kết hợp bộ điều khiển custom và CSS overlay bảo mật.
- **AI Integration:** Google Gemini (Free tier) gọi từ Backend.

### Infrastructure

- **Deploy:** Railway / Render (Backend) + Vercel (Frontend) + Upstash Kafka (Serverless Kafka Free tier) + Cloudflare R2.

---

## 6. Xử lý video — giải pháp quan trọng

Hệ thống sử dụng cơ chế **Hybrid Video Storage** để tối ưu hóa bảo mật và chi phí hạ tầng:

```
[Nguồn YouTube Unlisted]
- Chỉ trả Video ID qua API bảo mật có xác thực Enrollment.
- Frontend sử dụng CSS Overlay chặn các hành vi click chuột phải, click đúp hoặc tương tác trực tiếp lên video để ngăn trích xuất link YouTube gốc.

[Nguồn Cloudflare R2 Direct Upload]
- Upload: Frontend lấy Presigned PUT URL từ Backend và đẩy file trực tiếp lên R2.
- Playback: Backend sinh Presigned GET URL động có thời gian sống 10-15 phút để Frontend phát video, ngăn chặn triệt để việc chia sẻ link video ra bên ngoài.
```

---

## 7. Database — Các bảng chính

| Bảng                     | Mô tả                                 |
| ------------------------ | ------------------------------------- |
| `users`                  | Tài khoản học viên, giảng viên, admin, staff (Có thêm vai trò Staff) |
| `instructor_profiles`    | Hồ sơ giảng viên, lưu số dư `balance` |
| `categories`             | Danh mục khóa học                     |
| `courses`                | Khóa học                              |
| `modules`                | Chương học                            |
| `lessons`                | Bài học, chứa video_source (`youtube`, `vimeo`, `r2`) + video_id |
| `lesson_comments`        | Bình luận, thảo luận (Q&A) dưới bài học của học viên |
| `orders` / `order_items` | Đơn hàng mua khóa học, liên kết `voucher_id` trực tiếp, lưu `transaction_method` (VietQR/Stripe) và `transaction_proof_url` (bill VietQR) |
| `vouchers`               | Mã giảm giá (Phân biệt voucher của admin và voucher của instructor) |
| `enrollments`            | Ghi danh sau khi mua thành công       |
| `lesson_progress`        | Tiến độ xem từng bài học              |
| `course_reviews`         | Đánh giá khóa học (Chỉ cho phép tài khoản đã sở hữu khóa học) |
| `certificates`           | Chứng chỉ hoàn thành dạng PDF         |
| `withdrawal_requests`    | Quản lý yêu cầu rút tiền và trạng thái chuyển khoản thủ công |

---

## 8. Roadmap theo tuần (8 tuần)

| Tuần | Việc cần làm                                                       |
| ---- | ------------------------------------------------------------------ |
| 1    | Setup project Spring Boot & React Vite, thiết kế DB Schema PostgreSQL, cấu hình Security với JWT |
| 2    | Xây dựng API quản lý khóa học/chương/bài học, viết API ký Presigned URL upload video lên Cloudflare R2 |
| 3    | Xây dựng Cart Service sử dụng Redis (Sliding Expiration 48h), Voucher Engine |
| 4    | Tích hợp cổng thanh toán VietQR Webhook, xác thực chữ ký số Webhook, xử lý khóa dòng (SELECT FOR UPDATE) chống trùng lặp, mở Enrollment tự động |
| 5    | Xây dựng hệ thống cập nhật tiến độ học tập (Progress Tracking) xác thực thời gian thực qua Redis Hash |
| 6    | Cài đặt Upstash Kafka, viết Consumer sinh file Certificate PDF, gửi Email tự động và viết API quản lý ví/yêu cầu rút tiền cho Giảng viên |
| 7    | Phát triển Frontend: Giao diện học tập tích hợp Trợ lý học tập Gemini AI Chatbot, bọc bảo mật player |
| 8    | Viết Integration Tests, sửa lỗi hiệu năng, deploy hoàn chỉnh lên Vercel + Railway |

---

## 9. Điểm mạnh khi trình bày với nhà tuyển dụng

- **Cơ chế chống gian lận tiến độ học tập (Zero Trust Progress Tracking):** Kết hợp Redis Hash làm write-back cache và Server-side validation để tính toán thời gian học thực tế, tối ưu hóa I/O cho cơ sở dữ liệu chính.
- **Xử lý tài chính an toàn (Idempotency):** Áp dụng xác thực chữ ký số webhook ngân hàng và cơ chế khóa dòng database (`SELECT ... FOR UPDATE`) để đảm bảo các giao dịch tài chính thông qua VietQR chỉ được xử lý duy nhất một lần.
- **Hạ tầng lưu trữ video thông minh (Direct Upload R2):** Triển khai luồng ký Presigned URL để upload trực tiếp từ Frontend lên R2 giúp giảm 99% tải cho Backend. Bảo mật video tối đa bằng link R2 động hết hạn sau 15 phút.
- **Tích hợp AI Trợ giảng bằng Gemini API:** Đem lại tính năng thực tế cho học viên bằng cách sử dụng Generative AI tương tác trực tiếp theo ngữ cảnh bài học.
- **Kiến trúc hướng sự kiện (Event-Driven với Kafka):** Tách biệt luồng chính (Thanh toán) khỏi các luồng phụ nặng nề (Gửi mail hóa đơn, Render PDF chứng chỉ) giúp tăng khả năng phản hồi của hệ thống.

---

_Tài liệu được cập nhật ngày 23/06/2026_
