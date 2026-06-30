# API Specification — Nền tảng học trực tuyến (Zuimy)

> Base URL: `/api/v1`  
> Authentication: Bearer Token (JWT) trong header `Authorization: Bearer <token>`

---

## Quy chuẩn API Response (API Response Standard)

Tất cả các dữ liệu trả về từ hệ thống Zuimy đều tuân theo các quy tắc chuẩn hóa sau đây:

1.  **Định dạng dữ liệu:** JSON.
2.  **Quy tắc đặt tên (Naming Convention):**
    *   Cơ sở dữ liệu lưu dưới dạng `snake_case`.
    *   Các khóa JSON trong API Request/Response sử dụng **`camelCase`** (được Spring Boot tự động chuyển đổi qua Jackson ObjectMapper).
3.  **Cơ chế lưu dấu thời gian (Timestamp):**
    *   Mỗi response luôn đi kèm trường `timestamp` ở **dưới cùng (cuối)** của đối tượng JSON, được định dạng theo chuẩn ISO 8601 UTC (`yyyy-MM-dd'T'HH:mm:ss'Z'`).
4.  **Cấu trúc Response Thành công (Success Response):**
    *   *Dữ liệu đơn lẻ / Không phân trang:*
        ```json
        {
          "success": true,
          "data": {
            "id": "uuid-value",
            "fieldName": "value"
          },
          "timestamp": "2026-06-26T12:00:00Z"
        }
        ```
    *   *Dữ liệu danh sách phân trang (Paginated List):*
        ```json
        {
          "success": true,
          "data": [
            { "id": "...", "fieldName": "..." }
          ],
          "pagination": {
            "page": 1,
            "limit": 12,
            "total": 86
          },
          "timestamp": "2026-06-26T12:00:00Z"
        }
        ```
5.  **Cấu trúc Response Lỗi (Error Response):**
    *   Trả về mã lỗi HTTP tương ứng (400, 401, 403, 404, 500).
    *   Body response theo format chuẩn sau:
        ```json
        {
          "success": false,
          "error": {
            "code": "MÃ_LỖI",
            "message": "Thông điệp mô tả lỗi chi tiết"
          },
          "timestamp": "2026-06-26T12:00:00Z"
        }
        ```
    *   *Một số mã lỗi đặc trưng:*
        *   `INVALID_CREDENTIALS`: Sai email hoặc mật khẩu.
        *   `UNAUTHORIZED`: Chưa đăng nhập hoặc token hết hạn/không hợp lệ.
        *   `FORBIDDEN`: Không có quyền truy cập (chưa sở hữu khóa học, sai phân quyền role).
        *   `COURSE_NOT_FOUND`: Khóa học không tồn tại.
        *   `VOUCHER_INVALID`: Voucher không hợp lệ hoặc đã hết lượt sử dụng.
        *   `INSUFFICIENT_BALANCE`: Ví giảng viên không đủ số dư thực hiện rút tiền.

---

## 1. Authentication (Xác thực)

### POST `/auth/register`
Đăng ký tài khoản học viên mới.

**Request:**
```json
{
  "email": "student@example.com",
  "password": "secret123",
  "fullName": "Nguyễn Văn A"
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid-student-1",
      "email": "student@example.com",
      "role": "student"
    },
    "accessToken": "jwt-access-token-string",
    "refreshToken": "jwt-refresh-token-string"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/auth/login`
Đăng nhập tài khoản.

**Request:**
```json
{
  "email": "student@example.com",
  "password": "secret123"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid-student-1",
      "email": "student@example.com",
      "role": "student"
    },
    "accessToken": "jwt-access-token-string",
    "refreshToken": "jwt-refresh-token-string"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/auth/google`
Đăng nhập thông qua tài khoản Google.

**Request:**
```json
{
  "idToken": "google-id-token-here..."
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid-student-1",
      "email": "student@example.com",
      "role": "student"
    },
    "accessToken": "jwt-access-token-string",
    "refreshToken": "jwt-refresh-token-string"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/auth/refresh`
Lấy lại Access Token mới khi hết hạn sử dụng.

**Request:**
```json
{
  "refreshToken": "jwt-refresh-token-string"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "accessToken": "jwt-new-access-token-string"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 2. Courses & Curriculum (Khóa học & Bài học)

### GET `/courses`
Danh sách các khóa học công khai, hỗ trợ tìm kiếm, lọc và phân trang.

**Query Params:** `?category=web-dev&level=beginner&search=react&page=1&limit=12`

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-course-1",
      "title": "React từ Zero đến Hero",
      "thumbnailUrl": "https://r2.cloudflare.com/thumbnails/react.jpg",
      "price": 499000,
      "instructor": {
        "fullName": "Trần B"
      },
      "ratingAvg": 4.7,
      "totalStudents": 1200
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 12,
    "total": 86
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/courses/:slug`
Chi tiết thông tin một khóa học (trang public, đề cương không bao giờ trả về `videoId` hay URL phát của video có phí).

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-course-1",
    "title": "React từ Zero đến Hero",
    "description": "Khóa học React toàn diện từ cơ bản...",
    "price": 499000,
    "modules": [
      {
        "title": "Chương 1: Giới thiệu",
        "lessons": [
          {
            "id": "uuid-lesson-1",
            "title": "Bài 1: Cài đặt môi trường",
            "durationSeconds": 360,
            "isPreview": true
          }
        ]
      }
    ],
    "isEnrolled": false
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/courses` 🔒 *(role: instructor)*
Giảng viên khởi tạo khóa học mới (ở chế độ Draft).

**Request:**
```json
{
  "title": "React từ Zero đến Hero",
  "categoryId": "uuid-category-1",
  "price": 499000,
  "level": "beginner"
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-course-1",
    "title": "React từ Zero đến Hero",
    "status": "draft"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/courses/:id/modules` 🔒 *(role: instructor)*
Thêm chương học mới vào khóa học.

**Request:**
```json
{
  "title": "Chương 1: Giới thiệu"
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-module-1",
    "title": "Chương 1: Giới thiệu",
    "courseId": "uuid-course-1"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/lessons/upload-url` 🔒 *(role: instructor)*
Giảng viên xin link upload trực tiếp video bài học lên Cloudflare R2 (Direct Upload).

**Query Params:** `?filename=lesson_intro.mp4`

**Response 200:**
```json
{
  "success": true,
  "data": {
    "uploadUrl": "https://r2.cloudflare.com/my-bucket/lessons/uuid-lesson_intro.mp4?X-Amz-Signature=...",
    "videoId": "uuid-lesson_intro.mp4"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/modules/:id/lessons` 🔒 *(role: instructor)*
Tạo bài học mới thuộc chương học.

**Request (Video từ Cloudflare R2):**
```json
{ 
  "title": "Bài 2: Sử dụng JSX", 
  "videoSource": "r2", 
  "videoId": "uuid-lesson_intro.mp4", 
  "durationSeconds": 600,
  "isPreview": false 
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-lesson-2",
    "title": "Bài 2: Sử dụng JSX",
    "moduleId": "uuid-module-1",
    "videoSource": "r2",
    "videoId": "uuid-lesson_intro.mp4"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 3. Cart & Voucher (Giỏ hàng & Mã giảm giá)

### POST `/cart/items` 🔒
Thêm một khóa học vào giỏ hàng (lưu Redis theo `userId`, TTL 48h sliding).

**Request:**
```json
{
  "courseId": "uuid-course-1"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "message": "Đã thêm khóa học vào giỏ hàng thành công."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/cart` 🔒
Xem thông tin chi tiết giỏ hàng hiện tại.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "courseId": "uuid-course-1",
        "title": "React từ Zero đến Hero",
        "price": 499000
      }
    ],
    "subtotal": 499000
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### DELETE `/cart/items/:courseId` 🔒
Xóa khóa học khỏi giỏ hàng.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "message": "Đã xóa khóa học khỏi giỏ hàng thành công."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/cart/apply-voucher` 🔒
Áp dụng mã giảm giá cho giỏ hàng hiện tại.

**Request:**
```json
{
  "code": "SUMMER20"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "valid": true,
    "discountAmount": 99800,
    "total": 399200
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/vouchers` 🔒 *(role: admin, instructor)*
Tạo mã giảm giá mới.

**Request:**
```json
{
  "code": "INSTRUCT25",
  "discountType": "percent",
  "discountValue": 25.00,
  "minOrderAmount": 100000,
  "maxUses": 100,
  "scope": "course",
  "courseId": "uuid-course-1",
  "validFrom": "2026-06-26T17:00:00Z",
  "validUntil": "2026-07-26T17:00:00Z"
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-voucher-1",
    "code": "INSTRUCT25",
    "discountType": "percent",
    "discountValue": 25.00,
    "scope": "course",
    "courseId": "uuid-course-1",
    "maxUses": 100
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/vouchers` 🔒 *(role: admin, instructor)*
Xem danh sách voucher do người dùng này quản lý/tạo ra.

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-voucher-1",
      "code": "INSTRUCT25",
      "discountType": "percent",
      "discountValue": 25.00,
      "scope": "course",
      "courseId": "uuid-course-1",
      "usedCount": 5,
      "maxUses": 100
    }
  ],
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 4. Orders & Payment (Đơn hàng & Thanh toán)

### POST `/orders/checkout` 🔒
Thanh toán giỏ hàng hiện tại, trả về thông tin tùy theo phương thức thanh toán được chọn.

**Request (Thanh toán qua VietQR):**
```json
{
  "voucherCode": "SUMMER20",
  "paymentMethod": "vietqr"
}
```

**Response 200 (VietQR):**
```json
{
  "success": true,
  "data": {
    "orderId": "uuid-order-1",
    "total": 399200,
    "paymentMethod": "vietqr",
    "qrCodeUrl": "https://api.vietqr.io/image/970415-123456789-Q4zK9t.jpg?amount=399200&addInfo=DH123",
    "transactionReference": "DH123",
    "message": "Vui lòng chuyển khoản đúng số tiền và nội dung chuyển khoản, sau đó tải lên ảnh minh chứng giao dịch."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

**Request (Thanh toán qua Stripe):**
```json
{
  "voucherCode": "SUMMER20",
  "paymentMethod": "stripe"
}
```

**Response 200 (Stripe):**
```json
{
  "success": true,
  "data": {
    "orderId": "uuid-order-2",
    "total": 399200,
    "paymentMethod": "stripe",
    "stripeCheckoutUrl": "https://checkout.stripe.com/c/pay/cs_test_..."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/orders/vietqr-webhook` *(Webhook VietQR nhận từ cổng thanh toán)*
Xử lý thanh toán tự động khi đối tác thanh toán (PayOS/Casso/SePay) gửi Webhook biến động số dư chuyển khoản VietQR thành công.

**Headers:**
- `x-api-key`: Hoặc chữ ký số bảo mật `Signature` để xác thực nguồn gửi.

**Request Body:**
```json
{
  "success": true,
  "data": {
    "orderCode": 10293,
    "amount": 399200,
    "description": "DH123 thanh toan khoa hoc",
    "reference": "FT2617823912",
    "transactionDateTime": "2026-06-27T09:50:00Z"
  },
  "signature": "6a9e1d8820c749b567b57b54..."
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "message": "Webhook processed successfully, order updated to PAID."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/orders/stripe-webhook` *(Webhook Stripe gọi về)*
Xử lý thanh toán tự động khi học viên thanh toán thành công qua Stripe.

**Headers:** `Stripe-Signature: t=1614...v3=...`

**Request Body:** Stripe Event JSON (chứa thông tin sự kiện giao dịch).

**Response 200:**
```json
{
  "success": true,
  "data": {
    "received": true
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 5. Learning & Progress (Học bài & Tiến độ)

### GET `/my-courses` 🔒
Danh sách các khóa học đã sở hữu của học viên hiện tại.

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-course-1",
      "title": "React từ Zero đến Hero",
      "progressPercent": 45
    }
  ],
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/courses/:id/learn` 🔒 *(Yêu cầu đã sở hữu khóa học)*
Lấy toàn bộ khung đề cương chi tiết để bắt đầu học. **Lưu ý:** Không trả về `videoId` hay link video để bảo vệ bản quyền.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "courseTitle": "React từ Zero đến Hero",
    "modules": [
      {
        "title": "Chương 1",
        "lessons": [
          {
            "id": "uuid-lesson-1",
            "title": "Bài 1: Cài đặt môi trường",
            "durationSeconds": 360,
            "isCompleted": false
          }
        ]
      }
    ]
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/lessons/:id/video` 🔒
Lấy liên kết video bài học bảo mật để phát trên Player của Client.

**Response 200 (Nguồn Cloudflare R2):**
```json
{
  "success": true,
  "data": {
    "videoSource": "r2",
    "videoUrl": "https://r2.cloudflare.com/bucket/lesson1.mp4?X-Amz-Expires=900&X-Amz-Signature=..."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

**Response 200 (Nguồn YouTube):**
```json
{
  "success": true,
  "data": {
    "videoSource": "youtube",
    "videoId": "dQw4w9WgXcQ"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/lessons/:id/progress` 🔒
Cập nhật định kỳ tiến độ xem video (được gọi từ Client mỗi 30 giây).

**Request:**
```json
{
  "watchedSeconds": 90,
  "isCompleted": false
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "progressPercent": 45,
    "isCompleted": false
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/courses/:id/reviews` 🔒 *(Yêu cầu đã mua khóa học)*
Đánh giá khóa học (từ 1 đến 5 sao) kèm bình luận.

**Request:**
```json
{
  "rating": 5,
  "comment": "Khóa học rất hay, dễ hiểu!"
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-review-1",
    "rating": 5,
    "comment": "Khóa học rất hay, dễ hiểu!",
    "courseId": "uuid-course-1"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/lessons/:id/comments` 🔒
Lấy danh sách các bình luận, thảo luận bài học (Q&A) theo cấu trúc phân cấp (cây thảo luận).

**Response 200:**
```json
{
  "success": true,
  "data": {
    "comments": [
      {
        "id": "uuid-cmt-1",
        "user": {
          "fullName": "Nguyễn Văn A",
          "avatarUrl": "https://..."
        },
        "content": "Em gặp lỗi khi chạy lệnh npm run dev ạ.",
        "createdAt": "2026-06-26T10:00:00Z",
        "replies": [
          {
            "id": "uuid-cmt-2",
            "user": {
              "fullName": "Trần B (Giảng viên)",
              "avatarUrl": "https://..."
            },
            "content": "Em kiểm tra lại xem đã chạy npm install chưa nhé.",
            "createdAt": "2026-06-26T10:05:00Z"
          }
        ]
      }
    ]
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/lessons/:id/comments` 🔒
Viết bình luận mới hoặc phản hồi lại một bình luận có sẵn dưới video bài học.

**Request:**
```json
{
  "content": "Em làm được rồi ạ, cảm ơn thầy!",
  "parentId": "uuid-cmt-1"
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-cmt-3",
    "content": "Em làm được rồi ạ, cảm ơn thầy!",
    "parentId": "uuid-cmt-1",
    "createdAt": "2026-06-26T10:10:00Z"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 6. AI Chatbot Trợ giảng

### POST `/courses/:id/ai-chat` 🔒 *(Yêu cầu đã sở hữu khóa học)*
Gửi câu hỏi và thảo luận với AI trợ giảng ngay trong bài học.

**Request:**
```json
{
  "lessonId": "uuid-lesson-1",
  "message": "JSX viết tắt của từ gì thế AI?"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "reply": "JSX viết tắt của cụm từ JavaScript XML. Nó cho phép ta viết cú pháp HTML trong tệp JavaScript..."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 7. Certificates (Chứng chỉ)

### GET `/certificates/:code/verify`
Kiểm tra tính hợp lệ của chứng chỉ công khai thông qua mã định danh.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "isValid": true,
    "studentName": "Nguyễn Văn A",
    "courseTitle": "React từ Zero đến Hero",
    "issuedAt": "2026-06-26T12:00:00Z"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 8. Instructor & Wallet (Giảng viên & Rút tiền)

### GET `/instructor/revenue` 🔒 *(role: instructor)*
Thống kê ví tiền, tổng thu nhập và số dư có thể rút của Giảng viên.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "balance": 15400000.00,
    "totalEarned": 28000000.00,
    "totalWithdrawn": 12600000.00
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/instructor/withdrawals` 🔒 *(role: instructor)*
Gửi yêu cầu rút tiền về tài khoản ngân hàng cá nhân.

**Request:**
```json
{
  "amount": 5000000.00,
  "bankName": "Vietcombank",
  "accountNumber": "1012345678",
  "accountHolder": "TRAN VAN B"
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "requestId": "uuid-request-1",
    "status": "pending",
    "amount": 5000000.00
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### GET `/instructor/withdrawals` 🔒 *(role: instructor)*
Xem lịch sử danh sách yêu cầu rút tiền của giảng viên hiện tại.

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-request-1",
      "amount": 5000000.00,
      "status": "pending",
      "createdAt": "2026-06-26T18:00:00Z"
    }
  ],
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

## 9. Admin Operations (Duyệt khóa học & Duyệt rút tiền) 🔒 *(role: admin)*

### POST `/admin/courses/:id/approve`
Phê duyệt và xuất bản khóa học lên sàn thương mại.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-course-1",
    "status": "published"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/admin/courses/:id/reject`
Từ chối xuất bản khóa học và gửi trả về kèm theo lý do từ chối.

**Request:**
```json
{
  "rejectReason": "Nội dung chương 3 không đúng chuẩn chuẩn mực..."
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-course-1",
    "status": "rejected",
    "rejectReason": "Nội dung chương 3 không đúng chuẩn chuẩn mực..."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/admin/withdrawals/:id/approve`
Phê duyệt yêu cầu rút tiền của giảng viên (sau khi Admin đã thực hiện chuyển tiền thành công bên ngoài ngân hàng).

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-request-1",
    "status": "approved"
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

---

### POST `/admin/withdrawals/:id/reject`
Từ chối yêu cầu rút tiền và hoàn tiền lại ví số dư cho giảng viên.

**Request:**
```json
{
  "rejectReason": "Tên chủ tài khoản không khớp với tên giảng viên đã xác minh."
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-request-1",
    "status": "rejected",
    "rejectReason": "Tên chủ tài khoản không khớp với tên giảng viên đã xác minh."
  },
  "timestamp": "2026-06-26T12:00:00Z"
}
```

