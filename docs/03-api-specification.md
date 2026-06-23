# API Specification — Nền tảng học trực tuyến (Udemy Mini)

> Base URL: `/api/v1`
> Authentication: Bearer Token (JWT) trong header `Authorization: Bearer <token>`

---

## 1. Authentication

### POST `/auth/register`
Đăng ký tài khoản học viên.

**Request:**
```json
{
  "email": "student@example.com",
  "password": "secret123",
  "full_name": "Nguyễn Văn A"
}
```

**Response 201:**
```json
{
  "user": { "id": "uuid", "email": "...", "role": "student" },
  "access_token": "jwt...",
  "refresh_token": "jwt..."
}
```

---

### POST `/auth/login`
**Request:** `{ "email": "...", "password": "..." }`
**Response 200:** giống register

---

### POST `/auth/google`
Đăng nhập bằng Google OAuth.

**Request:** `{ "id_token": "google-id-token..." }`
**Response 200:** giống register

---

### POST `/auth/refresh`
**Request:** `{ "refresh_token": "..." }`
**Response 200:** `{ "access_token": "jwt..." }`

---

## 2. Courses & Curriculum (Khóa học & Bài học)

### GET `/courses`
Danh sách khóa học, có filter & search.

**Query params:** `?category=web-dev&level=beginner&search=react&page=1&limit=12`

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "React từ Zero đến Hero",
      "thumbnail_url": "...",
      "price": 499000,
      "instructor": { "full_name": "Trần B" },
      "rating_avg": 4.7,
      "total_students": 1200
    }
  ],
  "pagination": { "page": 1, "limit": 12, "total": 86 }
}
```

---

### GET `/courses/:slug`
Chi tiết 1 khóa học (trang public, đề cương không bao giờ chứa `video_id` của các bài học tính phí).

**Response 200:**
```json
{
  "id": "uuid",
  "title": "React từ Zero đến Hero",
  "description": "...",
  "price": 499000,
  "modules": [
    {
      "title": "Chương 1: Giới thiệu",
      "lessons": [
        { "id": "uuid", "title": "Bài 1: Cài đặt", "duration_seconds": 360, "is_preview": true }
      ]
    }
  ],
  "is_enrolled": false
}
```

---

### POST `/courses` 🔒 *(role: instructor)*
Tạo khóa học mới (draft).

**Request:** `{ "title": "...", "category_id": "uuid", "price": 499000, "level": "beginner" }`

---

### POST `/courses/:id/modules` 🔒 *(role: instructor)*
Thêm chương học.

---

### GET `/lessons/upload-url` 🔒 *(role: instructor)*
Giảng viên xin link upload video thẳng lên Cloudflare R2 (Direct Upload).

**Query params:** `?filename=lesson_intro.mp4`

**Response 200:**
```json
{
  "upload_url": "https://r2.cloudflare.com/my-bucket/lessons/uuid-lesson_intro.mp4?X-Amz-Signature=...",
  "video_id": "uuid-lesson_intro.mp4"
}
```

---

### POST `/modules/:id/lessons` 🔒 *(role: instructor)*
Thêm bài học mới.

**Request (nguồn R2):**
```json
{ 
  "title": "Bài 2: Sử dụng JSX", 
  "video_source": "r2", 
  "video_id": "uuid-lesson_intro.mp4", 
  "duration_seconds": 600,
  "is_preview": false 
}
```

**Request (nguồn YouTube):**
```json
{ 
  "title": "Bài 3: Props là gì", 
  "video_source": "youtube", 
  "video_id": "dQw4w9WgXcQ", 
  "duration_seconds": 300,
  "is_preview": false 
}
```

---

## 3. Cart & Voucher (Giỏ hàng)

### POST `/cart/items` 🔒
Thêm khóa học vào giỏ (lưu Redis theo `user_id`, TTL 48h sliding expiration, tự reset gia hạn khi có tương tác).

**Request:** `{ "course_id": "uuid" }`

---

### GET `/cart` 🔒
**Response 200:**
```json
{
  "items": [{ "course_id": "uuid", "title": "...", "price": 499000 }],
  "subtotal": 998000
}
```

---

### DELETE `/cart/items/:course_id` 🔒
Xóa 1 khóa học khỏi giỏ.

---

### POST `/cart/apply-voucher` 🔒
Áp voucher giảm giá cho giỏ hàng.

**Request:** `{ "code": "SUMMER20" }`

**Response 200:**
```json
{ "valid": true, "discount_amount": 199600, "total": 798400 }
```

---

## 4. Orders & Payment (Thanh toán VietQR)

### POST `/orders/checkout` 🔒
Tạo đơn hàng từ giỏ hàng hiện tại, sinh mã VietQR quét thanh toán động.

**Request:** `{ "voucher_code": "SUMMER20" }` *(optional)*

**Response 200:**
```json
{
  "order_id": "uuid",
  "total": 798400,
  "payment_method": "vietqr",
  "qr_code_url": "https://api.vietqr.io/image/970415-123456789-Q4zK9t.jpg?amount=798400&addInfo=HOCVIEN_DH123",
  "transaction_reference": "HOCVIEN_DH123"
}
```

---

### POST `/orders/vietqr-webhook` *(Webhook Ngân hàng / Cổng dịch vụ gọi về)*
Xử lý thanh toán tự động, verify chữ ký số để bảo mật tuyệt đối.

**Headers:** `X-Webhook-Signature: <hmac_sha256_signature>`

**Request Body:**
```json
{
  "transaction_reference": "HOCVIEN_DH123",
  "amount": 798400,
  "status": "PAID",
  "bank_transaction_id": "FT262356237"
}
```

**Logic Backend:**
```
1. Verify signature trong header bằng Webhook Secret để chặn request giả mạo.
2. Mở Database Transaction với khóa dòng (SELECT ... FOR UPDATE) trên bảng orders theo transaction_reference.
3. Nếu đơn hàng có trạng thái PENDING:
   - Cập nhật trạng thái đơn hàng thành PAID.
   - Cộng số dư balance cho Giảng viên tương ứng (sau khi trừ % phí sàn).
   - Tạo bản ghi Enrollment tương ứng mở quyền truy cập khóa học.
   - Xóa giỏ hàng trong Redis.
   - Bắn sự kiện "order.paid" sang Kafka để gửi hóa đơn bất đồng bộ.
4. Trả về HTTP 200 (Idempotency) để thông báo cho Webhook không gửi lại (retry) nữa.
```

---

## 5. Learning & Progress (Học bài & Bảo mật Video)

### GET `/my-courses` 🔒
Danh sách khóa học đã mua của học viên hiện tại.

---

### GET `/courses/:id/learn` 🔒 *(phải đã mua)*
Lấy danh sách đề cương đầy đủ để học. **Lưu ý:** Không trả về video_id hay link video của bài học tại đây để tránh trộm link.

**Response 200:**
```json
{
  "course_title": "...",
  "modules": [
    {
      "title": "Chương 1",
      "lessons": [
        { "id": "uuid_lesson_1", "title": "Bài 1", "duration_seconds": 360, "is_completed": false }
      ]
    }
  ]
}
```

---

### GET `/lessons/:id/video` 🔒
Lấy thông tin video bảo mật để phát trên player.

**Logic Backend:**
1. Lấy `user_id` đăng nhập trực tiếp từ Spring Security Context (JWT), kiểm tra xem học viên có sở hữu khóa học (`Enrollment`) chứa bài học này không, hoặc bài học có `is_preview = true` không.
2. Nếu hợp lệ:
   - Nếu `video_source == 'r2'`: Sinh **Presigned GET URL** từ Cloudflare R2 (hết hạn sau 15 phút). Trả về `{ "video_source": "r2", "video_url": "https://..." }`.
   - Nếu `video_source == 'youtube'`: Trả về `{ "video_source": "youtube", "video_id": "abc123" }`.
3. Nếu không có quyền: Trả về lỗi `403 Forbidden`.

---

### POST `/lessons/:id/progress` 🔒
Cập nhật tiến độ xem định kỳ (gọi mỗi 30s từ Frontend).

**Request:** `{ "watched_seconds": 90, "is_completed": false }`

**Logic Backend:**
1. Lấy `user_id` đăng nhập từ Spring Security.
2. Đọc tiến độ học gần nhất và `last_updated_at` trong **Redis Hash** của học viên.
3. Validate: `watched_seconds - cached_seconds <= (current_time - last_updated_at) + 3s`.
4. Nếu hợp lệ: Cập nhật Redis Hash với giá trị mới và đặt `last_updated_at = current_time`.
5. Khi kết thúc phiên học, chuyển bài, hoặc nếu `is_completed = true` -> Thực hiện ghi dồn xuống PostgreSQL (`lesson_progress`).
6. Nếu tiến độ đạt >= 90% -> Tự động đánh dấu hoàn thành bài học, kiểm tra nếu hoàn thành toàn bộ khóa học -> Đẩy sự kiện Kafka `course.completed` sinh chứng chỉ.

---

### POST `/courses/:id/reviews` 🔒 *(chỉ dành cho học viên đã mua khóa học)*
Đánh giá khóa học.

---

## 6. AI Chatbot Trợ giảng

### POST `/courses/:id/ai-chat` 🔒 *(phải đã mua)*
Gửi câu hỏi cho AI trợ giảng ngay trong trang học.

**Request:**
```json
{
  "lesson_id": "uuid_lesson_1",
  "message": "Em chưa hiểu đoạn định nghĩa JSX ở trên, giải thích lại giúp em."
}
```

**Response 200:**
```json
{
  "reply": "JSX viết tắt của JavaScript XML. Nó cho phép chúng ta viết mã HTML trong React..."
}
```

**Logic Backend:** Backend lấy context bài học (`course_title`, `lesson_title`, `lesson_description`) làm system prompt ghép với câu hỏi của học viên, gọi tới **Gemini 1.5 Flash API** để sinh câu trả lời thông minh.

---

## 7. Certificates (Chứng chỉ)

### GET `/certificates/:code/verify`
Kiểm tra tính hợp lệ của chứng chỉ (Public).

---

## 8. Instructor & Wallet (Giảng viên & Rút tiền)

### GET `/instructor/revenue` 🔒 *(role: instructor)*
Thống kê doanh thu và số dư khả dụng (`balance`) hiện tại.

**Response 200:**
```json
{
  "balance": 15400000.00,
  "total_earned": 28000000.00,
  "total_withdrawn": 12600000.00
}
```

---

### POST `/instructor/withdrawals` 🔒 *(role: instructor)*
Yêu cầu rút tiền về tài khoản ngân hàng.

**Request:**
```json
{
  "amount": 5000000.00,
  "bank_name": "Vietcombank",
  "account_number": "1012345678",
  "account_holder": "TRAN VAN B"
}
```

**Logic Backend:** Kiểm tra số dư `balance` hiện tại của Giảng viên. Nếu đủ số tiền rút, tạo yêu cầu rút tiền với trạng thái `PENDING` và khóa tạm thời số tiền này lại (hoặc trừ thẳng số dư).

---

### GET `/instructor/withdrawals` 🔒 *(role: instructor)*
Xem danh sách lịch sử yêu cầu rút tiền.

---

## 9. Admin Operations (Duyệt khóa học & Duyệt rút tiền) 🔒 *(role: admin)*

### POST `/admin/courses/:id/approve`
Phê duyệt khóa học lên sàn. Khóa học chuyển trạng thái thành `PUBLISHED`.

---

### POST `/admin/courses/:id/reject`
Từ chối duyệt khóa học. Khóa học về trạng thái `REJECTED` kèm cột lưu lý do từ chối gửi về giảng viên.

**Request:** `{ "reject_reason": "Video bài học số 3 bị lỗi tiếng." }`

---

### POST `/admin/withdrawals/:id/approve`
Admin xác nhận đã chuyển khoản ngân hàng thủ công thành công bên ngoài và duyệt yêu cầu.

**Logic Backend:** Trạng thái yêu cầu rút tiền chuyển từ `PENDING` sang `APPROVED`. Cập nhật số dư `balance` chính thức của giảng viên (nếu chưa trừ lúc tạo yêu cầu).

---

### POST `/admin/withdrawals/:id/reject`
Admin từ chối yêu cầu rút tiền do sai thông tin hoặc có khiếu nại vi phạm.

**Request:** `{ "reject_reason": "Tên tài khoản không khớp với tên giảng viên." }`

**Logic Backend:** Trạng thái chuyển thành `REJECTED`. Hoàn trả lại số tiền về số dư `balance` khả dụng cho Giảng viên.
