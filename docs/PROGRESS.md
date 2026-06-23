# PROGRESS.md

> File theo dõi tiến trình thực hiện dự án Udemy Mini.  
> Cập nhật file này sau mỗi buổi code — giúp bạn (và AI coding agent) biết đang ở đâu, làm gì tiếp theo.

---

## Trạng thái tổng quan

**Giai đoạn hiện tại:** 📋 Thiết lập và đồng bộ tài liệu kiến trúc mới (Chưa bắt đầu code)  
**Cập nhật lần cuối:** 23/06/2026

---

## Checklist theo Roadmap 8 tuần (Đã đồng bộ)

### Tuần 1 — Setup & Auth (Spring Boot & React Vite)
- [ ] Khởi tạo dự án Spring Boot Backend & React Vite Frontend.
- [ ] Cấu hình PostgreSQL, kết nối thành công.
- [ ] Thiết kế và chạy scripts khởi tạo Database Schema (UUID PK).
- [ ] Tích hợp Spring Security, viết API Đăng ký / Đăng nhập sử dụng JWT.
- [ ] Tích hợp Google OAuth2 cho Auth flow.
- [ ] Viết bộ lọc bảo mật JWT (JWT Authentication Filter) và phân quyền Role (`student`, `instructor`, `admin`).

### Tuần 2 — Course / Module / Lesson & R2 Presigned Upload
- [ ] Xây dựng APIs CRUD cho `courses` (Instructor quản lý).
- [ ] Xây dựng APIs CRUD cho `modules` và `lessons`.
- [ ] Tích hợp AWS S3 Java SDK (R2) để sinh Presigned PUT URL hỗ trợ Frontend tải video trực tiếp lên Cloudflare R2.
- [ ] Viết hàm parse URL YouTube Unlisted để extract `video_id` trên Backend.
- [ ] Validate đầu vào của Lesson: chỉ nhận `{ video_source, video_id }`, tuyệt đối không lưu raw URL.

### Tuần 3 — Cart & Voucher (Redis Cache)
- [ ] Setup Redis Connection trong Spring Boot.
- [ ] Xây dựng Cart Service sử dụng Redis làm cache lưu giỏ hàng theo `user_id` với TTL 48h Sliding Expiration (tự động reset/gia hạn 48h khi học viên tương tác).
- [ ] Xây dựng APIs quản lý `vouchers` (Admin).
- [ ] API áp voucher vào giỏ hàng.
- [ ] Áp dụng transaction hoặc khóa dòng (`SELECT ... FOR UPDATE`) để giải quyết tranh chấp Voucher (`max_uses`) khi có lượng lớn người Checkout đồng thời.

### Tuần 4 — VietQR Webhook & Enrollment
- [ ] Xây dựng API Checkout: tạo Order (status: `PENDING`), tạo nội dung chuyển khoản động (`transaction_reference`) và sinh link mã quét VietQR.
- [ ] Xây dựng API nhận Webhook thanh toán ngân hàng (`POST /orders/vietqr-webhook`).
- [ ] Triển khai bộ lọc bảo mật 2 lớp:
  - [ ] Xác thực chữ ký số Webhook (`X-Webhook-Signature`).
  - [ ] Database Transaction với khóa dòng (`SELECT ... FOR UPDATE` trong PostgreSQL) để đảm bảo Idempotency (chặn xử lý trùng giao dịch).
- [ ] Xây dựng logic tạo `enrollments`, cộng số dư `balance` (sau khi chia sẻ 80/20) vào ví Giảng viên khi đơn hàng chuyển sang `PAID`.
- [ ] Setup Upstash Kafka (Serverless Kafka), bắn sự kiện `order.paid`.
- [ ] Viết Kafka Consumer xử lý gửi email hóa đơn bất đồng bộ qua Spring Mail.

### Tuần 5 — Learning & Progress Tracking (Redis Hash)
- [ ] Viết API lấy đề cương học (`/courses/:id/learn`) - ẩn hoàn toàn `video_id` của bài học.
- [ ] Viết API bảo mật lấy link/ID video (`/lessons/:id/video`) - xác thực người dùng từ Security Context JWT và kiểm tra Enrollment hợp lệ. Sinh link Presigned GET URL (15p) đối với nguồn video R2.
- [ ] Xây dựng API cập nhật tiến độ xem video (`POST /lessons/:id/progress`).
- [ ] Thiết kế cơ chế bảo mật tiến độ ở Backend (Server-side Validation): so sánh khoảng tăng tiến độ với khoảng thời gian thực tế trôi qua.
- [ ] Cấu hình Redis Hash lưu tạm thời tiến độ học tập và last update time trên RAM.
- [ ] Viết logic tự động flush (ghi dồn) dữ liệu tiến độ từ Redis xuống PostgreSQL (`lesson_progress`) vào cuối phiên học/chuyển bài.
- [ ] Ghi nhận `is_completed = true` khi xem đạt >= 90%, kiểm tra hoàn thành toàn khóa học và đẩy sự kiện `course.completed` sang Kafka.

### Tuần 6 — Certificate & Instructor Dashboard (Ví & Rút tiền)
- [ ] Viết Kafka Consumer nhận sự kiện `course.completed` để sinh file PDF Certificate và gửi email chúc mừng qua Spring Mail.
- [ ] Viết API verify online Certificate (Public).
- [ ] Xây dựng ví Giảng viên: API rút tiền `POST /instructor/withdrawals` (kiểm tra số dư balance, tạo lệnh rút status `PENDING`).
- [ ] Xây dựng Panel của Admin: duyệt rút tiền thủ công `POST /admin/withdrawals/:id/approve` (admin chuyển tiền ngoài hệ thống ngân hàng thực tế, nhấn duyệt để cập nhật DB) và từ chối rút tiền.
- [ ] API duyệt/từ chối phê duyệt khóa học (`DRAFT` -> `PUBLISHED` / `REJECTED`).

### Tuần 7 — Frontend & Custom Player & Gemini AI
- [ ] Xây dựng giao diện trang chủ, danh sách khóa học (tìm kiếm, lọc) bằng React + Tailwind CSS + Shadcn/ui.
- [ ] Xây dựng giao diện trang học video tích hợp player bảo mật (chặn chuột phải, chặn click đúp, Custom controls che giấu link gốc Youtube).
- [ ] Tích hợp Gemini 1.5 Flash API (gói Free tier) ở Backend. Xây dựng ô chat AI Trợ giảng bên cạnh Video bài học để học viên hỏi đáp nhanh theo ngữ cảnh bài học.
- [ ] Xây dựng giao diện Dashboard Giảng viên (theo dõi số dư, rút tiền, tạo khóa học, upload video R2) và Panel của Admin.

### Tuần 8 — Testing, Optimize & Deploy
- [ ] Viết Integration Tests kiểm tra toàn bộ luồng mua khóa học qua VietQR Webhook.
- [ ] Viết Integration Tests kiểm tra luồng bảo mật tiến độ và tự động cấp chứng chỉ.
- [ ] Deploy Backend lên Railway/Render, Frontend lên Vercel.
- [ ] Cấu hình hạ tầng Cloud: Cloudflare R2, Upstash Kafka, Google AI Studio.
- [ ] Viết tài liệu hướng dẫn vận hành chi tiết trong `README.md`.

---

## Nhật ký làm việc (Work Log)

| Ngày | Đã làm | Đang vướng / cần làm tiếp |
|---|---|---|
| 23/06/2026 | Đồng bộ 100% tài liệu kiến trúc mới (VietQR, Java Spring Boot + React Vite, R2 Direct Upload, Ví tiền, Progress Redis Hash). Cập nhật 01-tong-quan, 02-schema, 03-api-spec, 04-user-flow và AGENTS.md | Chuẩn bị tiến hành setup dự án Backend & Frontend theo Roadmap tuần 1. |

---

## Vấn đề đã chốt (Decided Issues)

- [x] **Quy trình duyệt:** Bắt buộc có kiểm duyệt khóa học bởi Admin trước khi xuất bản công khai.
- [x] **Chứng chỉ:** Chứng chỉ được xuất dưới dạng file PDF/Ảnh sinh ngầm tự động và gửi qua email cho học viên.
- [x] **Cổng thanh toán:** Chọn cổng thanh toán quét mã VietQR tự động qua Webhook thay vì dùng VNPay Sandbox Redirect.
- [x] **Bảo mật video:** Ẩn hoàn toàn video_id ở đề cương, chỉ trả link qua API xem bài học có kiểm tra JWT + Enrollment. Nguồn R2 dùng Presigned GET URL 15 phút. Nguồn Youtube dùng React Player custom che giấu link.
- [x] **Giao dịch rút tiền:** Giảng viên gửi yêu cầu rút tiền -> Admin chuyển khoản ngân hàng thủ công -> Admin bấm duyệt trên Dashboard để cập nhật DB trừ số dư của Giảng viên.

---

## Quick reference — lệnh hay dùng (Spring Boot + React)

```bash
# Chạy Spring Boot Backend (Development)
./mvnw spring-boot:run

# Chạy React Vite Frontend (Development)
npm run dev

# Khởi chạy Docker chứa Redis & Postgres cục bộ phục vụ cho test local
docker compose up -d redis postgres
```
