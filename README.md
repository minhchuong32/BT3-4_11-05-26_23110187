# FlashLearn System Logic

## Giới thiệu

Đây là một ứng dụng full-stack tập trung vào xác thực người dùng, quản lý hồ sơ và phân quyền truy cập.

- Backend: Node.js, Express, MongoDB, JWT, cookie httpOnly
- Frontend: React, Vite, Redux Toolkit, React Router, Axios, Tailwind CSS

Hệ thống hỗ trợ các chức năng chính:

- Đăng ký tài khoản
- Đăng nhập bằng username hoặc email
- Đăng nhập bằng Google
- Xem và cập nhật hồ sơ cá nhân
- Quên mật khẩu bằng email và token tạm thời
- Phân quyền user/admin ở backend

## Cấu trúc dự án

- `BE/`: API server, middleware, service, model và seed dữ liệu
- `FE/`: giao diện người dùng và luồng thao tác phía client

## Công nghệ sử dụng

### Backend

- ExpressJS để xây dựng API
- MongoDB + Mongoose để lưu người dùng
- bcryptjs để mã hóa mật khẩu
- jsonwebtoken để tạo access token và refresh token
- google-auth-library để xác thực Google ID token
- express-rate-limit để giới hạn số lần đăng nhập

### Frontend

- React 19 + Vite
- React Router DOM để điều hướng trang
- Redux Toolkit để quản lý trạng thái auth
- Axios để gọi API
- Tailwind CSS để dựng giao diện

## Logic dữ liệu người dùng

Schema người dùng ở backend có các trường chính:

- `username`: tên đăng nhập duy nhất
- `email`: email duy nhất, dùng cho đăng ký và quên mật khẩu
- `password`: mật khẩu đã mã hóa
- `googleId`: mã Google khi đăng nhập bằng Google
- `profile.fullName`, `profile.avatarUrl`, `profile.bio`, `profile.phoneNumber`: thông tin hồ sơ
- `isVerified`: trạng thái xác thực tài khoản
- `otp.code`, `otp.expiresAt`: dữ liệu tạm cho khôi phục mật khẩu
- `role`: `user` hoặc `admin`

## Luồng hoạt động của hệ thống

### 1. Khởi động ứng dụng

1. `BE/src/main.js` cấu hình Express, CORS, view engine và nạp route API.
2. `BE/src/routes/authRoutes.js` gắn nhóm route xác thực vào prefix `/api/auth`.
3. `BE/src/routes/api.js` gắn nhóm route hồ sơ vào prefix `/`.
4. `FE/src/main.jsx` khởi tạo React app và Redux store.
5. `FE/src/App.jsx` khai báo router và chặn các trang cần đăng nhập.

Ý nghĩa: backend chịu trách nhiệm xác thực và trả dữ liệu, frontend chỉ điều hướng và hiển thị trạng thái.

### 2. Đăng ký tài khoản

1. Người dùng nhập `username`, `email`, `password` ở trang đăng ký.
2. Frontend gọi `POST /api/auth/register`.
3. Backend kiểm tra dữ liệu trống, email trùng hoặc username trùng.
4. Nếu hợp lệ, mật khẩu được hash bằng `bcryptjs`.
5. User mới được tạo với role mặc định là `user`.
6. Frontend hiển thị thông báo thành công và chuyển sang trang đăng nhập.

Ý nghĩa: đăng ký chỉ tạo tài khoản mới, chưa tự đăng nhập.

### 3. Đăng nhập thường

1. Người dùng nhập `identifier` và `password` ở trang login.
2. Frontend gọi `POST /api/auth/login`.
3. Middleware `loginLimiter` giới hạn số lần thử đăng nhập.
4. Middleware `validateLogin` chặn request thiếu dữ liệu hoặc sai định dạng.
5. Backend tìm user theo username hoặc email và so sánh mật khẩu bằng `bcryptjs`.
6. Nếu đúng, service tạo `accessToken` và `refreshToken`.
7. Controller set hai token vào cookie httpOnly là `jwt` và `refreshToken`.
8. Backend trả về `redirect_url` để frontend điều hướng sau khi đăng nhập.

Ý nghĩa: phiên đăng nhập chính của hệ thống nằm trong cookie httpOnly, không phụ thuộc vào việc user nhìn thấy token trên giao diện.

### 4. Đăng nhập bằng Google

1. Người dùng chọn đăng nhập bằng Google trên trang login.
2. Frontend nhận `idToken` từ Google Identity.
3. Frontend gọi `POST /api/auth/google`.
4. Backend xác minh token Google bằng `GOOGLE_CLIENT_ID`.
5. Nếu user chưa tồn tại, hệ thống tạo tài khoản mới và gắn `googleId`.
6. Nếu user đã có email phù hợp, backend liên kết `googleId` vào tài khoản hiện tại.
7. Backend tạo token phiên làm việc mới và set cookie giống luồng đăng nhập thường.

Ý nghĩa: Google login dùng chung cơ chế phiên làm việc với đăng nhập nội bộ.

### 5. Khôi phục phiên làm việc

1. Sau khi app mount, frontend kiểm tra trạng thái auth trong Redux.
2. Nếu cần, app gọi API profile để lấy lại thông tin người dùng từ cookie `jwt`.
3. Backend đọc token qua middleware `verifyToken` và gán `req.user`.
4. Nếu token hợp lệ, frontend giữ trạng thái đã đăng nhập và cho vào các trang protected.
5. Nếu token không hợp lệ, user bị điều hướng về trang login.

Ý nghĩa: backend là nguồn xác thực chính, frontend chỉ giữ state hiển thị.

### 6. Xem hồ sơ cá nhân

1. Người dùng đã đăng nhập vào trang `/profile`.
2. Frontend gọi `GET /user/profile`.
3. Middleware `verifyToken` đọc cookie `jwt` và xác thực user.
4. `profileController.userProfile` lấy user từ MongoDB và loại bỏ `password`, `otp` khỏi dữ liệu trả về.
5. Frontend hiển thị `fullName`, `bio`, `phoneNumber`, `avatarUrl`, email và role.

Ý nghĩa: trang hồ sơ chỉ đọc thông tin an toàn, không trả dữ liệu nhạy cảm.

### 7. Cập nhật hồ sơ

1. Người dùng bật chế độ chỉnh sửa trên trang profile.
2. Frontend cho phép sửa `fullName`, `bio`, `phoneNumber`, `avatarUrl`.
3. Khi lưu, frontend gọi `PUT /user/profile`.
4. Backend xác thực token, tìm user hiện tại và cập nhật từng trường vào `profile`.
5. Sau khi lưu thành công, frontend cập nhật lại state và tắt chế độ chỉnh sửa.

Ý nghĩa: cập nhật profile chỉ tác động vào phần hồ sơ, không thay đổi mật khẩu hay role.

### 8. Phân quyền user/admin

1. Backend luôn kiểm tra token trước bằng `verifyToken`.
2. Với các route cần quyền cao hơn, backend gọi thêm `authorizeRole`.
3. Route `/admin/profile` chỉ cho phép role `admin` truy cập.
4. Nếu role không đúng, backend trả lỗi `403 Forbidden`.

Ý nghĩa: chặn quyền ở tầng API, không phụ thuộc hoàn toàn vào UI.

### 9. Quên mật khẩu

1. Người dùng nhập email ở trang quên mật khẩu.
2. Frontend gọi `POST /api/auth/forgot-password`.
3. Backend kiểm tra email có tồn tại không.
4. Nếu user hợp lệ, backend tạo token ngẫu nhiên bằng `crypto.randomBytes`.
5. Token được lưu tạm vào `user.otp.code` cùng thời gian hết hạn `user.otp.expiresAt`.
6. Backend trả về thông báo gửi email khôi phục thành công.

Ý nghĩa: token chỉ sống tạm thời và có thời hạn rõ ràng để giảm rủi ro bảo mật.

### 10. Đặt lại mật khẩu

1. Người dùng gửi token và mật khẩu mới qua `POST /api/auth/reset-password`.
2. Backend tìm user có `otp.code` khớp token và còn hiệu lực.
3. Nếu hợp lệ, mật khẩu mới được hash rồi lưu lại.
4. OTP cũ bị xóa khỏi tài khoản sau khi đổi mật khẩu thành công.

Ý nghĩa: mật khẩu chỉ được cập nhật sau khi token khôi phục được xác thực thành công.

### 11. Đăng xuất

1. Người dùng bấm nút đăng xuất trên giao diện.
2. Frontend xóa state auth trong Redux.
3. Phiên làm việc ở phía client được dọn theo logic ứng dụng.
4. Người dùng được chuyển về trang login.

## Route chính

### Backend

| Method | Route                       | Mô tả                             |
| ------ | --------------------------- | --------------------------------- |
| `GET`  | `/api/auth/login`           | Render view EJS trang login       |
| `POST` | `/api/auth/register`        | Đăng ký tài khoản                 |
| `POST` | `/api/auth/login`           | Đăng nhập thường                  |
| `POST` | `/api/auth/google`          | Đăng nhập bằng Google             |
| `POST` | `/api/auth/forgot-password` | Khởi tạo luồng khôi phục mật khẩu |
| `POST` | `/api/auth/reset-password`  | Đặt lại mật khẩu                  |
| `GET`  | `/user/profile`             | Lấy profile người dùng            |
| `PUT`  | `/user/profile`             | Cập nhật profile người dùng       |
| `GET`  | `/admin/profile`            | Lấy profile admin                 |

### Frontend

| Route              | Mô tả                                |
| ------------------ | ------------------------------------ |
| `/login`           | Trang đăng nhập thường và Google     |
| `/register`        | Trang đăng ký                        |
| `/forgot-password` | Trang gửi yêu cầu khôi phục mật khẩu |
| `/home`            | Trang chào mừng sau đăng nhập        |
| `/profile`         | Trang xem và chỉnh sửa hồ sơ         |

## Middleware quan trọng

### `verifyToken`

- Đọc JWT từ cookie `jwt`
- Giải mã token bằng `JWT_SECRET`
- Gán thông tin user vào `req.user`
- Trả lỗi nếu token thiếu, sai hoặc hết hạn

### `authorizeRole`

- Nhận danh sách role được phép
- Chỉ cho request đi tiếp nếu role của user nằm trong danh sách đó

### `loginLimiter`

- Giới hạn số lần đăng nhập trong một khoảng thời gian ngắn
- Dùng để giảm brute-force và spam login

### `validateLogin`

- Kiểm tra request login có đủ `identifier` và `password`
- Chặn sớm dữ liệu rỗng hoặc sai định dạng

## Cấu hình môi trường

### Backend `BE/.env`

Các biến thường dùng:

- `PORT`
- `MONGODB_URI`
- `JWT_SECRET`
- `REFRESH_SECRET`
- `GOOGLE_CLIENT_ID`

### Frontend `FE/.env.development`

- `VITE_BACKEND_URL`
- `VITE_GOOGLE_CLIENT_ID`

## Chạy dự án

### Backend

```bash
cd BE
npm install
npm run dev
```

### Frontend

```bash
cd FE
npm install
npm run dev
```

## Dữ liệu mẫu

File `BE/src/seeders/userSeeder.js` tạo sẵn dữ liệu test:

- một tài khoản admin
- một tài khoản user

Mật khẩu mẫu đang dùng là `123456`.

## Ghi chú triển khai

- Backend đặt JWT vào cookie httpOnly để tăng mức an toàn so với lưu token thuần trong JS.
- Axios frontend bật `withCredentials` để gửi cookie lên backend.
- Trang profile là luồng chính sau đăng nhập, còn trang home chủ yếu dùng để chào mừng và điều hướng nhanh.
- Màn reset password ở frontend chưa được hoàn thiện, nhưng API backend đã có sẵn để triển khai tiếp.
