# Hệ Thống Quản Lý Bãi Đỗ Xe & Sạc Điện

## Mô Tả

Xây dựng hệ thống web hoàn chỉnh gồm backend Python (Flask), cơ sở dữ liệu SQLite (không cần cài đặt MySQL), và giao diện HTML/CSS/JS hiện đại. Hệ thống phục vụ **3 vai trò**: User, Admin bãi đỗ, Tổng giám đốc.

---

## Quyết Định Kỹ Thuật Cần Xác Nhận

> [!IMPORTANT]
> **Database**: Dùng **SQLite** (file-based, không cần cài đặt server) thay vì MySQL để dễ chạy ngay. Nếu bạn muốn MySQL/PostgreSQL, vui lòng nói rõ.

> [!IMPORTANT]
> **Auth**: Dùng session-based login (Flask session) — không dùng JWT token để đơn giản hóa frontend.

> [!NOTE]
> Dữ liệu mẫu (seed data) sẽ được tạo sẵn để demo ngay sau khi chạy.

---

## Cấu Trúc Dự Án

```
parking_system/
├── app.py                  # Flask app chính, routes
├── database.py             # Khởi tạo DB, seed data
├── config.py               # Cấu hình (giá, hằng số)
│
├── modules/
│   ├── user_service.py         # Thành viên 1: Tài khoản & phương tiện
│   ├── parking_service.py      # Thành viên 2: Gửi xe, lấy xe, sạc
│   └── report_service.py       # Thành viên 3: Báo cáo tháng
│
├── static/
│   ├── css/style.css
│   └── js/main.js
│
└── templates/
    ├── base.html
    ├── landing.html           # Trang chủ giới thiệu
    ├── auth/
    │   ├── login.html
    │   └── register.html
    ├── user/
    │   ├── dashboard.html      # Tổng quan cá nhân
    │   ├── vehicles.html       # Quản lý phương tiện
    │   ├── park.html           # Gửi xe
    │   ├── checkout.html       # Lấy xe
    │   ├── charge.html         # Thuê sạc
    │   └── history.html        # Lịch sử giao dịch
    ├── admin/
    │   ├── dashboard.html      # Tổng quan vận hành
    │   ├── slots.html          # Quản lý vị trí đỗ
    │   ├── stations.html       # Quản lý trụ sạc
    │   ├── parking_orders.html # Đơn gửi xe active
    │   └── charging_orders.html# Đơn sạc active
    └── director/
        └── report.html         # Báo cáo tháng (chart.js)
```

---

## Proposed Changes

### Database Schema (`database.py`)

#### Bảng dữ liệu

| Bảng | Mô tả |
|------|-------|
| `users` | id, full_name, phone, email, password_hash, role (user/admin/director), created_at |
| `vehicles` | id, user_id, plate_number, vehicle_type (motorcycle/car/e_motorcycle/e_car), brand, model, color, battery_capacity, created_at |
| `parking_slots` | id, slot_code, slot_type (motorcycle/car/both), floor_area, status (available/occupied/reserved/maintenance), notes |
| `charging_stations` | id, station_code, station_type (slow/fast), power_kw, status (available/busy/maintenance), area |
| `parking_orders` | id, user_id, vehicle_id, slot_id, time_in, time_out, status (active/completed/cancelled), unit_price, total_fee, notes |
| `charging_orders` | id, user_id, vehicle_id, station_id, charge_type (slow/fast), time_start, time_end, kwh_consumed, total_fee, status (active/completed) |
| `payments` | id, order_type (parking/charging), order_id, amount, paid_at, method |

---

### Module 1 — `user_service.py` (Thành viên 1)

**Chức năng:**
- `register_user(full_name, phone, email, password)` → validate & hash password → insert users
- `login(email_or_phone, password)` → xác thực → trả về user + role
- `update_profile(user_id, ...)` → cập nhật thông tin cá nhân
- `add_vehicle(user_id, plate, type, ...)` → kiểm tra biển số unique toàn hệ thống
- `update_vehicle(vehicle_id, user_id, ...)` → chỉ chủ xe được sửa
- `delete_vehicle(vehicle_id, user_id)` → kiểm tra không có đơn active
- `get_vehicles(user_id)` → danh sách xe của user

---

### Module 2 — `parking_service.py` (Thành viên 2)

**Gửi xe:**
- `get_available_slots(vehicle_type)` → lọc slot theo loại xe
- `create_parking_order(user_id, vehicle_id, slot_id)` → kiểm tra xe chưa active, slot available
- `checkout(order_id, user_id)` → tính phí, update slot → available, tạo payment

**Lấy xe — Bảng giá:**
| Loại xe | Giá/giờ | Tối đa/ngày |
|---------|---------|-------------|
| Xe máy | 5.000đ | 25.000đ |
| Ô tô | 20.000đ | 120.000đ |
| Xe máy điện | 8.000đ | 40.000đ |
| Ô tô điện | 25.000đ | 150.000đ |

**Sạc:**
- `get_available_stations(charge_type)` → lọc trụ theo loại
- `create_charging_order(user_id, vehicle_id, station_id, charge_type)`
- `end_charging(order_id, kwh)` → tính phí sạc, update station → available

**Giá sạc:**
- Phí giữ chỗ: 10.000đ/lượt
- Sạc chậm: 8.000đ/kWh
- Sạc nhanh: 12.000đ/kWh

**Admin:**
- `get_active_parking_orders()` — xe đang trong bãi
- `get_active_charging_orders()` — trụ đang sạc
- `admin_confirm_checkout(order_id)` — xác nhận xe ra
- `update_slot_status(slot_id, status)`
- `update_station_status(station_id, status)`

---

### Module 3 — `report_service.py` (Thành viên 3)

**Hàm báo cáo:**
- `get_monthly_revenue(year, month)` → tổng doanh thu, chia parking/charging, theo ngày
- `get_monthly_activity(year, month)` → số lượt gửi xe, lấy xe, sạc
- `get_customer_stats(year, month)` → khách mới, khách quay lại, top customer
- `get_occupancy_stats(year, month)` → tỉ lệ lấp đầy, giờ cao điểm, slot hot nhất
- `compare_with_previous_month(year, month)` → % thay đổi so với tháng trước

---

### UI / Frontend

**Giao diện sẽ dùng:**
- Dark mode gradient design
- Chart.js cho biểu đồ báo cáo của Tổng giám đốc
- Responsive layout với CSS Grid/Flexbox
- Micro-animations và hover effects

**Trang theo vai trò:**

| Role | Trang |
|------|-------|
| Guest | Landing, Login, Register |
| User | Dashboard, Vehicles, Park, Checkout, Charge, History |
| Admin | Dashboard, Slots, Stations, Parking Orders, Charging Orders |
| Director | Report (với Chart.js) |

---

## Verification Plan

### Automated
- Chạy `python app.py` → server khởi động tại `localhost:5000`
- Seed data tự động tạo tài khoản demo:
  - **User**: `user@demo.com` / `123456`
  - **Admin**: `admin@demo.com` / `123456`
  - **Director**: `director@demo.com` / `123456`

### Manual
- Đăng nhập thử 3 vai trò khác nhau
- Thử flow: Gửi xe → Lấy xe → Kiểm tra lịch sử
- Thử flow: Sạc → Kết thúc sạc → Kiểm tra hóa đơn
- Xem báo cáo tháng với biểu đồ

---

## Open Questions

> [!IMPORTANT]
> Bạn có muốn dùng **MySQL** thay SQLite không? (SQLite chạy ngay, MySQL cần cài đặt server)

> [!NOTE]
> Hệ thống sẽ có **seed data mẫu** (5–10 vị trí đỗ, 4 trụ sạc, vài đơn hàng lịch sử) để báo cáo có số liệu ngay từ đầu — bạn có đồng ý không?
