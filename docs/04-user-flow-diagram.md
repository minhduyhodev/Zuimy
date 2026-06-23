# User Flow Diagram — Nền tảng học trực tuyến (Udemy Mini)

> Mô tả luồng di chuyển và hành động của từng vai trò người dùng trong hệ thống

---

## 1. Luồng Học viên (Student) — Tổng quan

```
┌─────────────┐
│  Truy cập   │
│  trang chủ  │
└──────┬──────┘
       │
       ▼
┌─────────────────┐      Chưa có tài khoản     ┌──────────────┐
│  Duyệt khóa học  │ ───────────────────────►  │ Đăng ký/     │
│  (browse/search) │                            │ Đăng nhập    │
└──────┬───────────┘                            └──────┬───────┘
       │                                                │
       ▼                                                │
┌─────────────────┐                                     │
│  Xem chi tiết    │ ◄───────────────────────────────────┘
│  khóa học        │
└──────┬───────────┘
       │
       ├──► Xem preview bài học miễn phí (is_preview = true)
       │
       ▼
┌─────────────────┐
│  Thêm vào         │
│  giỏ hàng         │
└──────┬───────────┘
       │
       ▼
┌─────────────────┐
│  Xem giỏ hàng     │
│  + áp voucher     │
└──────┬───────────┘
       │
       ▼
┌─────────────────┐
│  Checkout         │
│  → Quét VietQR    │
└──────┬───────────┘
       │
       ├──► Hủy thanh toán / Hết hạn ──► Quay lại giỏ hàng
       │
       ▼ Thành công (Webhook xử lý ngầm tự động)
┌─────────────────┐
│  Enrollment       │
│  được tạo         │
└──────┬───────────┘
       │
       ▼
┌─────────────────┐
│  Vào "Khóa học    │
│  của tôi"         │
└──────┬───────────┘
       │
       ▼
┌─────────────────┐
│  Học bài          │
│  (xem video)      │
└──────┬───────────┘
       │
       ├──► Tiến độ được validate và lưu tạm trên Redis Hash mỗi 30s
       │
       ▼ Hoàn thành tất cả bài học + Quiz đạt điểm
┌─────────────────┐
│  Nhận             │
│  Certificate      │
└──────┬───────────┘
       │
       ▼
┌─────────────────┐
│  Đánh giá         │
│  khóa học         │
└───────────────────┘
```

---

## 2. Luồng chi tiết — Mua khóa học (Cart & Payment)

```
[Trang chi tiết khóa học]
        │
        │ Click "Thêm vào giỏ"
        ▼
[Giỏ hàng — Redis lưu tạm theo user_id, TTL 48h sliding expiration]
        │
        ├─── Thêm khóa học khác ──► Quay lại browse
        │
        │ Nhập mã voucher
        ▼
[Backend validate voucher]
        │
        ├─── Không hợp lệ ──► Hiển thị lỗi, giữ nguyên giỏ hàng
        │
        ▼ Hợp lệ
[Hiển thị giá mới sau giảm]
        │
        │ Click "Thanh toán"
        ▼
[Tạo Order — status: PENDING]
        │
        ▼
[Hiển thị mã QR thanh toán động VietQR & Nội dung chuyển khoản]
        │
        │ Học viên thực hiện chuyển khoản bằng app ngân hàng
        ▼
[Ngân hàng bắn Webhook giao dịch về Backend]
        │
        ▼
[Backend xác thực chữ ký Webhook (Signature Verification)]
        │
        ▼
[Mở DB Transaction với khóa dòng SELECT FOR UPDATE trên bảng orders]
        │
        ├─── Trùng lặp (status đã là PAID) ──► Trả về HTTP 200 ngay (Idempotent API)
        │
        ▼ Chưa xử lý (status là PENDING)
[Cập nhật Order status = PAID]
        │
        ├──► Cộng tiền vào ví balance của Giảng viên sở hữu (chia sẻ 80/20)
        ├──► Tạo Enrollment cho từng course trong đơn hàng
        ├──► Đẩy sự kiện sang Kafka "order.paid"
        │       ├─ Consumer: gửi email hóa đơn thành công (Spring Mail)
        │       └─ Consumer: xóa giỏ hàng trong Redis
        │
        ▼
[Frontend nhận tín hiệu tự động chuyển sang trang "Khóa học của tôi"]
```

---

## 3. Luồng chi tiết — Học bài, Bảo mật Video & Tiến độ

```
[Khóa học của tôi] → Chọn 1 khóa học đã mua
        │
        ▼
[Backend kiểm tra: user_id đăng nhập (Spring Security JWT) đã enroll?]
        │
        ├─── Chưa ──► Trả về 403 Forbidden
        │
        ▼ Đã Enroll
[Trả về đề cương bài học — ẨN hoàn toàn video_id]
        │
        │ Học viên click chọn học 1 bài
        ▼
[Call GET /api/v1/lessons/:id/video]
        │
        ├─── Nguồn R2: Backend sinh Presigned GET URL (hạn 15 phút)
        │              → Trả về Frontend phát trên ReactPlayer
        │
        └─── Nguồn YouTube: Trả về video_id
                           → Frontend bọc ReactPlayer + CSS Overlay chặn chuột phải/click đúp
        │
        ▼ Player chạy → Tự động call API tiến độ mỗi 30 giây
[POST /lessons/:id/progress { watched_seconds }]
        │
        ▼
[Backend kiểm tra thời gian thực ở Server (Server-side Validation)]
        │
        ├─── Vượt quá thời gian trôi qua thực tế ──► Từ chối, ghi log cảnh báo gian lận
        │
        ▼ Hợp lệ
[Lưu tạm tiến độ vào Redis Hash]
        │
        ├─── Chuyển bài/Kết thúc phiên học ──► Flush (ghi dồn) từ Redis xuống PostgreSQL
        │
        ▼
[Backend check: watched_seconds >= 90% duration?]
        │
        ├─── Chưa ──► Lưu tiến độ bình thường
        │
        ▼ Đủ
[Đánh dấu lesson.is_completed = true]
        │
        ▼
[Kiểm tra: tất cả lessons & Quiz trong course đã hoàn thành?]
        │
        ├─── Chưa ──► Cập nhật progress_percent của enrollment
        │
        ▼ Đã hoàn thành 100%
[Kafka event "course.completed"]
        │
        ├──► Consumer: sinh file PDF Certificate
        └──► Consumer: gửi email chúc mừng kèm link tải chứng chỉ
        │
        ▼
[Học viên thấy nút "Tải chứng chỉ" hiển thị trên giao diện]
```

---

## 4. Luồng Giảng viên (Instructor) — Tạo khóa học & Tải Video R2

```
[Đăng nhập với tài khoản instructor]
        │
        ▼
[Dashboard giảng viên] → Click "Tạo khóa học mới"
        │
        ▼
[Điền thông tin: tên, danh mục, giá, mô tả → Trạng thái DRAFT]
        │
        ▼
[Thêm Module (chương học) → Thêm Lesson]
        │
        ├─── Nhánh 1: Dán video URL Youtube 
        │             → Backend parse lưu video_id
        │
        └─── Nhánh 2: Upload trực tiếp video R2 (Direct Upload)
                      ├─ Frontend xin Backend link upload
                      ├─ Backend sinh Presigned PUT URL từ R2 gửi về
                      ├─ Frontend gửi file video trực tiếp lên R2 qua link (không qua Backend)
                      └─ Upload xong, Frontend gửi video_id (tên file R2) lên Backend để lưu
        │
        ▼
[Lặp lại thêm đầy đủ nội dung bài học & Quiz]
        │
        │ Click "Yêu cầu xuất bản"
        ▼
[Course status = PENDING_APPROVAL]
        │
        ▼
[Admin xem danh sách duyệt khóa học]
        │
        ├─── Admin từ chối ──► Trạng thái REJECTED (gửi kèm lý do cho giảng viên sửa)
        │
        ▼ Admin phê duyệt
[Course status = PUBLISHED] → Hiển thị công khai trên sàn khóa học
```

---

## 5. Luồng Giảng viên — Theo dõi ví & Rút tiền thủ công

```
[Dashboard giảng viên] → Tab "Doanh thu & Số dư"
        │
        ├─── Xem số dư khả dụng (balance) hiện tại
        │
        │ Click "Yêu cầu rút tiền"
        ▼
[Điền số tiền muốn rút + thông tin tài khoản ngân hàng nhận tiền]
        │
        ▼
[Backend kiểm tra balance]
        │
        ├─── Số dư không đủ ──► Trả về lỗi 400
        │
        ▼ Đủ số dư
[Tạo lệnh rút trạng thái PENDING]
        │
        ▼
[Admin nhận thông tin yêu cầu rút trên Admin Panel]
        │
        ├─ Admin thực hiện chuyển khoản thủ công trên ứng dụng ngân hàng thực tế
        ▼
[POST /admin/withdrawals/:id/approve]
        │
        ▼
[Trạng thái lệnh rút thành APPROVED, cập nhật trừ balance của Giảng viên]
```

---

## 6. Sơ đồ phân quyền các vai trò hệ thống

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   STUDENT    │     │  INSTRUCTOR  │     │    ADMIN     │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ Browse course│     │ Create course│     │ Manage users │
│ Add to cart  │     │ Add lessons  │     │ Manage       │
│ Apply voucher│     │ Direct Up R2 │     │   vouchers   │
│ Checkout QR  │     │ Request pub  │     │ Moderate     │
│ Watch video  │     │ View revenue │     │   courses    │
│ Ask AI Chat  │     │ View students│     │ Manual pay   │
│ Get cert PDF │     │ Withdraw request │ Approve withdraw│
│ Write review │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
        │                    │                    │
        └────────────────────┴────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Shared Backend  │
                    │  PostgreSQL      │
                    │  Redis (cache)   │
                    │  Kafka (events)  │
                    └──────────────────┘
```

---

## Điểm cần lưu ý khi code (Dành cho Developer)

- **Nguyên lý Idempotency thanh toán:** Giao dịch VietQR bắn webhook trùng lặp do retry phải bị chặn ở database layer bằng transaction với khóa dòng (`SELECT ... FOR UPDATE` trong JPA).
- **Nguyên lý Zero Trust Tiến độ:** Không được cập nhật thẳng tiến độ từ client gửi lên. Bắt buộc phải validate khoảng thời gian trôi qua thực tế ở server và cache tạm bằng Redis Hash trước khi ghi dồn (flush) xuống PostgreSQL.
- **Bảo mật video ở mức tối đa:** Ẩn toàn bộ `video_id` ở API Đề cương. Chỉ trả link động 15 phút (Presigned GET URL R2) hoặc ID Youtube qua API xem bài học có xác thực JWT của Spring Security.
- **Vấn đề đồng thì (Concurrency) của Voucher:** Khi học viên Checkout cùng lúc áp dụng voucher giới hạn lượt dùng, phải sử dụng transaction hoặc khóa bi quan/lạc quan để tránh việc vượt quá số lượng dùng tối đa (`max_uses`).
- **Tránh trùng Certificate:** Cần check xem certificate đã được sinh cho `(user_id, course_id)` chưa trước khi tạo mới để tránh Kafka consumer chạy lại sự kiện gây lỗi trùng bản ghi.
