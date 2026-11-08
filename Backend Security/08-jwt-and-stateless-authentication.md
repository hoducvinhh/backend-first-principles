# Phần 8: JWT & Xác thực Phi trạng thái (JWT & Stateless Authentication)

> **Tư duy cốt lõi:**  
> *"JWT không phải là phép màu. Payload của JWT chỉ là Base64Url, bất kỳ ai cũng có thể đọc được. Nếu dùng JWT, hãy dùng Access Token ngắn hạn kết hợp Refresh Token và lưu trong HttpOnly Cookie để giảm thiểu rủi ro."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### So sánh: Stateful Session vs Stateless JWT
- **Stateful Session:** Server lưu dữ liệu trong Redis/DB, giao cho client chuỗi Session ID ngẫu nhiên. Dễ dàng thu hồi tức thì, nhưng tốn chi phí RAM và tra cứu DB.
- **Stateless JWT:** Server đóng gói dữ liệu vào Token, ký số rồi giao cho client tự giữ. Mỗi request client gửi toàn bộ token lên, server chỉ cần giải mã chữ ký mà không cần chạm vào DB $\rightarrow$ Dễ dàng scale ngang (horizontal scaling).

---

### Giải phẫu cấu trúc của JSON Web Token (JWT)

$$\text{JWT} = \underbrace{\text{Header}}_{\text{Base64Url}} \boldsymbol{.} \underbrace{\text{Payload}}_{\text{Base64Url}} \boldsymbol{.} \underbrace{\text{Signature}}_{\text{Chữ ký số}}$$

1. **Header:** Thuật toán (`alg`: HS256, RS256) và kiểu token (`typ: "JWT"`).
2. **Payload (Claims):** Chứa các thông tin (`sub`: userId, `role`: 'admin', `exp`: hạn dùng).
   > **CẢNH BÁO CHÍ MẠNG:** Payload **CHỈ ĐƯỢC MÃ HÓA BASE64, HOÀN TOÀN KHÔNG BẢO MẬT**. Bất kỳ ai vào trang `jwt.io` đều decode và đọc được 100% nội dung. **TUYỆT ĐỐI KHÔNG LƯU MẬT KHẨU, SỐ THẺ, HAY BÍ MẬT VÀO ĐÂY!**
3. **Signature (Chữ ký):** Đảm bảo tính toàn vẹn (Integrity). Nếu ai sửa `role: "member"` thành `"admin"`, chữ ký sẽ bị lệch và server từ chối ngay.

---

### Bất cập Lớn nhất của JWT: Thu hồi phiên rất khó (Revocation is Hard)
- Khi một JWT được phát hành, nó có hiệu lực cho đến khi chạm mốc `exp`. Nếu tài khoản bị hack, bạn không thể ép buộc client xóa token từ xa.
- **Giải pháp chuẩn công nghiệp:**
  - **Access Token:** Vòng đời cực ngắn (**5 đến 15 phút**).
  - **Refresh Token:** Vòng đời dài hơn (**7 ngày**), được băm và lưu trong Database phía server.
  - Khi Access Token hết hạn, client dùng Refresh Token để xin cấp cặp token mới. Nếu tài khoản bị lộ, chỉ cần xóa Refresh Token trong DB là hacker sẽ bị chặn sau tối đa 15 phút.

---

## 2. Ánh xạ vào Hệ sinh thái NestJS (`@nestjs/jwt` & `passport-jwt`)

Trong NestJS, triển khai JWT an toàn bao gồm:
1. Module `@nestjs/jwt` để ký và giải mã token.
2. Thư viện `passport-jwt` để xây dựng `JwtStrategy`.
3. Guard chuyên dụng: `JwtAuthGuard`.

---

## 3. Thực hành Code NestJS: Triển khai Cặp Access / Refresh Token Chuẩn mực

### Cài đặt thư viện
```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install -D @types/passport-jwt
```

### 1. Cấu hình `JwtStrategy` trong NestJS
Trích xuất token an toàn từ `HttpOnly` Cookie hoặc Authorization Header:

```typescript
// auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { Request } from 'express';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor() {
    super({
      // Ưu tiên đọc Token từ Cookie an toàn, nếu không có mới tìm trong Bearer Header
      jwtFromRequest: ExtractJwt.fromExtractors([
        (request: Request) => {
          return request?.cookies?.access_token || null;
        },
        ExtractJwt.fromAuthHeaderAsBearerToken(),
      ]),
      ignoreExpiration: false, // Bắt buộc kiểm tra hạn dùng (exp)
      secretOrKey: process.env.JWT_ACCESS_SECRET || 'access-secret-32-chars-minimum',
    });
  }

  async validate(payload: any) {
    // Payload sau khi đã được verify chữ ký thành công
    if (!payload.sub) {
      throw new UnauthorizedException('Token không hợp lệ');
    }
    // Gán dữ liệu này vào req.user
    return { userId: payload.sub, email: payload.email, role: payload.role };
  }
}
```

---

### 2. Guard Bảo vệ Route: `JwtAuthGuard`
```typescript
// common/guards/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

---

### 3. Service Cấp phát & Thu hồi Token An toàn
```typescript
// auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as argon2 from 'argon2';
import { User } from '../users/entities/user.entity';

@Injectable()
export class AuthService {
  constructor(
    private readonly jwtService: JwtService,
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  /**
   * Sinh cặp Access Token (15 phút) và Refresh Token (7 ngày)
   */
  async generateTokens(user: User) {
    const payload = { sub: user.id, email: user.email, role: user.role };

    const [accessToken, refreshToken] = await Promise.all([
      // Access Token: Hết hạn sau 15 phút
      this.jwtService.signAsync(payload, {
        secret: process.env.JWT_ACCESS_SECRET,
        expiresIn: '15m',
      }),
      // Refresh Token: Hết hạn sau 7 ngày
      this.jwtService.signAsync(payload, {
        secret: process.env.JWT_REFRESH_SECRET,
        expiresIn: '7d',
      }),
    ]);

    // Băm Refresh Token trước khi lưu vào DB (Nếu DB bị lộ, hacker không dùng được Refresh Token)
    const hashedRefreshToken = await argon2.hash(refreshToken);
    await this.userRepository.update(user.id, { hashedRefreshToken });

    return { accessToken, refreshToken };
  }

  /**
   * Cơ chế Refresh Token: Cấp Access Token mới và xoay vòng Refresh Token (Token Rotation)
   */
  async refreshTokens(userId: string, incomingRefreshToken: string) {
    const user = await this.userRepository.findOne({ where: { id: userId } });
    if (!user || !user.hashedRefreshToken) {
      throw new UnauthorizedException('Truy cập bị từ chối');
    }

    // So sánh token gửi lên với hash trong DB
    const isMatched = await argon2.verify(user.hashedRefreshToken, incomingRefreshToken);
    if (!isMatched) {
      // PHÁT HIỆN GIAN LẬN: Xóa luôn hash trong DB để ép login lại
      await this.userRepository.update(userId, { hashedRefreshToken: null });
      throw new UnauthorizedException('Token gian lận! Vui lòng đăng nhập lại');
    }

    // Cấp cặp token mới và cập nhật hash mới (Refresh Token Rotation)
    return this.generateTokens(user);
  }

  /**
   * Thu hồi phiên (Logout): Xóa Refresh Token trong Database
   */
  async logout(userId: string) {
    await this.userRepository.update(userId, { hashedRefreshToken: null });
    return { success: true, message: 'Đã thu hồi token thành công' };
  }
}
```

---

### 4. Đính kèm Token vào `HttpOnly` Cookie trong Controller
```typescript
// auth/auth.controller.ts
import { Controller, Post, Body, Res, HttpCode, HttpStatus } from '@nestjs/common';
import { Response } from 'express';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('login')
  @HttpCode(HttpStatus.OK)
  async login(@Body() dto: LoginDto, @Res({ passthrough: true }) res: Response) {
    const user = await this.authService.validateUser(dto);
    const { accessToken, refreshToken } = await this.authService.generateTokens(user);

    // Gửi Access Token qua Cookie HttpOnly an toàn
    res.cookie('access_token', accessToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 15 * 60 * 1000, // 15 phút
    });

    // Gửi Refresh Token qua Cookie riêng biệt với path chỉ định
    res.cookie('refresh_token', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      path: '/auth/refresh', // Chỉ gửi cookie này khi gọi endpoint refresh
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 ngày
    });

    return { message: 'Đăng nhập thành công' };
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Tuyệt đối không lưu Access Token trong `LocalStorage` của trình duyệt:** XSS có thể lấy cắp token ngay tức khắc. Luôn dùng `HttpOnly Cookie`.
- **Áp dụng Refresh Token Rotation:** Mỗi lần cấp Access Token mới, hãy cấp luôn Refresh Token mới và hủy token cũ.
- **Nếu hệ thống chỉ là monolithic hoặc ít microservices:** Hãy cân nhắc dùng **Stateful Session với Redis** (Phần 7) vì nó đơn giản và an toàn hơn rất nhiều so với việc quản lý vòng đời phức tạp của JWT.

