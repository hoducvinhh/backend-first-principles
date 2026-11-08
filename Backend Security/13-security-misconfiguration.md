# Phần 13: Lỗ hổng Cấu hình Sai (Security Misconfiguration)

> **Tư duy cốt lõi:**  
> *"Lỗ hổng bảo mật nguy hiểm nhất thường không nằm ở thuật toán phức tạp mà bắt nguồn từ những sai sót cấu hình ngớ ngẩn: để lộ secret key trong Git history, bật Debug Mode trên Production hoặc thiếu các Security Headers cơ bản."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### 1. Quản lý Bí mật (Secrets Management)
- **Sai lầm kinh điển:** Hardcode API Key, mật khẩu Database, JWT Secret vào code rồi commit lên GitHub/GitLab.
- **Cạm bẫy Lịch sử Git (Git History):** Dù bạn có tạo commit mới để xóa chuỗi secret đi, **chuỗi đó vẫn tồn tại vĩnh viễn trong commit history**. Hacker chỉ cần clone repo về và dùng lệnh `git log` hoặc tool rà quét tự động là lấy được bí mật ngay.
- **Hành động bắt buộc:** Khi một secret vô tình bị commit, bạn **BẮT BUỘC PHẢI THU HỒI VÀ THAY MỚI (ROTATE)** secret đó ngay lập tức!
- **Giải pháp:** Sử dụng Biến môi trường (`.env` nằm trong `.gitignore`) hoặc các kho quản lý bí mật chuyên nghiệp (AWS Secrets Manager, HashiCorp Vault).

---

### 2. Bật Chế độ Gỡ lỗi (Debug Mode) trên Production
- Ở môi trường Local, việc in ra câu lệnh SQL, cấu hình kết nối, và toàn bộ Stack Trace là hữu ích để debug.
- Nếu để nguyên chế độ này trên Production:
  - Khi ứng dụng gặp lỗi, phản hồi HTTP trả về toàn bộ **Stack Trace**.
  - Stack trace tiết lộ: tên file, đường dẫn thư mục trên máy chủ (`/var/www/app/...`), phiên bản thư viện, tên hàm nội bộ, cấu trúc câu query SQL.
  - Hacker sẽ sử dụng các thông tin rò rỉ này để vẽ nên bức tranh cấu trúc hệ thống và lên kịch bản khai thác chính xác.
- **Quy tắc:**
  - Local/Dev: Log level `DEBUG`, hiển thị lỗi chi tiết.
  - Production: Log level `INFO`, `WARN` hoặc `ERROR`. **Tuyệt đối ẩn 100% Stack Trace khỏi phản hồi gửi về client!**

---

### 3. Cấu hình các Header Bảo mật HTTP (Security Headers)
- **`X-Frame-Options: DENY` (hoặc `SAMEORIGIN`):** Ngăn chặn trang web bị nhúng vào bên trong thẻ `<iframe>` của domain khác. Chống cuộc tấn công **Clickjacking** (kẻ gian nhúng trang web của bạn dưới một nút bấm trong suốt để lừa người dùng click chuyển tiền/xóa tài khoản).
- **`X-Content-Type-Options: nosniff`:** Ngăn trình duyệt tự ý đoán kiểu dữ liệu (MIME-sniffing), ép trình duyệt tuân thủ đúng `Content-Type` do server khai báo.
- **`Strict-Transport-Security (HSTS)`:** Ép buộc trình duyệt chỉ được kết nối qua HTTPS trong tương lai, ngăn chặn các cuộc tấn công hạ cấp giao thức (SSL Stripping).

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong NestJS, việc ngăn chặn Security Misconfiguration được thực hiện qua 3 kỹ thuật:
1. **Module `@nestjs/config` kết hợp với `Joi`:** Validate toàn bộ biến môi trường lúc khởi động ứng dụng (nếu thiếu secret key quan trọng, ứng dụng sẽ từ chối khởi động ngay lập tức).
2. **Global Exception Filter (`AllExceptionsFilter`):** Bắt toàn bộ lỗi chưa được xử lý, ghi log chi tiết phía server nhưng **chỉ trả về mã lỗi chung và `traceId` cho client** khi chạy ở môi trường Production.
3. **Thư viện `helmet`:** Tự động kích hoạt toàn bộ các security headers chỉ bằng một dòng code.

---

## 3. Thực hành Code NestJS: Cấu hình Production Chuẩn mực

### Cài đặt thư viện
```bash
npm install @nestjs/config joi helmet
```

### 1. Validate Biến Môi trường Khắt khe với Joi
Ngăn chặn việc ứng dụng khởi động với secret mặc định hoặc thiếu cấu hình quan trọng:

```typescript
// app.module.ts - VALIDATE BIẾN MÔI TRƯỜNG LÚC KHỞI ĐỘNG
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      // ĐỊNH NGHĨA SCHEMA BẮT BUỘC CHO FILE .ENV
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test')
          .default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required(),
        JWT_ACCESS_SECRET: Joi.string().min(32).required(), // Bắt buộc dài tối thiểu 32 ký tự!
        REDIS_HOST: Joi.string().required(),
      }),
      validationOptions: {
        allowUnknown: true, // Cho phép các biến môi trường khác của hệ thống
        abortEarly: false,  // Báo toàn bộ các biến bị thiếu cùng lúc
      },
    }),
  ],
})
export class AppModule {}
```

---

### 2. Global Exception Filter: Ẩn Hoàn toàn Stack Trace trên Production
```typescript
// common/filters/all-exceptions.filter.ts - CHỐNG RÒ RỈ THÔNG TIN NỘI BỘ
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger('AllExceptionsFilter');

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    // Sinh traceId duy nhất để phục vụ tra cứu log nội bộ
    const traceId = uuidv4();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const message =
      exception instanceof HttpException
        ? exception.getResponse()
        : 'Internal server error';

    // 1. GHI LOG ĐẦY ĐỦ PHÍA SERVER (Dành riêng cho kỹ sư nội bộ xem)
    this.logger.error(
      `[TraceId: ${traceId}] Lỗi tại ${request.method} ${request.url} - Status: ${status}`,
      exception instanceof Error ? exception.stack : exception,
    );

    // 2. PHẢN HỒI CHO CLIENT: TUYỆT ĐỐI ẨN STACK TRACE TRÊN PRODUCTION
    const isProduction = process.env.NODE_ENV === 'production';

    if (isProduction && status === HttpStatus.INTERNAL_SERVER_ERROR) {
      // Trên Production: Không tiết lộ bất kỳ chi tiết kỹ thuật nào!
      return response.status(status).json({
        statusCode: status,
        message: 'Đã xảy ra lỗi nội bộ trong hệ thống. Vui lòng liên hệ hỗ trợ.',
        traceId: traceId, // Cung cấp traceId để người dùng báo cho bộ phận support
        timestamp: new Date().toISOString(),
      });
    }

    // Ở môi trường Development: Có thể hiển thị chi tiết để tiện debug
    return response.status(status).json({
      statusCode: status,
      error: typeof message === 'object' ? message : { message },
      traceId: traceId,
      stack: isProduction ? undefined : (exception instanceof Error ? exception.stack : undefined),
      timestamp: new Date().toISOString(),
    });
  }
}
```

---

### 3. Kích hoạt Security Headers với Helmet trong `main.ts`
```typescript
// main.ts - KÍCH HOẠT HELMET
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import helmet from 'helmet';
import { AllExceptionsFilter } from './common/filters/all-exceptions.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 1. BẬT TOÀN BỘ CÁC HTTP SECURITY HEADERS
  app.use(
    helmet({
      frameguard: { action: 'deny' }, // X-Frame-Options: DENY (Chống Clickjacking)
      noSniff: true,                   // X-Content-Type-Options: nosniff
      hsts: {
        maxAge: 31536000,              // Strict-Transport-Security: 1 năm
        includeSubDomains: true,
        preload: true,
      },
      hidePoweredBy: true,             // Xóa bỏ header "X-Powered-By: Express" (tránh lộ công nghệ)
    }),
  );

  // 2. KÍCH HOẠT EXCEPTION FILTER BẢO VỆ LỖI TOÀN CỤC
  app.useGlobalFilters(new AllExceptionsFilter());

  await app.listen(3000);
}
bootstrap();
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Không bao giờ dùng giá trị fallback cho secret quan trọng:** Tuyệt đối không viết `process.env.JWT_SECRET || 'secret'`. Nếu thiếu secret trong `.env`, ứng dụng phải lập tức crash lúc khởi động (Fail Fast)!
- **Không bao giờ commit file `.env` lên Git:** Luôn đưa `.env` vào `.gitignore` và chỉ commit file mẫu `.env.example` chứa các key rỗng không có giá trị thật.
- **Trace ID là chiếc cầu nối an toàn:** Luôn sinh một `traceId` ngẫu nhiên khi có lỗi 500 xảy ra, in vào file log nội bộ và trả về cho client. Bạn vừa ẩn được chi tiết lỗi nhạy cảm, vừa có thể tra cứu chính xác nguyên nhân lỗi trong hệ thống log.

