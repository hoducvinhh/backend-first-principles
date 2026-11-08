# Phần 7: Quản lý Phiên & Bảo mật Cookie (Session Management & Cookie Security)

> **Tư duy cốt lõi:**  
> *"Session ID phải là một chuỗi ngẫu nhiên khổng lồ được tạo bởi CSPRNG. Cookie chứa Session ID bắt buộc phải có đủ 3 cờ: HttpOnly, Secure và SameSite để chống lại cả XSS lẫn CSRF."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Tại sao cần Quản lý Phiên (Session Management)?
- Giao thức HTTP mang tính chất phi trạng thái (Stateless).
- Sau khi người dùng đăng nhập thành công, hệ thống không thể bắt họ gửi lại mật khẩu ở mỗi request tiếp theo. Hệ thống cần ghi nhớ trạng thái này qua **Phiên làm việc (Session)**.

### Cơ chế Stateful Session chuẩn mực
1. **Sinh Session ID:** Sử dụng CSPRNG để sinh chuỗi ngẫu nhiên dài từ **128 đến 256 bits** (không gian mẫu $2^{128}$ lớn hơn số nguyên tử trong vũ trụ $\rightarrow$ bất khả thi để đoán mò).
2. **Lưu trữ Metadata trong Redis:** Server lưu Session ID làm khóa tra cứu trong Redis, kèm thông tin: `userId`, `createdAt`, `expiresAt`, `ipAddress`, `userAgent`.
3. **Phát hành Cookie an toàn:** Trình duyệt nhận cookie qua header `Set-Cookie` và tự động gửi lại ở các request sau.
4. **Xác thực:** Server lấy Session ID từ Cookie, tra cứu Redis để xác định người dùng.

---

### Bộ 3 Cờ Bảo mật Cookie Bắt buộc

```
Set-Cookie: session_id=abc123xyz...; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=604800
```

1. **`HttpOnly` (`true`):** Cấm hoàn toàn JavaScript phía client (`document.cookie`) đọc cookie. Nếu website vô tình dính XSS, kẻ tấn công **KHÔNG THỂ** đánh cắp được Session ID. (Tuyệt đối không lưu Session ID hay Token trong `LocalStorage`!).
2. **`Secure` (`true`):** Chỉ cho phép trình duyệt gửi cookie qua kết nối mã hóa **HTTPS/TLS**. Chống việc nghe lén gói tin trên Wi-Fi công cộng.
3. **`SameSite` (`Lax` hoặc `Strict`):**
   - `Strict`: Chỉ gửi cookie khi request bắt nguồn từ chính domain của bạn. Chống CSRF tuyệt đối.
   - `Lax` (Khuyến nghị chuẩn): Cho phép gửi cookie khi click link điều hướng cấp cao nhất, nhưng chặn gửi trong các request ngầm chạy qua `<img>`, `<iframe>` hay `POST` từ domain khác.
   - `None`: Gửi trong mọi trường hợp (cực kỳ rủi ro, bắt buộc kèm `Secure: true`).

---

## 2. Ánh xạ vào Hệ sinh thái NestJS (Express-Session & Redis)

Trong NestJS, cách chuẩn nhất để triển khai Stateful Session là tích hợp `express-session` cùng với `connect-redis` và `ioredis` tại tầng khởi tạo ứng dụng (`main.ts`).

---

## 3. Thực hành Code NestJS: Cấu hình Stateful Session & Guard

### Cài đặt thư viện cần thiết
```bash
npm install express-session connect-redis ioredis
npm install -D @types/express-session
```

### 1. Cấu hình Session với Redis trong `main.ts`
```typescript
// main.ts - CẤU HÌNH REDIS SESSION AN TOÀN TRONG NESTJS
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as session from 'express-session';
import RedisStore from 'connect-redis';
import Redis from 'ioredis';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 1. Khởi tạo Redis Client
  const redisClient = new Redis({
    host: process.env.REDIS_HOST || 'localhost',
    port: Number(process.env.REDIS_PORT) || 6379,
    password: process.env.REDIS_PASSWORD || undefined,
  });

  // 2. Cấu hình Express Session với Redis Store
  app.use(
    session({
      store: new RedisStore({ 
        client: redisClient,
        prefix: 'sess:', // Tiền tố lưu trong Redis
        ttl: 86400 * 7,  // Thời gian sống: 7 ngày
      }),
      name: '__Host-sid', // Tiền tố __Host- tăng cường bảo mật cookie trên HTTPS
      secret: process.env.SESSION_SECRET || 'super-secret-key-at-least-32-chars',
      resave: false,
      saveUninitialized: false, // Không lưu session rác nếu chưa đăng nhập
      cookie: {
        httpOnly: true, // CHỐNG XSS: JavaScript không thể đọc cookie này
        secure: process.env.NODE_ENV === 'production', // Chỉ gửi qua HTTPS trên production
        sameSite: 'lax', // CHỐNG CSRF
        maxAge: 1000 * 60 * 60 * 24 * 7, // Hạn sử dụng: 7 ngày
        path: '/',
      },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

---

### 2. Guard Kiểm tra Session: `SessionAuthGuard`
Tạo một Guard để bảo vệ các endpoint yêu cầu đăng nhập:

```typescript
// common/guards/session-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Request } from 'express';

@Injectable()
export class SessionAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>();

    // Kiểm tra xem session có chứa thông tin user hay không
    if (!request.session || !(request.session as any).userId) {
      throw new UnauthorizedException('Phiên làm việc đã hết hạn hoặc không hợp lệ');
    }

    return true;
  }
}
```

---

### 3. Controller Xử lý Login, Logout và Quản lý Thiết bị
```typescript
// auth.controller.ts
import { Controller, Post, Body, Req, UseGuards, HttpCode, HttpStatus } from '@nestjs/common';
import { Request } from 'express';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';
import { SessionAuthGuard } from '../common/guards/session-auth.guard';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('login')
  @HttpCode(HttpStatus.OK)
  async login(@Body() loginDto: LoginDto, @Req() req: Request) {
    const user = await this.authService.validateUser(loginDto);

    // 1. Lưu metadata vào session trong Redis
    const sessionData = req.session as any;
    sessionData.userId = user.id;
    sessionData.role = user.role;
    sessionData.ipAddress = req.ip;
    sessionData.userAgent = req.headers['user-agent'];
    sessionData.loginAt = new Date().toISOString();

    return { message: 'Đăng nhập thành công', user: { id: user.id, email: user.email } };
  }

  @Post('logout')
  @UseGuards(SessionAuthGuard)
  @HttpCode(HttpStatus.OK)
  async logout(@Req() req: Request) {
    return new Promise((resolve, reject) => {
      // 2. Thu hồi phiên (Xóa ngay lập tức khỏi Redis)
      req.session.destroy((err) => {
        if (err) {
          reject(err);
        }
        // Xóa cookie khỏi trình duyệt
        req.res.clearCookie('__Host-sid');
        resolve({ message: 'Đăng xuất thành công, phiên đã được thu hồi' });
      });
    });
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Ưu điểm vượt trội của Stateful Session:** Khả năng **thu hồi phiên (Revoke) ngay lập tức**. Khi người dùng đổi mật khẩu hoặc phát hiện tài khoản bị hack, chỉ cần gọi `redisClient.del('sess:' + sessionId)` hoặc xóa toàn bộ key của `userId` đó là họ bị đăng xuất trên mọi thiết bị.
- **Không bao giờ lưu dữ liệu nhạy cảm trong Cookie:** Cookie chỉ chứa duy nhất `Session ID` được mã hóa. Toàn bộ thông tin quyền hạn (`role`), dữ liệu cá nhân phải nằm an toàn trong Redis phía Server.
- **Luôn kích hoạt cờ `httpOnly: true` và `secure: true`:** Đây là chốt chặn phòng thủ bắt buộc cho mọi cookie xác thực.

