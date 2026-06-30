# Database Schema — Nền tảng học trực tuyến (Udemy Mini)

> Thiết kế cơ sở dữ liệu PostgreSQL cho hệ thống bán khóa học online

---

## Sơ đồ ERD tổng quan (dạng text)

```
users ──┬──< enrollments >──┬── courses ──┬──< modules >──< lessons ──< lesson_comments
        │                   │             │
        │                   │             ├──< course_reviews
        │                   │             │
        ├──< orders >──< order_items >─────┘
        │      │
        │      └──> vouchers (FK trực tiếp trong orders)
        │
        ├──< lesson_progress >── lessons
        │
        ├──< lesson_comments >── lessons
        │
        ├──< certificates >── courses
        │
        ├──< instructor_profile (nếu role = instructor)
        │
        └──< withdrawal_requests (nếu role = instructor)
```

---

## 1. `users` — Người dùng

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| email | VARCHAR(255) UNIQUE | |
| password_hash | VARCHAR(255) | null nếu login Google |
| full_name | VARCHAR(100) | |
| avatar_url | VARCHAR(255) | |
| role | ENUM('student','instructor','admin','staff') | default 'student' |
| google_id | VARCHAR(100) | null nếu đăng ký thường |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

> [!NOTE]
> **Quy tắc xác thực & Xử lý xung đột tài khoản (OAuth2 Google vs Local):**
> * **Tài khoản Google thuần:** `password_hash` IS NULL, `google_id` NOT NULL. Giao diện Profile ẩn tính năng đổi mật khẩu. Khi Quên mật khẩu, Backend không tạo link reset mà gửi email thông báo yêu cầu đăng nhập qua Google.
> * **Tài khoản Local:** `password_hash` NOT NULL, `google_id` IS NULL. Đăng nhập bằng mật khẩu bình thường.
> * **Auto-Linking:** Nếu người dùng đã đăng ký Local trước đó bằng Gmail, khi họ bấm "Đăng nhập bằng Google", Backend sẽ tự động cập nhật `google_id` và liên kết tài khoản (cả hai trường đều NOT NULL). Giao diện Profile vẫn cho phép đổi mật khẩu.

---

## 2. `instructor_profiles` — Hồ sơ giảng viên

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users.id) | |
| headline | VARCHAR(150) | VD: "Senior Backend Developer" |
| bio | TEXT | |
| balance | DECIMAL(10,2) | Số dư thu nhập hiện tại (sau phí sàn), mặc định 0.00 |
| total_students | INT | tính toán định kỳ |
| total_courses | INT | tính toán định kỳ |

---

## 3. `categories` — Danh mục khóa học

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| name | VARCHAR(100) | VD: "Lập trình Web" |
| slug | VARCHAR(100) UNIQUE | dùng cho URL |

---

## 4. `courses` — Khóa học

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| instructor_id | UUID (FK → users.id) | |
| category_id | UUID (FK → categories.id) | |
| title | VARCHAR(200) | |
| slug | VARCHAR(200) UNIQUE | |
| description | TEXT | |
| thumbnail_url | VARCHAR(255) | |
| price | DECIMAL(10,2) | giá gốc |
| level | ENUM('beginner','intermediate','advanced') | |
| status | ENUM('draft','published','archived') | |
| total_duration_minutes | INT | tổng thời lượng video |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

---

## 5. `modules` — Chương học (nhóm bài học)

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| course_id | UUID (FK → courses.id) | |
| title | VARCHAR(200) | VD: "Chương 1: Giới thiệu" |
| position | INT | thứ tự hiển thị |

---

## 6. `lessons` — Bài học

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| module_id | UUID (FK → modules.id) | |
| title | VARCHAR(200) | |
| video_source | ENUM('youtube','vimeo','r2') | Đã đổi drive thành r2 |
| video_id | VARCHAR(255) | ID hoặc tên tệp đã extract (không lưu raw URL) |
| duration_seconds | INT | |
| position | INT | thứ tự trong chương |
| is_preview | BOOLEAN | cho xem free thử không cần mua |
| created_at | TIMESTAMP | |

---

## 7. `orders` — Đơn hàng

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users.id) | |
| voucher_id | UUID (FK → vouchers.id) | nullable, liên kết trực tiếp để theo dõi voucher áp dụng |
| subtotal | DECIMAL(10,2) | tổng trước giảm giá |
| discount_amount | DECIMAL(10,2) | số tiền được giảm |
| total | DECIMAL(10,2) | số tiền thực tế phải thanh toán |
| status | ENUM('pending','paid','failed','cancelled') | trạng thái đơn hàng |
| payment_method | ENUM('vietqr','stripe') | Phương thức thanh toán |
| transaction_reference | VARCHAR(150) | Mã nội dung chuyển khoản động (VietQR) hoặc mã tham chiếu giao dịch (Stripe) |
| created_at | TIMESTAMP | |
| paid_at | TIMESTAMP | nullable |

---

## 8. `order_items` — Chi tiết đơn hàng (giỏ hàng đã chốt)

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| order_id | UUID (FK → orders.id) | |
| course_id | UUID (FK → courses.id) | |
| price_at_purchase | DECIMAL(10,2) | giá lúc mua, tránh đổi giá sau ảnh hưởng lịch sử |

---

## 9. `vouchers` — Mã giảm giá

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| code | VARCHAR(50) UNIQUE | VD: "SUMMER20" |
| discount_type | ENUM('percent','fixed') | |
| discount_value | DECIMAL(10,2) | |
| min_order_amount | DECIMAL(10,2) | nullable |
| max_uses | INT | nullable, null = không giới hạn |
| used_count | INT | default 0 |
| scope | ENUM('global','course') | Phạm vi áp dụng: toàn sàn hoặc chỉ khóa học |
| course_id | UUID (FK → courses.id) | nullable, chỉ định khóa học nếu scope = course |
| created_by_id | UUID (FK → users.id) | người tạo (Admin tạo global, Instructor tạo course) |
| valid_from | TIMESTAMP | |
| valid_until | TIMESTAMP | |

---

## 10. `enrollments` — Ghi danh (đã mua khóa học)

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users.id) | |
| course_id | UUID (FK → courses.id) | |
| order_id | UUID (FK → orders.id) | |
| enrolled_at | TIMESTAMP | |
| progress_percent | DECIMAL(5,2) | tính toán định kỳ, 0–100 |
| completed_at | TIMESTAMP | nullable |

> Index UNIQUE (user_id, course_id) — tránh enroll trùng

---

## 11. `lesson_progress` — Tiến độ học từng bài

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users.id) | |
| lesson_id | UUID (FK → lessons.id) | |
| watched_seconds | INT | xem đến giây thứ mấy |
| is_completed | BOOLEAN | đã xem hết chưa |
| last_watched_at | TIMESTAMP | |

> Index UNIQUE (user_id, lesson_id)

---

## 12. `lesson_comments` — Bình luận, thảo luận bài học (Q&A)

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| lesson_id | UUID (FK → lessons.id) | |
| user_id | UUID (FK → users.id) | Người bình luận |
| content | TEXT | Nội dung bình luận |
| parent_id | UUID (FK → lesson_comments.id) | nullable, tự tham chiếu làm reply |
| created_at | TIMESTAMP | |

---

## 13. `course_reviews` — Đánh giá khóa học

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| course_id | UUID (FK → courses.id) | |
| user_id | UUID (FK → users.id) | |
| rating | INT | 1–5 |
| comment | TEXT | |
| created_at | TIMESTAMP | |

---

## 14. `certificates` — Chứng chỉ hoàn thành

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users.id) | |
| course_id | UUID (FK → courses.id) | |
| certificate_code | VARCHAR(50) UNIQUE | dùng để verify online |
| issued_at | TIMESTAMP | |

---

## 15. `withdrawal_requests` — Yêu cầu rút tiền giảng viên

| Cột | Loại | Ghi chú |
|---|---|---|
| id | UUID (PK) | |
| instructor_id | UUID (FK → users.id) | Giảng viên tạo yêu cầu |
| amount | DECIMAL(10,2) | Số tiền yêu cầu rút |
| status | ENUM('pending','approved','rejected') | Trạng thái phê duyệt và chuyển khoản |
| bank_name | VARCHAR(100) | Tên ngân hàng nhận tiền |
| account_number | VARCHAR(50) | Số tài khoản ngân hàng |
| account_holder | VARCHAR(100) | Tên chủ tài khoản |
| reject_reason | VARCHAR(255) | Lý do từ chối rút (nếu status = rejected) |
| created_at | TIMESTAMP | |
| processed_at | TIMESTAMP | nullable, thời điểm Admin duyệt |

---

## Chiến lược Foreign Key & Index

> Không đánh index toàn bộ cột. Chỉ dùng **Foreign Key** cho dữ liệu lõi cần bảo toàn quan hệ chặt chẽ; các dữ liệu phụ/log/snapshot có thể chỉ lưu UUID tham chiếu và kiểm tra bằng Backend nếu cần.

### Foreign Key bắt buộc cho dữ liệu quan trọng

Các quan hệ này nên khai báo FK thật trong PostgreSQL để tránh dữ liệu mồ côi:

```
users.id       ← instructor_profiles.user_id
users.id       ← courses.instructor_id
users.id       ← orders.user_id
users.id       ← enrollments.user_id
users.id       ← lesson_progress.user_id
users.id       ← certificates.user_id

categories.id  ← courses.category_id
courses.id     ← modules.course_id
modules.id     ← lessons.module_id
orders.id      ← order_items.order_id
courses.id     ← order_items.course_id
courses.id     ← enrollments.course_id
lessons.id     ← lesson_progress.lesson_id
courses.id     ← certificates.course_id
```

### Quan hệ có thể dùng FK mềm hoặc xử lý bằng Backend

Các quan hệ này không nhất thiết phải dùng FK cứng nếu muốn linh hoạt khi xóa/ẩn dữ liệu, nhưng Backend phải kiểm tra tồn tại khi tạo/cập nhật:

```
vouchers.id           ← orders.voucher_id        -- có thể ON DELETE SET NULL
vouchers.course_id    → courses.id               -- voucher theo khóa học, có thể validate bằng Backend
vouchers.created_by_id → users.id                -- lưu người tạo, có thể giữ snapshot/log
lesson_comments.user_id → users.id               -- có thể giữ bình luận kể cả khi user bị khóa/xóa mềm
lesson_comments.lesson_id → lessons.id           -- tùy chính sách xóa bài học
lesson_comments.parent_id → lesson_comments.id   -- reply comment, có thể SET NULL khi comment cha bị xóa
course_reviews.user_id → users.id                -- review có thể giữ lịch sử
course_reviews.course_id → courses.id            -- tùy chính sách archive course
withdrawal_requests.instructor_id → users.id     -- dữ liệu tài chính nên ưu tiên không xóa cứng user
```

### Quy tắc `ON DELETE` đề xuất

- **Không xóa cứng dữ liệu tài chính/học tập đã phát sinh**: `orders`, `order_items`, `enrollments`, `certificates` nên được giữ để bảo toàn lịch sử.
- **Course đã bán/published**: không `DELETE`, chỉ chuyển `status = 'archived'`.
- **Course draft chưa ai mua**: có thể cho xóa, dùng `ON DELETE CASCADE` có kiểm soát cho `modules` và `lessons`.
- **Voucher trong order**: nếu cho xóa voucher thì dùng `ON DELETE SET NULL`, nhưng vẫn giữ `discount_amount` trong order.
- **Comment/review**: ưu tiên soft delete bằng `deleted_at` nếu cần kiểm duyệt, tránh mất lịch sử thảo luận.

### Index đề xuất theo hướng doanh nghiệp

#### Unique index / constraint

```
users(email)
users(google_id)                  -- nullable unique nếu hỗ trợ Google OAuth
categories(slug)
courses(slug)
vouchers(code)
certificates(certificate_code)
enrollments(user_id, course_id)
lesson_progress(user_id, lesson_id)
```

#### Index cho query phổ biến và join

```
instructor_profiles(user_id)

courses(instructor_id)
courses(category_id)
courses(category_id, status)
courses(instructor_id, status)

modules(course_id, position)
lessons(module_id, position)

orders(user_id, created_at)
orders(transaction_reference)

order_items(order_id)
order_items(course_id)

enrollments(user_id, course_id)
enrollments(course_id)

lesson_progress(user_id, lesson_id)
lesson_progress(lesson_id)

lesson_comments(lesson_id, created_at)
lesson_comments(parent_id)

course_reviews(course_id, created_at)
course_reviews(user_id, course_id)

certificates(user_id, course_id)

withdrawal_requests(instructor_id, status)
```

#### Không index mặc định các cột ít lọc hoặc dữ liệu dài

Không nên index bừa các cột chỉ dùng để hiển thị hoặc cập nhật thường xuyên:

```
users.full_name
users.avatar_url
courses.description
courses.thumbnail_url
instructor_profiles.bio
orders.subtotal
orders.discount_amount
orders.total
lesson_progress.watched_seconds
lesson_comments.content
course_reviews.comment
withdrawal_requests.reject_reason
```

---

## Lưu ý kỹ thuật khi implement

- **video_id, không lưu raw URL** — bảo vệ và chuẩn hóa nguồn video (xem tài liệu xử lý video riêng)
- **price_at_purchase** trong order_items — bắt buộc lưu snapshot giá, tránh trường hợp giảng viên đổi giá sau làm sai lịch sử đơn hàng
- **voucher_id** liên kết trực tiếp trong orders — giúp truy xuất voucher đã áp dụng nhanh chóng mà không cần qua bảng trung gian phức tạp.
- **progress_percent** nên cache trong `enrollments`, tính lại bằng cron job hoặc trigger khi `lesson_progress` cập nhật — tránh tính realtime tốn query.
- **Redis cache** — cache danh sách `lessons` theo `course_id`, cache `course` detail vì đọc nhiều hơn viết.
- **Kafka events** — bắn event khi: `order.paid`, `lesson.completed`, `course.completed` → consumer gửi email, cập nhật certificate.
