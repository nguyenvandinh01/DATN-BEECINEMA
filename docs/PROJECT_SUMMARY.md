# BEE CINEMA - Project Documentation

> **Hệ thống quản lý rạp chiếu phim** - Laravel RESTful API
> *Tự động tạo từ phân tích codebase - 2026-11-13*

---

## 📋 Tổng quan

**BEE CINEMA** là hệ thống quản lý rạp chiếu phim full-feature được xây dựng trên **Laravel 10+** với kiến trúc RESTful API, hỗ trợ realtime, đa phương thức thanh toán, hệ thống thành viên phân cấp và báo cáo thống kê chi tiết.

---

## 🏗️ Kiến trúc & Công nghệ

| Thành phần | Công nghệ / Thư viện |
|------------|---------------------|
| **Framework** | Laravel 10+ |
| **Database** | MySQL (Eloquent ORM, Migrations, Seeders) |
| **Authentication** | Laravel Sanctum (API Token) |
| **Authorization** | Spatie Laravel Permission (RBAC) |
| **Realtime** | Laravel Broadcasting (Pusher/Soketi) |
| **Queue/Jobs** | Laravel Queue (Redis/Database driver) |
| **Payment Gateway** | VNPAY, ZaloPay, MoMo |
| **Email** | Laravel Mail + Queue |
| **Slug/SEO** | Cviebrock Eloquent Sluggable |
| **Validation** | Form Requests, Custom Rules |

---

## 🗃️ Database Schema (27 Models)

### Phim & Suất chiếu
| Model | Bảng | Mô tả |
|-------|------|-------|
| `Movie` | `movies` | Phim chính (thông tin, poster, trailer, rating, duration) |
| `MovieVersion` | `movie_versions` | Phiên bản: Phụ đề / Lồng tiếng / Thuyết minh |
| `Showtime` | `showtimes` | Suất chiếu: cinema, room, movie, version, date, time, format |
| `SeatShowtime` | `seat_showtimes` | Ghế theo suất chiếu (pivot: status, price, user_id, hold_expires_at) |

### Rạp & Phòng chiếu
| Model | Bảng | Mô tả |
|-------|------|-------|
| `Branch` | `branches` | Chi nhánh rạp (brand) |
| `Cinema` | `cinemas` | Rạp cụ thể (thuộc branch, address, surcharge) |
| `Room` | `rooms` | Phòng chiếu (cinema, type_room, seat_template) |
| `TypeRoom` | `type_rooms` | Loại phòng: 2D, 3D, IMAX, 4DX... |
| `Seat` | `seats` | Ghế vật lý (room, type_seat, coordinates x/y, name) |
| `SeatTemplate` | `seat_templates` | Template sơ đồ ghế (matrix layout) |
| `TypeSeat` | `type_seats` | Loại ghế: Standard, VIP, Couple, Sweetbox (giá cơ sở) |

### Người dùng & Phân quyền
| Model | Bảng | Mô tả |
|-------|------|-------|
| `User` | `users` | Người dùng (Sanctum, HasRoles, SoftDeletes) |
| `Role` / `Permission` | `roles`/`permissions` | Spatie RBAC |
| `Membership` | `memberships` | Thông tin thành viên (total_spent, rank_id, points, code) |
| `Rank` | `ranks` | Hạng: Member, Silver, Gold, Platinum (ticket/combo %, total_spent min) |

### Vé & Đặt chỗ
| Model | Bảng | Mô tả |
|-------|------|-------|
| `Ticket` | `tickets` | Vé (user, cinema, room, movie, showtime, prices, discounts, points, status) |
| `Ticket_Seat` | `ticket_seats` | Ghế trong vé (ticket, seat, price) |
| `Ticket_Combo` | `ticket_combos` | Combo trong vé (ticket, combo, quantity, price) |

### Đồ ăn & Combo
| Model | Bảng | Mô tả |
|-------|------|-------|
| `Food` | `foods` | Món ăn đơn lẻ |
| `Combo` | `combos` | Combo (foods + quantity, discount_price) |
| `ComboFood` | `combo_food` | Pivot combo-food (quantity) |

### Khuyến mãi & Điểm
| Model | Bảng | Mô tả |
|-------|------|-------|
| `Voucher` | `vouchers` | Voucher (code, type: percent/fixed, discount, quantity, expiry) |
| `UserVoucher` | `user_vouchers` | Voucher của user (usage_count, ticket_id) |
| `PointHistory` | `point_histories` | Lịch sử điểm (membership, points, type, expired_at) |

### Khác
| Model | Bảng | Mô tả |
|-------|------|-------|
| `Banner` | `banners` | Banner quảng cáo (carousel) |
| `Post` | `posts` | Bài viết/tin tức |
| `MovieReview` | `movie_reviews` | Đánh giá phim (user, movie, rating, comment) |
| `Contact` | `contacts` | Liên hệ/hỗ trợ |
| `SpecialDay` | `special_days` | Ngày đặc biệt (giảm giá combo cuối tuần, 2/9...) |
| `PriceRule` | `price_rules` | Quy tắc giá động (cinema, type_room, type_seat, day_type, multiplier) |
| `RevenueReport` | `revenue_reports` | Báo cáo doanh thu ngày |

---

## 🎯 Chức năng chính theo Module

### 1. 🎬 Quản lý Phim (`MovieController`)
- **CRUD** phim với validation (`StoreMovieRequest`, `UpdateMovieRequest`)
- Quản lý **versions** (Phụ đề/Lồng tiếng/Thuyết minh)
- Trạng thái: `is_active`, `is_hot`, `is_publish`, `is_special`
- Phân loại client: **Đang chiếu** / **Sắp chiếu** / **Đặc biệt** (`moviesTabPageClient`)
- Slug tự động từ tên phim (Eloquent Sluggable)
- Không cho sửa duration/dates khi đã `is_publish`

### 2. 🎟️ Quản lý Suất chiếu (`ShowtimeController`)
- **Tạo suất chiếu**:
  - Tự động theo khung giờ (`auto_generate_showtimes`)
  - Thủ công từng suất
  - Preview trước khi tạo (`previewShowtimes`)
  - Copy nhiều ngày (`copyShowtime`)
- **Quản lý ghế** (`SeatShowtime`): Tự động tạo khi có suất chiếu mới (Job `CreateSeatShowtimesJob`)
- **Trạng thái ghế**: `available` → `hold` (15p) → `booked`
- **Realtime**: Broadcast `SeatStatusChange` khi ghế thay đổi
- **Tính giá động**: `PriceCalculationService` + `PriceRule` + `SpecialDay`
- **Lọc suất chiếu**: Theo cinema, room, date, branch (`index`, `pageShowtime`, `showtimeMovie`, `listShowtimesByDate`)

### 3. 🪑 Chọn ghế & Giữ ghế (`ChooseSeatController`)
- `GET /choose-seat/{slug}` - Lấy seat map, matrix, trạng thái realtime
- `POST /update-seat` - Hold/Release ghế (cập nhật `hold_expires_at`, `user_id`)
- `POST /save-information/{id}` - Lưu thông tin khách trước thanh toán
- `GET /userHoldSeats/{slug}` - Lấy ghế user đang hold
- `POST /updateSeatHoldtime` - Gia hạn thời gian hold (thanh toán online)

### 4. 💳 Thanh toán (`PaymentController`)
| Phương thức | Flow |
|-------------|------|
| **VNPAY** | Tạo URL → Redirect → Callback `returnVnpay` → Tạo ticket |
| **ZaloPay** | Tạo order → Callback `zalopayCallback` → Tạo ticket |
| **MoMo** | Tạo payUrl → IPN `paymentIpn` → Tạo ticket |
| **Tiền mặt** | `paymentOffline` - Tạo ticket ngay tại quầy |

**Xử lý chung:**
- Validate seat status = `hold` + `user_id` match + chưa hết hạn
- Tính giá: seat + combo - discount(rank, voucher, point, special day)
- Cache payment data 60p (`payment_{orderCode}`)
- Job `ReleaseSeatHoldJob` giải phóng ghế sau 15p
- Job `CancelVoucherJob` hủy voucher nếu fail
- Email ticket qua `SendTicketEmail` job

### 5. 🎫 Quản lý Vé (`TicketController`)
- `store` - Tạo vé (auth user) với seats, combos, voucher, points
- `getBookingHistory` - Lịch sử vé user (with seats, combos, movie, showtime)
- `show($code)` - Chi tiết vé theo code (public)
- `confirm` - Xác nhận "Đã xuất vé" tại quầy (staff)
- `filter` - Lọc vé admin (branch, cinema, movie, date, status)
- **Tích điểm**: 2500đ/ghế + 3% total_price
- **Cập nhật rank**: Tự động dựa trên `total_spent`

### 6. ⭐ Hệ thống Thành viên (`Membership`, `Rank`, `PointController`)
| Hạng | `total_spent` min | Ticket % | Combo % |
|------|------------------|----------|---------|
| Member | 0 | 0% | 0% |
| Silver | 1,000,000 | 5% | 5% |
| Gold | 5,000,000 | 10% | 10% |
| Platinum | 20,000,000 | 15% | 15% |

- **Tích điểm**: Mua vé (3%), mua combo, cuối tuần x2
- **Trừ điểm**: 1 điểm = 1 VND (tối đa total_price)
- **Hết hạn**: 12 tháng (`PointHistory::POINT_EXPIRY_DURATION`)
- **API**: `GET /points/available`, `POST /get-points` (by membership code)

### 7. 🎁 Voucher (`VoucherApiController`)
- Loại: `percent` / `fixed`
- Giới hạn lượng, ngày hết hạn, user chỉ dùng 1 lần
- `applyVoucher` / `removeVoucher` (validate realtime)
- Tự động `usage_count++` khi thanh toán thành công

### 8. 📊 Báo cáo & Thống kê (`ReportController`, `OverviewController`)
| Endpoint | Mô tả |
|----------|-------|
| `/revenue-by-combo` | Doanh thu theo combo |
| `/revenue-by-food` | Doanh thu theo món ăn |
| `/revenue-by-total` | Tổng doanh thu (filter date, cinema) |
| `/revenue-ticket-statistics` | Lượt đặt vé, tỷ lệ hủy |
| `/customer` | Thống kê khách hàng (mới, quay lại, hạng) |
| `/booking-trends` | Xu hướng đặt vé theo thời gian |
| `/overview` | Dashboard cards (doanh thu, vé, khách, ghế) |
| `/seatOccupancyByDay` | % lấp đầy ghế theo ngày |
| `/seatOccupancyByMonth` | % lấp đầy ghế theo tháng |
| `/dashboard` | Tổng quan admin |

### 9. ⚙️ Quản trị (Admin / Admin Cinema)
| Resource | Roles | Endpoints |
|----------|-------|-----------|
| Users | admin | CRUD, change password, restore, force delete |
| Branch/Cinema/Room | admin | CRUD |
| TypeRoom/TypeSeat/SeatTemplate | admin | CRUD |
| Food/Combo | admin_cinema | CRUD |
| Movie/Showtime | admin | CRUD + special actions |
| Banner/Post/Contact | admin | CRUD |
| Rank/Voucher/SpecialDay/PriceRule | admin | CRUD |
| Permission/Role | admin | CRUD (Spatie) |

---

## 🔐 Phân quyền (Roles & Permissions)

```php
// User.php constants
TYPE_ADMIN = 'admin';           // Toàn quyền
TYPE_MANAGER = 'manager';       // Quản lý rạp (admin_cinema role)
TYPE_MEMBER = 'member';         // Khách hàng

// Spatie Roles
admin          // Super admin
admin_cinema   // Quản lý rạp (phim, suất chiếu, combo, food, room)
staff          // Bán vé, xác nhận vé
member         // User thường
```

**Middleware:**
- `auth:sanctum` - Xác thực API token
- `role:admin|admin_cinema` - Quản trị rạp
- `role:admin` - Super admin only

---

## 📡 API Routes Map

### Public (No Auth)
```
GET  /movies                 # Danh sách phim (admin)
GET  /movies/tab             # Phim client (showing/upcoming/special)
GET  /movies/{movie}         # Chi tiết phim
GET  /showtimespage          # Lịch chiếu theo rạp (client)
GET  /showtimemovie          # Lịch chiếu theo phim (client)
GET  /showtimes/slug/{slug}  # Chi tiết suất chiếu + seat map
GET  /choose-seat/{slug}     # Trang chọn ghế
GET  /cinemas, /cinemas/{id}
GET  /branches, /branches/{id}
GET  /foods, /foods/{id}
GET  /combos, /combos/{id}
GET  /banners, /banners/active
GET  /posts, /posts/{id}
GET  /ranks, /ranks/{id}
GET  /seat-templates, /seat-templates/{id}
GET  /rooms, /rooms/{id}
GET  /movie-reviews
GET  /contact
GET  /vouchers               # Danh sách voucher public
GET  /tickets/filter         # Lọc vé (admin)
GET  /tickets/{code}         # Chi tiết vé theo code
GET  /revenue-*, /overview, /dashboard, /seatOccupancy*
POST /register, /login, /forgot-password, /reset-password
POST /verify/email/{id}/{hash}
GET  /auth/google, /auth/google/callback
POST /zalopay/callback, GET /vnpay-return, POST /momo-ipn
GET  /handleZalopayRedirect
```

### Auth Required (`auth:sanctum`)
```
GET  /user                   # Profile user
PUT  /users/update           # Cập nhật profile
POST /logout
POST /email/resend
POST /change-password
GET  /profile
GET  /user/membership
GET  /user/vouchers
GET  /points/available
POST /tickets                # Đặt vé
GET  /booking-history
POST /tickets/confirm        # Xác nhận vé (staff)
POST /update-seat            # Hold/Release ghế
POST /save-information/{id}
GET  /userHoldSeats/{slug}
POST /updateSeatHoldtime
POST /payment                # VNPAY/ZaloPay/MoMo
POST /payment/offline        # Tiền mặt
GET  /vnpay-payment
POST /zalopay/payment
POST /momo-payment
POST /movie-reviews          # Đánh giá phim
POST /get-points             # Check điểm by membership code
POST /vouchers/apply-voucher
POST /vouchers/remove-voucher
```

### Admin Only (`role:admin`)
```
POST   /movies, PUT/PATCH /movies/{id}, DELETE /movies/{id}
POST   /movies/update-active, /movies/update-hot
POST   /showtimes (store), PUT/PATCH /showtimes/{id}, DELETE /showtimes/{id}
POST   /showtimes/add-showtime-per-day
GET    /showtimes, /showtimes/{id}, /listshowtimesdate
POST   /showtimes/{id}/copy
POST   /banners, /branches, /cinemas, /ranks
POST   /permission, /roles
POST   /price-rules, /special-days
```

### Admin Cinema (`role:admin|admin_cinema`)
```
POST   /foods, /combos, /seat-templates, /rooms
POST   /type-rooms, /type-seats
PUT/PATCH /showtimes/{id}
```

---

## ⚙️ Jobs & Queue (Background Processing)

| Job | Trigger | Mô tả |
|-----|---------|-------|
| `CreateSeatShowtimesJob` | Tạo suất chiếu mới | Tạo hàng nghìn record `seat_showtimes` (chunk 500) |
| `ReleaseSeatHoldJob` | Hold ghế (delay 15p) | Giải phóng ghế nếu không thanh toán |
| `BroadcastSeatStatusChange` | Ghế thay đổi status | Realtime push tới client (Pusher/Soketi) |
| `SendTicketEmail` | Thanh toán thành công | Gửi email vé + QR code (queue: emails) |
| `CancelVoucherJob` | Thanh toán fail/timeout | Hủy voucher, trả quantity |
| `ApplyVoucherJob` | Áp voucher | Xử lý logic voucher phức tạp |

**Queue config:** `config/queue.php` - Redis/Database driver, queues: `default`, `seat_showtimes`, `emails`

---

## 💰 Logic Tính Giá (`PriceCalculationService`)

```php
// Base price từ TypeSeat->price
$basePrice = $seat->typeSeat->price;

// Áp dụng PriceRule (cinema, type_room, type_seat, day_type, multiplier)
$price = $basePrice * $rule->multiplier;

// SpecialDay multiplier (cuối tuần, 2/9, Tết...)
$price *= $specialDay->multiplier ?? 1;

// Rank discount (ticket_percentage)
$finalPrice = $price * (1 - $rank->ticket_percentage / 100);
```

**PriceRule fields:** `cinema_id`, `type_room_id`, `type_seat_id`, `day_type` (weekday/weekend/holiday), `multiplier`, `start_date`, `end_date`

---

## 📅 Special Day Logic

- **Weekend** (Thứ 7, CN): Combo discount theo rank `combo_percentage`
- **2/9** (Quốc khánh): Combo discount
- **Custom** (`SpecialDay` model): Ngày tự định nghĩa với multiplier

---

## 🔄 Real-time Events

```php
// Events
SeatStatusChange    // status, seat_id, showtime_id, user_id
SeatHold            // hold_expires_at, seat_id, showtime_id, user_id
SeatRelease         // seat_id, showtime_id
SeatSold            // ticket_id, seat_ids

// Broadcast channel
private-showtime.{showtime_id}  // Client listen để cập nhật UI realtime
```

---

## 🗂️ Cấu trúc thư mục quan trọng

```
app/
├── Console/Commands/           # Scheduled commands
│   ├── CalculateDailyRevenue.php
│   ├── ExpirePoints.php
│   ├── NotifyExpiringPoints.php
│   └── UpdateTicketStatus.php
├── Events/                     # Broadcasting events
├── Exceptions/Handler.php
├── Http/
│   ├── Controllers/Api/        # 30+ API Controllers
│   ├── Kernel.php              # Middleware registration
│   ├── Middleware/             # RoleMiddleware, ValidateSignature...
│   └── Requests/Api/           # Form Requests validation
├── Jobs/                       # Queue jobs
├── Models/                     # 27 Eloquent Models
├── Notifications/              # CustomVerifyEmail, ResetPasswordCustom
├── Services/
│   └── PriceCalculationService.php
└── Providers/                  # EventServiceProvider, BroadcastServiceProvider

database/
├── migrations/                 # 35+ migrations
└── seeders/

routes/
├── api.php                     # 400+ lines API routes
├── web.php
└── channels.php                # Broadcast channels

config/
├── sanctum.php
├── permission.php              # Spatie config
├── broadcasting.php
├── queue.php
└── cors.php
```

---

## 🚀 Deployment Checklist

- [ ] `.env` production: `APP_ENV=production`, `APP_DEBUG=false`
- [ ] Database: MySQL 8+, charset `utf8mb4`
- [ ] Queue worker: `php artisan queue:work --queue=default,seat_showtimes,emails --sleep=3 --tries=3`
- [ ] Scheduler: `* * * * * php artisan schedule:run` (cron)
- [ ] Broadcasting: Pusher/Soketi + Laravel Echo Server
- [ ] Storage link: `php artisan storage:link`
- [ ] Optimize: `php artisan config:cache && php artisan route:cache && php artisan view:cache`
- [ ] SSL/HTTPS cho payment callback URLs
- [ ] Payment credentials: VNPAY, ZaloPay, MoMo (sandbox → production)
- [ ] Email: SMTP (Mailgun/SendGrid/Gmail)
- [ ] Backup: Daily DB backup + storage backup

---

## 📝 Commands Hữu ích

```bash
# Development
php artisan serve
php artisan queue:work
php artisan schedule:work
npm run dev (nếu có frontend)

# Database
php artisan migrate:fresh --seed
php artisan migrate --path=database/migrations/2025_04_16_145523_create_price_rules_table.php

# Cache
php artisan config:clear && php artisan cache:clear && php artisan route:clear && php artisan view:clear

# Testing
php artisan test
php artisan test --filter=TicketTest

# Production
php artisan optimize
php artisan horizon (nếu dùng Laravel Horizon)
```

---

## 🔗 Tài liệu tham khảo

- [Laravel 10.x Docs](https://laravel.com/docs/10.x)
- [Spatie Permission](https://spatie.be/docs/laravel-permission)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Broadcasting](https://laravel.com/docs/broadcasting)
- [VNPAY Sandbox](https://sandbox.vnpayment.vn/)
- [ZaloPay Sandbox](https://docs.zalopay.vn/)
- [MoMo Sandbox](https://developers.momo.vn/)

---

## 📌 Ghi chú phát triển

1. **SeatShowtime** là bảng trung tâm - mọi thao tác ghế đều qua bảng này
2. **Cache payment data** key: `payment_{orderCode}` TTL 60 phút
3. **Slug showtime** dùng UUID (`Str::uuid()`) cho URL chọn ghế
4. **Point expiry** chạy daily qua `ExpirePoints` command
5. **Revenue report** chạy daily qua `CalculateDailyRevenue` command
6. **SoftDeletes** trên User, Ticket (có thể restore)
7. **Validation** tập trung ở Form Requests (`StoreMovieRequest`, `UpdateShowtimeRequest`...)

---

*File này được tạo tự động từ phân tích codebase. Cập nhật khi có thay đổi lớn.*