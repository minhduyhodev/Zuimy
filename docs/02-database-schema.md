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
| status | ENUM('pending','submitted_proof','paid','failed','cancelled','rejected_proof') | trạng thái đơn hàng |
| payment_method | ENUM('vietqr','stripe') | Phương thức thanh toán |
| transaction_reference | VARCHAR(150) | Nội dung chuyển khoản hoặc mã tham chiếu giao dịch (đối soát Stripe hoặc nhập tay) |
| transaction_proof_url | VARCHAR(255) | nullable, link ảnh minh chứng chuyển khoản tải lên cho VietQR |
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

## Quan hệ chính (Foreign Keys)

```
users.id            ← courses.instructor_id
users.id            ← orders.user_id
users.id            ← enrollments.user_id
users.id            ← lesson_progress.user_id
users.id            ← withdrawal_requests.instructor_id
users.id            ← lesson_comments.user_id
lessons.id          ← lesson_comments.lesson_id
lesson_comments.id  ← lesson_comments.parent_id
vouchers.created_by_id ← users.id
vouchers.course_id  ← courses.id
courses.id          ← modules.course_id
modules.id          ← lessons.module_id
courses.id          ← order_items.course_id
orders.id           ← order_items.order_id
vouchers.id         ← orders.voucher_id
courses.id          ← enrollments.course_id
lessons.id          ← lesson_progress.lesson_id
courses.id          ← certificates.course_id
```

---

## Lưu ý kỹ thuật khi implement

- **video_id, không lưu raw URL** — bảo vệ và chuẩn hóa nguồn video (xem tài liệu xử lý video riêng)
- **price_at_purchase** trong order_items — bắt buộc lưu snapshot giá, tránh trường hợp giảng viên đổi giá sau làm sai lịch sử đơn hàng
- **voucher_id** liên kết trực tiếp trong orders — giúp truy xuất voucher đã áp dụng nhanh chóng mà không cần qua bảng trung gian phức tạp.
- **progress_percent** nên cache trong `enrollments`, tính lại bằng cron job hoặc trigger khi `lesson_progress` cập nhật — tránh tính realtime tốn query.
- **Redis cache** — cache danh sách `lessons` theo `course_id`, cache `course` detail vì đọc nhiều hơn viết.
- **Kafka events** — bắn event khi: `order.paid`, `lesson.completed`, `course.completed` → consumer gửi email, cập nhật certificate.
