# Phần 12: Tổng quan về CSRF (Cross-Site Request Forgery)

> **Tư duy cốt lõi:**  
> *"CSRF lợi dụng cơ chế tự động gửi kèm cookie của trình duyệt. Dù các trình duyệt hiện đại đã bật SameSite=Lax làm mặc định, bạn vẫn phải tuân thủ nghiêm ngặt nguyên tắc REST: không bao giờ thay đổi trạng thái hệ thống bằng phương thức GET và cấu hình CORS chặt chẽ."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Bản chất của Tấn công CSRF
CSRF (Giả mạo yêu cầu liên trang) xảy ra khi kẻ tấn công lừa trình duyệt của nạn nhân thực hiện một hành động ngoài ý muốn trên một trang web mà nạn nhân **đã đăng nhập từ trước**.

```
[Nạn nhân đã đăng nhập vào bank.com]
Trình duyệt đang lưu Cookie xác thực của bank.com
                    │
                    ▼
[Nạn nhân truy cập vào trang web độc hại: evil.com]
Trang evil.com tự động chạy mã JavaScript gửi ngầm request:
POST https://bank.com/api/transfer
Body: { "amount": 10000, "recipient": "hacker" }
                    │
                    ▼
[TRÌNH DUYỆT TỰ ĐỘNG GỬI KÈM COOKIE CỦA bank.com!]
                    │
                    ▼
[Server bank.com thấy có Cookie hợp lệ ──► Thực hiện chuyển tiền!]
```

---

### Tại sao CSRF ít còn là mối đe dọa hàng đầu trong kỷ nguyên hiện đại?
1. **Thuộc tính `SameSite=Lax` mặc định:** Tất cả trình duyệt hiện đại (Chrome, Safari, Firefox, Edge) hiện nay đều tự động gán `SameSite=Lax` cho cookie nếu lập trình viên không chỉ định. Điều này tự động **chặn việc gửi cookie trong các POST request ngầm** xuất phát từ trang web khác (`evil.com`).
2. **Kiến trúc Token / Header (Bearer Token):** Các ứng dụng SPA (React, Vue, Flutter) thường gửi token qua header:
   ```http
   Authorization: Bearer <jwt_token>
   ```
   Trang web của hacker không có cơ chế nào để tự động nhúng header này vào request!
3. **Cơ chế CORS (Cross-Origin Resource Sharing):** Chặn các request AJAX/Fetch cross-origin không được server cấp phép.

---

### Khi nào ứng dụng của bạn VẪN CÓ NGUY CƠ bị CSRF?
- Khi bạn cấu hình cookie với cờ `sameSite: 'none'`.
- Khi bạn thiết kế API sai nguyên tắc REST: **Dùng phương thức `GET` để thay đổi dữ liệu** (ví dụ: `GET /transfer?amount=1000&to=hacker`). Thẻ `<img src="https://bank.com/transfer?amount=1000">` trên trang `evil.com` sẽ kích hoạt ngay lệnh chuyển tiền!
- Khi bạn xây dựng các ứng dụng Render HTML từ Server (SSR / MVC với NestJS + Handlebars/EJS) mà không có CSRF Token.

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong NestJS, việc phòng chống CSRF được giải quyết thông qua 3 trụ cột:
1. **Cấu hình Cookie `SameSite: 'lax'` hoặc `'strict'`.**
2. **Cấu hình CORS nghiêm ngặt trong `main.ts`.**
3. **Tích hợp Middleware Anti-CSRF Token (`csrf-csrf`)** đối với các endpoint dùng cookie để thay đổi trạng thái (`POST`, `PUT`, `DELETE`).

---

## 3. Thực hành Code NestJS: Phòng vệ CSRF Toàn diện

### 1. Cấu hình CORS và Cookie Chặt chẽ trong `main.ts`
```typescript
// main.ts - CẤU HÌNH CORS & SAMESITE CHỐNG CSRF TRONG NESTJS
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 1. CẤU HÌNH CORS CHẶT CHẼ
  app.enableCors({
    // Chỉ chấp nhận request từ domain chính thức của frontend
    origin: ['https://myfrontend.com'],
    // Cho phép gửi kèm cookie
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
  });

  await app.listen(3000);
}
bootstrap();
```

---

### 2. Tích hợp Anti-CSRF Token Middleware (Double Submit Cookie Pattern)
Sử dụng thư viện hiện đại `csrf-csrf` (thay thế cho thư viện cũ `csurf` đã bị deprecated):

```bash
npm install csrf-csrf cookie-parser
```

```typescript
// main.ts - TÍCH HỢP BẢO VỆ CSRF TOKEN
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as cookieParser from 'cookie-parser';
import { doubleCsrf } from 'csrf-csrf';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.use(cookieParser('my-cookie-secret-key-32-chars'));

  // 2. CẤU HÌNH DOUBLE CSRF PROTECTION
  const { doubleCsrfProtection, generateToken } = doubleCsrf({
    getSecret: () => 'csrf-secret-key-super-secure',
    cookieName: '__Host-ps.x-csrf-token',
    cookieOptions: {
      httpOnly: true,
      sameSite: 'lax',
      secure: process.env.NODE_ENV === 'production',
      path: '/',
    },
    size: 64,
    ignoredMethods: ['GET', 'HEAD', 'OPTIONS'], // Các phương thức an toàn không cần CSRF Token
    getTokenFromRequest: (req) => req.headers['x-csrf-token'] as string,
  });

  // Áp dụng middleware kiểm tra CSRF token cho toàn bộ các request POST/PUT/DELETE
  app.use(doubleCsrfProtection);

  await app.listen(3000);
}
bootstrap();
```

---

### 3. Controller Cung cấp CSRF Token cho Frontend
Frontend sẽ gọi endpoint này khi khởi động ứng dụng để lấy token và đính kèm vào header `x-csrf-token` cho các request `POST`:

```typescript
// auth/csrf.controller.ts
import { Controller, Get, Req, Res } from '@nestjs/common';
import { Request, Response } from 'express';

@Controller('auth')
export class CsrfController {
  @Get('csrf-token')
  getCsrfToken(@Req() req: Request, @Res() res: Response) {
    // Giả sử hàm generateToken từ doubleCsrf
    // Client nhận token này và đính kèm vào header: "X-CSRF-Token": token
    return res.json({
      csrfToken: (req as any).csrfToken ? (req as any).csrfToken() : 'token-sample',
    });
  }
}
```

---

### 4. TUÂN THỦ NGUYÊN TẮC THIẾT KẾ REST API: Cấm Thay đổi Trạng thái bằng `GET`

#### ❌ Thiết kế Cực kỳ Nguy hiểm (VULNERABLE)
```typescript
// payments.controller.ts - SAI LẦM TAI HẠI: Dùng GET để chuyển tiền
@Get('transfer')
async transferMoneyVulnerable(
  @Query('to') to: string,
  @Query('amount') amount: number,
  @CurrentUser() user: any,
) {
  // BỊ CSRF TẤN CÔNG 100%: Hacker chỉ cần chèn thẻ <img src="https://bank.com/transfer?to=hacker&amount=1000">
  return this.paymentsService.transfer(user.id, to, amount);
}
```

####  Thiết kế Chuẩn mực (SECURE)
```typescript
// payments.controller.ts - CHUẨN MỰC REST: Dùng POST kèm Body
@Post('transfer')
async transferMoneySafe(
  @Body() transferDto: TransferDto,
  @CurrentUser() user: any,
) {
  // Trình duyệt sẽ chặn CSRF vì request là POST, có SameSite=Lax và kiểm tra CSRF Token
  return this.paymentsService.transfer(user.id, transferDto.to, transferDto.amount);
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Phương thức `GET` phải tuyệt đối an toàn và mang tính Idempotent:** Chỉ dùng để đọc dữ liệu, không bao giờ dùng `GET` để xóa, cập nhật hay tạo mới dữ liệu.
- **Nếu dùng Authorization Header (`Bearer Token`):** Ứng dụng SPA của bạn đã tự nhiên có khả năng miễn nhiễm với CSRF thông thường.
- **Nếu dùng Cookie lưu Session/JWT:** Bắt buộc phải cấu hình `sameSite: 'lax'` (hoặc `'strict'`) và tích hợp cơ chế Anti-CSRF Token cho các thao tác thay đổi dữ liệu.

