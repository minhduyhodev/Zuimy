# Kiến trúc Hệ thống — Nền tảng học trực tuyến (Zuimy)

> Tài liệu mô tả cấu trúc, hạ tầng triển khai, các sơ đồ tuần tự (Sequence Diagram) chi tiết cho các luồng nghiệp vụ lõi và các giải pháp kỹ thuật đặc biệt của hệ thống Zuimy.

---

## 1. Kiến trúc Tổng thể (High-Level Architecture)

Hệ thống Zuimy được thiết kế theo mô hình Monolith (Backend) kết hợp hướng sự kiện (Event-Driven Architecture) thông qua Message Queue (Kafka) để xử lý các tiến trình nặng bất đồng bộ.

### Sơ đồ Kiến trúc Tổng thể

```
                              ┌──────────────────────────────────┐
                              │        TẦNG CLIENT (REACT)       │
                              │  - Custom Player (CSS Overlay)   │
                              └────────────────┬─────────────────┘
                                               │
                                               │ (HTTPS + JWT)
                                               ▼
                              ┌──────────────────────────────────┐
                              │ TẦNG BẢO MẬT: SPRING SECURITY    │
                              │  - JWT Verification Filter       │
                              └────────────────┬─────────────────┘
                                               │
                                               ▼
    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                       TẦNG BACKEND (SPRING BOOT MONOLITH)                     │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
    │  │ AuthSvc  │  │CourseSvc │  │ CartSvc  │  │ OrderSvc │  │  ProgressSvc   │  │
    │  └──────────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬────────┘  │
    │  ┌──────────┐       │             │             │                │           │
    │  │  AISvc   │       │             │             │                │           │
    │  └────┬─────┘       │             │             │                │           │
    │  ┌────┴─────┐       │             │             │                │           │
    │  │ CertSvc  │       │             │             │                │           │
    │  └────┬─────┘       │             │             │                │           │
    └───────┼─────────────┼─────────────┼─────────────┼────────────────┼───────────┘
            │             │             │             │                │
            │             │             │(Cart Cache) │(Cache/Progress)│
            │             │             └────────┐    │                │
            │             │                      ▼    ▼                ▼
            │             │               ┌──────────────────────────────┐
            │             │               │   TẦNG CACHING (REDIS)       │
            │             │               └──────────────────────────────┘
            │             │
            │             │               ┌──────────────────────────────┐
            │             │               │   MESSAGE BROKER (KAFKA)     │
            │             │               │   - order.paid               │
            │             │               │   - course.completed         │
            │             │               └──────────────┬───────────────┘
            │             │                              ▲
            │             │                              │ (Publish Event)
            │             ├──────────────────────────────┘
            │             │
            │             │               ┌──────────────────────────────┐
            │             │               │      THIRD-PARTY SERVICES    │
            │             │               │  - Stripe API                │
            │             │               │  - Google Gemini API         │
            │             │               │  - Spring Mail (SMTP)        │
            │             │               └──────────────────────────────┘
            │             │
            ▼             ▼
    ┌───────────────────────────────┐     ┌──────────────────────────────┐
    │     CLOUDFLARE R2 STORAGE     │     │    DATABASE: POSTGRESQL      │
    │  - Videos (.mp4)              │     │  - Core tables (UUID PK)     │
    │  - Certificate PDFs           │     │                              │
    └───────────────────────────────┘     └──────────────────────────────┘
```

### Mô tả chi tiết các phân tầng trong kiến trúc:

*   **Tầng Client (Frontend):** Xây dựng bằng ReactJS kết hợp Vite để tối ưu hóa tốc độ build. Sử dụng TailwindCSS và Shadcn/ui để xây dựng giao diện người dùng. Đặc biệt, bộ phát video (`Custom ReactPlayer`) được tích hợp các lớp bảo vệ CSS (CSS Overlay) để chặn các hành vi click chuột phải hoặc double-click từ học viên nhằm giấu đường dẫn video gốc.
*   **Tầng Bảo mật & Xác thực (Security Layer):** Tích hợp bộ lọc bảo mật `Spring Security Filter Chain`. Tất cả các request gửi từ Client đến API ngoại trừ các route công khai đều phải đi qua bộ lọc này để xác thực mã định danh JWT (Access Token).
*   **Tầng Backend (Spring Boot Monolith):** Chứa toàn bộ logic nghiệp vụ cốt lõi của hệ thống, được module hóa rõ ràng thành các Service riêng biệt:
    *   *Auth Service:* Quản lý đăng ký, đăng nhập bằng tài khoản nội bộ và xác thực Google OAuth 2.0.
    *   *Course & Curriculum Service:* Quản lý thông tin khóa học, chương học, bài học và cấu hình link video.
    *   *Cart & Voucher Service:* Xử lý logic giỏ hàng và áp dụng mã giảm giá.
    *   *Order & Payout Service:* Xử lý đơn hàng, tích hợp Stripe, kiểm soát ví tiền giảng viên và duyệt yêu cầu rút tiền.
    *   *Progress & Quiz Service:* Theo dõi tiến độ học tập và quản lý các bài kiểm tra trắc nghiệm.
    *   *Gemini AI Service:* Tích hợp trí tuệ nhân tạo Gemini làm trợ giảng cho học viên.
    *   *Certificate & PDF Service:* Tạo chứng chỉ hoàn thành khóa học dưới định dạng PDF.
*   **Tầng Cache & Message Broker:**
    *   *Redis Cache:* Quản lý giỏ hàng tạm của người dùng (TTL 48h tự động reset khi có hoạt động), cache thông tin bài học và lưu trữ tạm thời tiến độ học tập của học viên để tối ưu I/O.
    *   *Upstash Kafka:* Cung cấp message broker để xử lý giao tiếp bất đồng bộ, tách rời các tiến trình nặng như gửi email hóa đơn và render PDF chứng chỉ ra khỏi luồng xử lý chính.
*   **Tầng Lưu trữ (Database & Storage):**
    *   *PostgreSQL:* Cơ sở dữ liệu quan hệ lưu trữ dữ liệu có cấu trúc. Toàn bộ thực thể sử dụng định danh UUID để bảo mật đường dẫn API.
    *   *Cloudflare R2:* Lưu trữ các tệp lớn (Video bài học của giảng viên và tệp PDF chứng chỉ do hệ thống tự sinh).
*   **Dịch vụ bên thứ ba (Third-party Services):** Tích hợp Stripe Checkout để thanh toán tự động, Gemini API hỗ trợ AI chatbot và Mail Server gửi thông tin qua thư điện tử.

---

## 2. Sơ đồ Triển khai & Hạ tầng (Deployment & Infrastructure)

Toàn bộ hệ thống được triển khai trên các nền tảng đám mây hiện đại, tối ưu hóa chi phí hạ tầng (nhờ gói miễn phí/Serverless) và tăng tốc độ phân phối dữ liệu qua CDN.

### Sơ đồ Triển khai hạ tầng

```
    ┌──────────────┐          ┌───────────────────────┐
    │   Học viên   │─────────►│  Frontend Host: VERCEL│
    │ / Giảng viên │          │  - Edge Network CDN   │
    └──────┬───────┘          └───────────────────────┘
           │
           │ (REST API Requests)
           ▼
    ┌─────────────────────────┐          ┌─────────────────────────┐
    │   Backend Host: RAILWAY │─────────►│ Database Host: SUPABASE │
    │   - Spring Boot Monolith│          │ - PostgreSQL            │
    └──────┬────────────┬─────┘          └─────────────────────────┘
           │            │
           │            │ (Kafka Event Protocol)
           │            ▼
           │     ┌────────────────────────────────┐
           │     │ Message Broker: UPSTASH KAFKA  │
           │     └────────────────────────────────┘
           │
           │ (Redis Protocol)
           ▼
    ┌─────────────────────────┐          ┌─────────────────────────┐
    │  Cache Host: UPSTASH    │          │ Asset Storage: CF R2    │
    │  - Serverless Redis     │          │ - Videos & PDFs         │
    └─────────────────────────┘          └─────────────────────────┘
```

### Giải thích hạ tầng triển khai:
1.  **Frontend (ReactJS):** Triển khai trực tiếp lên **Vercel** để tận dụng hạ tầng Edge Network toàn cầu của họ, đảm bảo giao diện tải cực nhanh cho người dùng cuối.
2.  **Backend (Spring Boot):** Hosting trên **Railway** (hoặc Render) trong môi trường Docker Container hóa. Nền tảng tự động cấu hình CI/CD khi có thay đổi code mới.
3.  **PostgreSQL Database:** Lưu trữ trên **Supabase** (hoặc Neon DB) cung cấp cơ sở dữ liệu Postgres trên nền cloud ổn định. Giao tiếp qua kết nối bảo mật SSL.
4.  **Redis & Kafka (Serverless):** Sử dụng dịch vụ của **Upstash** để không cần tự quản lý (self-host) cụm Redis và Kafka phức tạp. Cơ chế thanh toán theo lượng truy cập thực tế (Pay-as-you-go) giúp tối ưu hóa chi phí vận hành ở giai đoạn đầu của sản phẩm.
5.  **Asset Storage:** **Cloudflare R2** được chọn vì R2 không thu phí băng thông tải ra (Egress Cost = 0). Khi học viên xem video hoặc tải chứng chỉ, Cloudflare CDN sẽ phân phối dữ liệu từ vùng lưu trữ gần nhất của R2 giúp video phát mượt mà không bị giật lag.

---

## 3. Các Sơ đồ Tuần tự (Sequence Diagram) Luồng Nghiệp vụ Lõi

### Luồng 3.1: Thanh toán Stripe tự động qua Webhook
*Luồng này đảm bảo tự động hóa hoàn toàn cho thẻ quốc tế Visa/Mastercard với cơ chế Idempotent chống xử lý trùng.*

#### Sơ đồ Tuần tự bằng ký tự

```
Học viên              Frontend (React)           Backend (Spring Boot)           Stripe Gateway
   │                         │                              │                          │
   │─── 1. Click Checkout ──►│                              │                          │
   │                         │─── 2. POST /orders/checkout ─►                          │
   │                         │    (method: stripe)          │                          │
   │                         │                              │─── 3. Tạo Session ──────►│
   │                         │                              │◄── 4. Trả về Session ────│
   │                         │                              │                          │
   │                         │◄── 5. Trả về checkout_url ───│                          │
   │◄── 6. Chuyển hướng ─────│                              │                          │
   │                                                                                   │
   │─────────────────────── 7. Điền thông tin thẻ & Thanh toán thành công ────────────►│
   │◄────────────────────── 8. Redirect về trang Success của Frontend ─────────────────│
   │                                                                                   │
   │                                                        │◄── 9. POST Webhook ──────│
   │                                                        │    (payment_intent)      │
   │                                                        │                          │
   │                                                        │─── 10. Xác thực chữ ký ──│
   │                                                        │─── 11. SELECT FOR UPDATE │
   │                                                        │─── 12. PAID & Enrollment │
   │                                                        │─── 13. Publish Kafka ────│
   │                                                        │                          │
   │                                                        │─── 14. Trả về HTTP 200 ─►│
```

#### Giải thích chi tiết từng bước (Step-by-step):
1.  **Học viên** bấm thanh toán bằng Stripe trên giao diện Frontend.
2.  **Frontend** gửi request API tạo đơn hàng tới Backend.
3.  **Backend** truy xuất các khóa học trong giỏ hàng hiện tại, kiểm tra và áp dụng voucher nếu có.
4.  **Backend** gọi tới Stripe API để khởi tạo một phiên thanh toán trực tuyến (Checkout Session).
5.  **Stripe** xác nhận thông tin đơn hàng và trả về đường dẫn thanh toán tạm thời (`checkout_url`).
6.  **Backend** tạo một bản ghi đơn hàng mới trong cơ sở dữ liệu PostgreSQL ở trạng thái `PENDING` (chờ thanh toán).
7.  **Backend** gửi trả đường dẫn thanh toán về cho Client.
8.  **Frontend** tự động chuyển hướng học viên tới trang bảo mật của Stripe.
9.  **Học viên** điền thông tin thẻ tín dụng và hoàn thành giao dịch trên Stripe.
10. **Stripe** xác nhận thanh toán thành công, hiển thị nút quay lại và chuyển hướng học viên về trang Success trên Frontend.
11. Đồng thời, **Stripe** gửi một yêu cầu POST chứa dữ liệu giao dịch (Webhook) về Backend thông qua route `/api/v1/orders/stripe-webhook`.
12. **Backend** sử dụng thư viện bảo mật để xác thực chữ ký số được Stripe gửi kèm trong Header (`Stripe-Signature`) nhằm chắc chắn dữ liệu không bị sửa đổi.
13. **Backend** khởi chạy một Database Transaction, thực hiện truy vấn khóa dòng (`SELECT ... FOR UPDATE`) đối với đơn hàng tương ứng.
14. Nếu đơn hàng vẫn ở trạng thái `PENDING`:
    *   Cập nhật trạng thái đơn hàng sang `PAID` và ghi nhận thời gian thanh toán.
    *   Tạo bản ghi `Enrollment` (Ghi danh học viên vào khóa học).
    *   Tính toán chia sẻ doanh thu và cộng tiền trực tiếp vào ví `balance` của Giảng viên sở hữu khóa học.
    *   Đẩy một sự kiện (Event) có tên `order.paid` vào Upstash Kafka.
    *   Gửi phản hồi HTTP 200 về Stripe để xác nhận đã xử lý xong.
15. Nếu đơn hàng đã chuyển sang `PAID` từ trước (do webhook bị gửi lặp lại): Hệ thống ngay lập tức trả về HTTP 200, bỏ qua toàn bộ xử lý nghiệp vụ để bảo vệ tính nhất quán của số dư ví giảng viên (Tính chất Idempotency).
16. Ở phía hạ tầng bất đồng bộ, Kafka Consumer nhận sự kiện `order.paid`:
    *   Thực hiện xóa sạch giỏ hàng của học viên trong Redis.
    *   Gọi SMTP Server gửi email chúc mừng và hóa đơn điện tử cho học viên.

---

### Luồng 3.2: Thanh toán VietQR & Đối soát thủ công bởi Staff
*Luồng này dành cho việc chuyển khoản ngân hàng nội địa tại Việt Nam, giảm thiểu chi phí tích hợp cổng thanh toán.*

#### Sơ đồ Tuần tự bằng ký tự

```
Học viên             Frontend (React)            Backend (Spring Boot)              Staff / R2
   │                        │                              │                            │
   │─── 1. Chọn VietQR ────►│                              │                            │
   │                        │─── 2. POST /orders/checkout ─►                            │
   │                        │    (method: vietqr)          │                            │
   │                        │                              │─── 3. Tạo Order PENDING ──│
   │                        │◄── 4. Trả về mã QR động ─────│                            │
   │◄── 5. Quét QR & CK ────│                              │                            │
   │─── 6. Tải ảnh Bill ───►│                              │                            │
   │                        │─── 7. Upload Proof (file) ──►│                            │
   │                        │                              │─── 8. Lưu ảnh lên R2 ─────►│ (R2 Storage)
   │                        │                              │◄── 9. Trả về URL ảnh ─────│
   │                        │                              │─── 10. Đổi sang SUBMITTED_│
   │                        │◄── 11. Báo trạng thái chờ ───│                            │
   │                                                                                    │
   │                                                       │◄── 12. GET pending-proof ──│ (Staff)
   │                                                       │─── 13. Trả về ds & bill ──►│
   │                                                       │                            │
   │                                                       │◄── 14. Duyệt / Từ chối ────│ (Staff)
   │                                                       │    (SELECT FOR UPDATE)     │
   │                        │◄── 15. Đơn hàng thành PAID ──│                            │
```

#### Giải thích chi tiết từng bước (Step-by-step):
1.  **Học viên** chọn VietQR và nhấn nút thanh toán trên Frontend.
2.  **Frontend** gửi yêu cầu tạo đơn hàng thanh toán VietQR lên Backend.
3.  **Backend** khởi tạo đơn hàng ở trạng thái `PENDING`, tự động sinh mã nội dung chuyển khoản ngẫu nhiên và duy nhất (`transaction_reference`).
4.  **Backend** tạo link ảnh QR động dựa trên thông số đơn hàng (thông qua dịch vụ VietQR API) và trả thông tin về Frontend.
5.  **Frontend** hiển thị mã QR chuyển khoản và thông tin tài khoản đích cho học viên.
6.  **Học viên** dùng ứng dụng ngân hàng quét mã, chuyển tiền và chụp lại màn hình hóa đơn thành công (Bill).
7.  **Học viên** tải ảnh chụp bill lên hệ thống thông qua giao diện Frontend.
8.  **Frontend** gửi tệp ảnh dạng Multipart File lên API của Backend.
9.  **Backend** thực hiện upload tệp ảnh này trực tiếp lên bộ lưu trữ Cloudflare R2 tại thư mục `/proofs`.
10. **R2** lưu trữ và gửi trả link truy cập ảnh công khai (`transaction_proof_url`) về Backend.
11. **Backend** cập nhật đơn hàng thành trạng thái `SUBMITTED_PROOF` (đã nộp minh chứng) và lưu đường dẫn ảnh bill vào DB.
12. **Backend** báo tin thành công về cho học viên và hiển thị trạng thái "Đang chờ đối soát".
13. **Nhân viên vận hành (Staff)** đăng nhập vào trang quản trị để xem danh sách đơn hàng đang chờ duyệt.
14. **Frontend quản trị** gọi API lấy danh sách các đơn hàng ở trạng thái `SUBMITTED_PROOF`.
15. **Backend** truy vấn dữ liệu đơn hàng và link ảnh bill từ Postgres rồi trả về.
16. **Staff** kiểm tra tài khoản ngân hàng thực tế để xem đã nhận được số tiền khớp với mã đơn hàng chưa.
17. **Nếu thông tin chuyển khoản chính xác:**
    *   Staff nhấn nút "Phê duyệt" trên hệ thống.
    *   API gửi request cập nhật đơn hàng lên Backend. Backend mở transaction với khóa dòng (`SELECT FOR UPDATE`), chuyển trạng thái đơn hàng thành `PAID`, tự động ghi nhận học viên vào khóa học và phân chia doanh thu cho giảng viên tương tự luồng Stripe.
18. **Nếu thông tin sai lệch hoặc không nhận được tiền:**
    *   Staff nhập lý do từ chối (ví dụ: "Thiếu tiền", "Ảnh chỉnh sửa") và nhấn từ chối.
    *   Backend cập nhật đơn hàng sang trạng thái `REJECTED_PROOF`.
    *   Gửi email thông báo lý do và hướng dẫn học viên kiểm tra lại.

---

### Luồng 3.3: Học bài & Bảo mật Video (Cloudflare R2 Direct Upload & Presigned URL)
*Luồng này đảm bảo giấu hoàn toàn link video gốc và hạn chế chia sẻ tài khoản xem lậu.*

#### Sơ đồ Tuần tự bằng ký tự

```
Giảng viên / Học viên         Frontend (React)           Backend (Spring Boot)          Cloudflare R2
   │                                 │                             │                          │
   │ [Upload Video]                  │                             │                          │
   │─── 1. Chọn file video ─────────►│                             │                          │
   │                                 │─── 2. GET upload-url ──────►│                          │
   │                                 │                             │─── 3. Sinh Presigned ───►│
   │                                 │                             │◄── 4. Trả về PUT URL ────│
   │                                 │◄── 5. Trả về upload_url ────│                          │
   │                                 │                                                        │
   │                                 │─── 6. Upload file trực tiếp lên R2 ───────────────────►│
   │                                 │                                                        │
   │                                 │─── 7. POST /lessons ───────►│                          │
   │                                 │    (Lưu thông tin & video_id)                          │
   │                                 │◄── 8. Trả về HTTP 201 ──────│                          │
   │                                                                                          │
   │ [Play Video]                                                                             │
   │─── 9. Chọn xem bài học ────────►│                             │                          │
   │                                 │─── 10. GET /lessons/video ─►│                          │
   │                                 │    (Check JWT & Enrollment) │                          │
   │                                 │                             │─── 11. Sinh Presigned ──►│
   │                                 │                             │◄── 12. Trả về GET URL ───│
   │                                 │◄── 13. Trả về video_url ────│                          │
   │◄── 14. Phát trên ReactPlayer ───│                                                        │
```

#### Giải thích chi tiết từng bước (Step-by-step):

**Luồng tải lên video bài học (Instructor):**
1.  **Giảng viên** chọn file video (.mp4) từ máy tính để tải lên bài học.
2.  **Frontend** không tự upload file lên Backend để tránh làm sập băng thông server. Nó gọi API xin Backend cấp quyền tải lên R2 trực tiếp.
3.  **Backend** sử dụng thư viện AWS S3 SDK kết nối với Cloudflare R2 để tạo một đường dẫn đăng ký ghi tạm thời (`Presigned PUT URL`).
4.  **R2** trả về URL đăng ký có kèm chữ ký số và thời hạn hiệu lực ngắn (15 phút).
5.  **Backend** trả về link này và tên file ngẫu nhiên bảo mật (`video_id`) cho Frontend.
6.  **Frontend** gửi file video trực tiếp từ trình duyệt lên Cloudflare R2 thông qua link `Presigned PUT URL` bằng giao thức HTTP PUT.
7.  Sau khi R2 báo upload thành công, **Frontend** gửi thông tin bài học kèm theo `video_id` (tên file trên R2) lên Backend để lưu trữ.
8.  **Backend** lưu thông tin bài học và ghi nhận video nguồn R2 vào PostgreSQL.
9.  Backend gửi trả mã HTTP 201 báo đã tạo bài học thành công.

**Luồng phát video bảo mật (Student):**
1.  **Học viên** nhấn vào bài học trong chương trình.
2.  **Frontend** gửi request API xin link phát video của bài học đó.
3.  **Backend** tự động giải mã JWT từ context bảo mật để lấy `user_id` hiện tại.
4.  **Backend** truy vấn PostgreSQL để kiểm tra quyền truy cập: Học viên phải sở hữu đơn hàng thành công của khóa học chứa bài học này, hoặc bài học đó phải được cấu hình là xem thử miễn phí (`is_preview = true`).
5.  **Nếu hợp lệ:**
    *   Backend gọi Cloudflare R2 sinh đường dẫn đọc tạm thời (`Presigned GET URL`) hết hạn sau 10 đến 15 phút.
    *   Backend trả về link động này cho Frontend.
    *   Frontend nạp link vào Custom Player và phát video. Link này không thể chia sẻ ra ngoài vì sẽ hết hạn sau vài phút.
6.  **Nếu không hợp lệ:** Backend trả về lỗi 403 Forbidden, từ chối cấp link video.

---

### Luồng 3.4: Ghi nhận tiến độ học chống gian lận (Redis Hash Write-Back)
*Cơ chế này sử dụng Redis Hash làm bộ nhớ đệm ghi nhận nhanh tiến độ mỗi 30 giây để tối ưu tài nguyên DB chính.*

#### Sơ đồ Tuần tự bằng ký tự

```
Học viên                   Frontend (React)             Backend (Spring Boot)           Redis / DB
   │                              │                              │                          │
   │ [Đang học bài]               │                              │                          │
   │─── 1. Xem video ────────────►│                              │                          │
   │                              │ (Lặp lại mỗi 30s)            │                          │
   │                              │─── 2. POST progress ────────►│                          │
   │                              │    (watched_seconds)         │─── 3. Đọc cache Redis ──►│
   │                              │                              │◄── 4. Trả về cache ──────│
   │                              │                              │─── 5. Validate trên SV   │
   │                              │                              │─── 6. Lưu cache mới ────►│ (Redis)
   │                              │◄── 7. Trả về HTTP 200 ───────│                          │
   │                                                                                        │
   │ [Chuyển bài hoặc thoát]                                                                │
   │─── 8. Thoát / Hoàn thành ───►│                              │                          │
   │                              │─── 9. POST progress ────────►│                          │
   │                              │    (is_completed = true)     │─── 10. Đọc cache Redis ─►│
   │                              │                              │─── 11. Ghi dồn (Flush) ─►│ (PostgreSQL)
   │                              │                              │─── 12. Xóa cache Redis ─►│
   │                              │◄── 13. Trả về HTTP 200 ──────│                          │
```

#### Giải thích chi tiết từng bước (Step-by-step):
1.  Trong khi xem video, **Frontend** được cấu hình tự động gọi API cập nhật tiến độ mỗi 30 giây.
2.  **Backend** tiếp nhận request, lấy `user_id` từ JWT và tìm kiếm cache tiến độ của bài học này trong **Redis Hash** (Key: `progress-cache:{user_id}:{lesson_id}`).
3.  **Nếu đây là giây học đầu tiên (chưa có cache Redis):** Backend khởi tạo một bản ghi Hash mới lưu `watched_seconds` và ghi nhận timestamp hiện tại (`last_updated_at`). Trả về HTTP 200.
4.  **Nếu đã có dữ liệu cache từ trước:**
    *   Backend lấy số giây đã xem trước đó và timestamp của lần cập nhật trước.
    *   Hệ thống thực hiện so sánh chênh lệch: Số giây học tăng thêm thực tế (`watched_seconds - cached_seconds`) không được lớn hơn thời gian trôi qua thực tế trên đồng hồ server (`current_time - last_updated_at`). Có cộng thêm sai số 3 giây để tránh trễ mạng.
    *   **Nếu chênh lệch hợp lệ:** Hệ thống cập nhật số giây mới và gán `last_updated_at = current_time` vào Redis Hash. Trả về HTTP 200.
    *   **Nếu chênh lệch không hợp lệ (Ví dụ: 30 giây trước học viên ở giây thứ 10, nhưng 30 giây sau gửi request báo đã xem đến giây thứ 500):** Backend lập tức từ chối ghi nhận, trả về lỗi 400 và ghi log cảnh báo hành vi gian lận.
5.  **Khi học viên chuyển bài học khác hoặc đóng tab học tập:** Frontend gửi request cập nhật cuối cùng kèm tham số `is_completed = true`.
6.  **Backend** lấy toàn bộ dữ liệu tiến độ đã lưu trong Redis Hash, thực hiện ghi dồn (Flush) một lần duy nhất vào cơ sở dữ liệu PostgreSQL.
7.  Sau khi ghi thành công xuống Postgres, **Backend** xóa sạch bản ghi tạm thời này trong Redis Hash để giải phóng bộ nhớ.
8.  **Backend** kiểm tra xem đây có phải là bài học cuối cùng của khóa học không. Nếu toàn bộ bài học đã hoàn thành, Backend đẩy sự kiện `course.completed` vào Kafka để kích hoạt tiến trình sinh chứng chỉ và gửi mail chúc mừng bất đồng bộ.

---

## 4. Các giải pháp và Quy tắc kỹ thuật đặc biệt

### 4.1. Cơ chế Khóa dòng (Database Pessimistic Locking)
Để tránh tình trạng **Race Condition** khi nhiều học viên cùng checkout và áp dụng một mã voucher giới hạn số lượng (`max_uses`), hệ thống sử dụng khóa bi quan (`Pessimistic Lock`) trên JPA.
```java
// Ví dụ mã nguồn xử lý cập nhật số lượng voucher
@Repository
public interface VoucherRepository extends JpaRepository<Voucher, UUID> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT v FROM Voucher v WHERE v.code = :code")
    Optional<Voucher> findByCodeForUpdate(@Param("code") String code);
}
```
*Tác dụng:* Khi một luồng thanh toán đang thực hiện kiểm tra và trừ lượt dùng của voucher, cơ sở dữ liệu sẽ khóa dòng voucher đó lại. Các giao dịch Checkout đồng thời khác bắt buộc phải xếp hàng chờ giao dịch đầu tiên `COMMIT` xong mới được đọc, ngăn chặn triệt để việc vượt quá `max_uses`.

### 4.2. Cấu trúc lưu trữ Redis Hash cho Tiến độ học tập
Hệ thống sử dụng cấu trúc dữ liệu **Redis Hash** để cache tiến độ học tạm thời nhằm giảm tải I/O ghi trực tiếp xuống PostgreSQL liên tục mỗi 30s.

*   **Key định dạng:** `progress-cache:{user_id}:{lesson_id}`
*   **Các trường (Fields) bên trong Hash:**
    1.  `watched_seconds`: Số giây hiện tại học viên đã xem (Integer).
    2.  `last_updated_at`: Epoch timestamp (Mili giây) của lần cập nhật tiến độ hợp lệ cuối cùng.
*   **Thời gian sống (TTL):** Đặt TTL là `12 giờ` để dọn dẹp các cache không được ghi dồn do học viên tắt trình duyệt đột ngột.

---

_Tài liệu được cập nhật ngày 26/06/2026_
