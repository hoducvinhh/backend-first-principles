# Phần 9: Chiến lược Giới hạn Tần suất (Rate Limiting Strategies)

> **Tư duy cốt lõi:**  
> *"Không có giới hạn tần suất (Rate Limiting), máy chủ của bạn chỉ cách sự sụp đổ đúng một script chạy vòng lặp vô tận. Hãy xây dựng hệ thống phòng thủ đa tầng: theo IP, theo Tài khoản và trên Toàn hệ thống."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Tại sao Rate Limiting là bắt buộc?
Thiếu cơ chế giới hạn tần suất sẽ dẫn tới 2 thảm họa:
1. **Chiếm đoạt tài khoản:** Kẻ tấn công gửi hàng triệu request dò mật khẩu (Brute-force / Credential Stuffing).
2. **Sập hệ thống (DoS):** Máy chủ cạn kiệt tài nguyên CPU, RAM và kết nối Database.

### Mô hình Phòng thủ Đa tầng (Layered Rate Limiting)

```
[Request Đăng Nhập Gửi Đến]
             │
             ▼
┌──────────────────────────────────────────────────────────┐
│ TẦNG 1: PER-IP RATE LIMITING (THEO ĐỊA CHỈ IP)           │
│ - Cho phép: Tối đa 5 lần thử / phút trên mỗi IP          │
│ - Chặn: Các tool tự động hóa cơ bản.                     │
│ - Điểm yếu: Bị vượt qua bởi Botnet xoay IP / Proxy.      │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│ TẦNG 2: PER-ACCOUNT RATE LIMITING (KHÓA THEO TÀI KHOẢN)  │
│ - Cho phép: Tối đa 5 lần sai / 15 phút trên 1 email      │
│ - Chặn: Tấn công xoay IP nhắm vào 1 tài khoản cụ thể.    │
│ - Điểm yếu: Không chặn được đòn Password Spraying.        │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│ TẦNG 3: GLOBAL RATE LIMITING (TOÀN HỆ THỐNG)             │
│ - Cho phép: Tối đa 100 lần đăng nhập thất bại / phút     │
│ - Hành động: Kích hoạt báo động đỏ, bật CAPTCHA diện rộng│
│   để bảo vệ máy chủ không bị quá tải.                    │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Ánh xạ vào Hệ sinh thái NestJS (`@nestjs/throttler`)

NestJS cung cấp module chính thức cực kỳ mạnh mẽ là **`@nestjs/throttler`**.  
Để hoạt động chính xác trong môi trường Production có nhiều server (Horizontal Scaling), Rate Limiting bắt buộc phải sử dụng **bộ nhớ phân tán Redis** để chia sẻ bộ đếm giữa các instance máy chủ.

---

## 3. Thực hành Code NestJS: Xây dựng Hệ thống Rate Limiting Đa tầng

### Cài đặt thư viện
```bash
npm install @nestjs/throttler ioredis
```

### 1. Cấu hình ThrottlerModule toàn cục với Redis trong `app.module.ts`
```typescript
// app.module.ts - CẤU HÌNH THROTTLER VỚI REDIS
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import Redis from 'ioredis';

const redisClient = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: Number(process.env.REDIS_PORT) || 6379,
});

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000, // 1 giây
        limit: 3,  // Tối đa 3 requests/giây (chống spam click)
      },
      {
        name: 'medium',
        ttl: 10000, // 10 giây
        limit: 20,  // Tối đa 20 requests/10 giây
      },
      {
        name: 'auth_strict',
        ttl: 60000, // 1 phút
        limit: 5,   // Nghiêm ngặt: Tối đa 5 lần gọi/phút cho các route đăng nhập
      },
    ]),
  ],
  providers: [
    // Kích hoạt Guard toàn cục
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}
```

---

### 2. Áp dụng Rate Limiting Nghiêm ngặt cho Auth Controller
Sử dụng decorator `@Throttle()` để ghi đè cấu hình cho các route nhạy cảm:

```typescript
// auth/auth.controller.ts
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { Throttle, SkipThrottle } from '@nestjs/throttler';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  /**
   * TẦNG 1: Áp dụng giới hạn nghiêm ngặt theo IP (5 lần/phút)
   */
  @Post('login')
  @HttpCode(HttpStatus.OK)
  @Throttle({ auth_strict: { limit: 5, ttl: 60000 } })
  async login(@Body() loginDto: LoginDto) {
    return this.authService.loginWithAccountLockout(loginDto);
  }

  /**
   * Bỏ qua rate limit cho các route static/public không tải nặng nếu cần
   */
  @SkipThrottle()
  @Post('health-check')
  healthCheck() {
    return { status: 'ok' };
  }
}
```

---

### 3. TẦNG 2: Khóa Tài khoản (Account Lockout) bằng Redis trong `AuthService`
Nếu kẻ tấn công dùng Proxy xoay IP liên tục để tấn công vào tài khoản `victim@gmail.com`, Tầng 1 theo IP sẽ bị bypass. Chúng ta cần Tầng 2 để khóa chính email đó:

```typescript
// auth/auth.service.ts - TẦNG 2: ACCOUNT LOCKOUT
import { Injectable, UnauthorizedException, HttpException, HttpStatus } from '@nestjs/common';
import Redis from 'ioredis';
import * as argon2 from 'argon2';
import { LoginDto } from './dto/login.dto';

@Injectable()
export class AuthService {
  private redis: Redis;

  constructor() {
    this.redis = new Redis(); // Kết nối Redis
  }

  async loginWithAccountLockout(loginDto: LoginDto) {
    const { email, password } = loginDto;
    const lockKey = `lock:account:${email}`;
    const failCountKey = `failed:count:${email}`;

    // 1. Kiểm tra tài khoản có đang bị khóa hay không
    const isLocked = await this.redis.get(lockKey);
    if (isLocked) {
      const ttl = await this.redis.ttl(lockKey);
      throw new HttpException(
        `Tài khoản này tạm thời bị khóa do đăng nhập sai quá nhiều lần. Vui lòng thử lại sau ${ttl} giây.`,
        HttpStatus.TOO_MANY_REQUESTS, // 429 Too Many Requests
      );
    }

    // 2. Tìm user trong cơ sở dữ liệu
    const user = await this.findUserByEmail(email);

    // 3. Kiểm tra mật khẩu
    const isPasswordValid = user && (await argon2.verify(user.password, password));

    if (!isPasswordValid) {
      // TĂNG BỘ ĐẾM SỐ LẦN ĐĂNG NHẬP THẤT BẠI TRONG REDIS
      const attempts = await this.redis.incr(failCountKey);
      
      // Đặt thời gian sống cho bộ đếm: 15 phút (900 giây)
      if (attempts === 1) {
        await this.redis.expire(failCountKey, 900);
      }

      // NẾU SAI TỪ 5 LẦN TRỞ LÊN: KHÓA TÀI KHOẢN TRONG 30 PHÚT
      if (attempts >= 5) {
        await this.redis.set(lockKey, 'locked', 'EX', 1800); // Khóa 30 phút (1800s)
        await this.redis.del(failCountKey); // Reset bộ đếm
        throw new HttpException(
          'Bạn đã nhập sai mật khẩu 5 lần. Tài khoản bị tạm khóa 30 phút để bảo vệ an toàn!',
          HttpStatus.TOO_MANY_REQUESTS,
        );
      }

      const remainingAttempts = 5 - attempts;
      throw new UnauthorizedException(
        `Mật khẩu không chính xác. Bạn còn ${remainingAttempts} lần thử trước khi tài khoản bị khóa!`,
      );
    }

    // 4. Đăng nhập thành công -> Xóa lịch sử đăng nhập sai
    await this.redis.del(failCountKey);
    return { message: 'Đăng nhập thành công', token: 'jwt_token_here' };
  }

  private async findUserByEmail(email: string) {
    // Giả lập query DB
    return { id: '1', email, password: '$argon2id$v=19$m=65536,t=3,p=1$...' };
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Phân loại giới hạn:**
  - Route đọc thông thường (`GET /products`): Thoáng (ví dụ 100 requests/phút).
  - Route nhạy cảm (`POST /auth/login`, `POST /auth/forgot-password`): Khắt khe (tối đa 5 lần/phút).
  - Route thanh toán (`POST /payments/checkout`): Rất khắt khe (1 request/3 giây) để chống double-click trừ tiền 2 lần.
- **Xử lý IP đằng sau Reverse Proxy (Nginx / Cloudflare):** Nếu NestJS chạy sau Nginx, `req.ip` có thể bị trùng thành `127.0.0.1`. Hãy bật `app.set('trust proxy', true)` trong Express/NestJS để đọc đúng IP thật từ header `X-Forwarded-For`.
- **Phản hồi chuẩn mã lỗi HTTP:** Khi bị chặn tần suất, luôn trả về mã **`429 Too Many Requests`** kèm header `Retry-After`.

