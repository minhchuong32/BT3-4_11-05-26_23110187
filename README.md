# FlashLearn System Logic

## Giới thiệu

Đây là một ứng dụng full-stack tập trung vào luồng xác thực người dùng và quản lý hồ sơ cá nhân.

- Backend: Node.js, Express, MongoDB, JWT, cookie httpOnly
- Frontend: React, Vite, Redux Toolkit, React Router, Axios, Tailwind CSS

Hệ thống hỗ trợ các chức năng chính:

- Đăng ký tài khoản
- Đăng nhập bằng email hoặc username
- Đăng nhập bằng Google
- Xem và chỉnh sửa hồ sơ cá nhân
- Quên mật khẩu bằng email và OTP/token
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
- cookie-parser để đọc JWT từ cookie

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

## Luồng hệ thống

### 1. Khởi động ứng dụng

1. `BE/src/main.js` kết nối MongoDB, cấu hình CORS, cookie parser và routes.
2. `FE/src/main.jsx` mount ứng dụng React.
3. `FE/src/App.jsx` cấu hình router và Redux store cho toàn bộ màn hình.
4. Axios ở frontend tự gắn `Authorization` nếu trong `localStorage` có token, đồng thời gửi cookie nhờ `withCredentials`.

### 2. Đăng ký tài khoản

1. Người dùng nhập `username`, `email`, `password` ở trang đăng ký.
2. Frontend gọi `POST /api/auth/register`.
3. Backend kiểm tra email hoặc username đã tồn tại chưa.
4. Nếu hợp lệ, mật khẩu được hash bằng `bcryptjs`.
5. User mới được tạo với role mặc định là `user`.
6. Frontend hiển thị thông báo thành công rồi chuyển sang trang đăng nhập.

Ý nghĩa: đăng ký chỉ tạo tài khoản mới, chưa tự đăng nhập.

### 3. Đăng nhập thường

1. Người dùng nhập `email` hoặc `username` cùng mật khẩu.
2. Frontend gọi `POST /api/auth/login`.
3. `rateLimiter` và `validateLogin` kiểm tra sớm dữ liệu đầu vào và giới hạn tần suất.
4. Backend tra user trong MongoDB và so sánh mật khẩu bằng `bcryptjs`.
5. Nếu đúng, service tạo `accessToken` 15 phút và `refreshToken` 7 ngày.
6. Controller set hai token vào cookie httpOnly là `jwt` và `refreshToken`.
7. Backend trả về `redirect_url` để frontend điều hướng sau đăng nhập.

Ý nghĩa: phiên đăng nhập chính của hệ thống nằm trong cookie httpOnly, không phụ thuộc vào việc user nhìn thấy token ở giao diện.

### 4. Đăng nhập Google

1. Trang login tải Google Identity script và hiển thị nút đăng nhập Google.
2. Khi người dùng chọn tài khoản Google, frontend nhận `idToken`.
3. Frontend gọi `POST /api/auth/google`.
4. Backend dùng `google-auth-library` để xác minh token với `GOOGLE_CLIENT_ID`.
5. Nếu user chưa tồn tại, hệ thống tạo user mới với `googleId`, `isVerified = true` và profile cơ bản.
6. Nếu user đã có email nhưng chưa liên kết Google, backend bổ sung `googleId` vào tài khoản đó.
7. Backend tạo lại access token/refresh token và set cookie giống luồng đăng nhập thường.

Ý nghĩa: Google login dùng chung cơ chế phiên làm việc với đăng nhập thường.

### 5. Khôi phục phiên làm việc

1. FE lưu trạng thái auth trong Redux.
2. Khi vào trang protected như `home` hoặc `profile`, app gọi API profile để lấy lại dữ liệu người dùng.
3. Request kèm cookie `jwt` nên backend có thể xác thực mà không cần người dùng nhập lại mật khẩu.
4. Nếu token còn hợp lệ, UI hiển thị trang đăng nhập thành công; nếu không thì chuyển về login.

Ý nghĩa: backend là nguồn xác thực chính, frontend chỉ giữ trạng thái hiển thị và điều hướng.

### 6. Xem hồ sơ cá nhân

1. Người dùng đã đăng nhập vào trang `/profile`.
2. FE gọi `GET /user/profile`.
3. Middleware `verifyToken` đọc cookie `jwt` và gán `req.user`.
4. `profileController.userProfile` lấy user từ MongoDB theo `req.user.id` và loại bỏ `password`, `otp` khỏi dữ liệu trả về.
5. FE hiển thị `fullName`, `bio`, `phoneNumber`, `avatarUrl`, email và role.

### 7. Cập nhật hồ sơ

1. Người dùng bật chế độ chỉnh sửa trên trang profile.
2. FE cho phép sửa các trường `fullName`, `bio`, `phoneNumber`, `avatarUrl`.
3. Khi lưu, FE gọi `PUT /user/profile`.
4. Backend kiểm tra token, tìm user hiện tại rồi ghi đè từng trường vào `profile`.
5. Sau khi lưu thành công, FE cập nhật lại state và tắt chế độ chỉnh sửa.

Ý nghĩa: cập nhật profile chỉ tác động vào phần hồ sơ, không đụng tới mật khẩu hay role.

### 8. Quyền truy cập user/admin

1. Middleware `verifyToken` xác thực token trước.
2. Middleware `authorizeRole` kiểm tra role của user.
3. Route `/admin/profile` chỉ cho role `admin` đi vào.
4. Nếu role không đúng, backend trả về lỗi `403 Forbidden`.

Ý nghĩa: backend chặn quyền ở tầng API, không phụ thuộc hoàn toàn vào UI.

### 9. Quên mật khẩu

1. Người dùng nhập email ở trang quên mật khẩu.
2. Frontend gọi `POST /api/auth/forgot-password`.
3. Backend kiểm tra email có tồn tại không.
4. Nếu user hợp lệ, backend sinh chuỗi OTP/token ngẫu nhiên, lưu vào `user.otp.code` và `user.otp.expiresAt`.
5. Backend trả về thông báo đã gửi email khôi phục.
6. Trang FE hiện tại đã có API gọi quên mật khẩu, còn màn reset password hoàn chỉnh chưa được gắn vào router.

Ý nghĩa: dữ liệu OTP chỉ sống tạm thời, có thời gian hết hạn rõ ràng.

### 10. Đặt lại mật khẩu

1. Backend có sẵn endpoint `POST /api/auth/reset-password`.
2. Endpoint này nhận `token` và `newPassword`.
3. Backend tìm user có `otp.code` khớp token và `otp.expiresAt`  còn hiệu lực.
4. Nếu hợp lệ, mật khẩu mới được hash rồi lưu lại.
5. OTP cũ bị xóa khỏi tài khoản sau khi đổi mật khẩu thành công.

Ghi chú: phần API đã sẵn sàng, nhưng màn FE cho reset password hiện còn thiếu trong router chính.

### 11. Đăng xuất

1. Người dùng bấm nút đăng xuất trên navbar hoặc trong trang home/profile.
2. Frontend gọi action `logout`.
3. Redux xóa trạng thái auth.
4. `access_token` trong `localStorage` bị xóa nếu có.
5. Người dùng được chuyển về trang đăng nhập.

## Route chính

### Backend

| Method | Route                       | Mô tả                             |
| ------ | --------------------------- | --------------------------------- |
| `GET`  | `/login`                    | Render view EJS trang login       |
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

- Giới hạn tối đa 10 lần đăng nhập trong 15 phút
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
