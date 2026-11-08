# Phần 1: Giới thiệu về Bảo mật trong Backend (Introduction to Security)

> **Tư duy cốt lõi:**  
> *"Không có hệ thống nào an toàn 100%. Bảo mật là một quá trình liên tục và đòi hỏi tư duy luôn cảnh giác (paranoid mindset) ở mỗi dòng code."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Tầm quan trọng của Bảo mật Backend
- Lỗi logic nghiệp vụ hay hiệu năng có thể làm chậm ứng dụng, nhưng **lỗ hổng bảo mật có thể phá hủy hoàn toàn doanh nghiệp**: rò rỉ dữ liệu người dùng, vi phạm pháp luật (GDPR, nghị định bảo vệ dữ liệu), đền bù tài chính khổng lồ và mất uy tín thương hiệu vĩnh viễn.
- Mọi sự cố bảo mật cuối cùng đều quy đổi thành chi phí tài chính.

### Bốn phân tầng bảo mật
1. **Browser Security:** Cookies, LocalStorage, DOM, Same-Origin Policy.
2. **Network Security:** HTTPS/TLS, mã hóa đường truyền, chứng chỉ SSL, chống nghe lén gói tin (Sniffing/Man-in-the-Middle).
3. **Server/OS Security:** Quản lý quyền người dùng trong Linux, firewall, bảo vệ các tiến trình (processes) đang chạy ứng dụng backend.
4. **Backend Application Security:** Logic xử lý dữ liệu, xác thực, phân quyền, câu lệnh database, tích hợp thư viện bên thứ ba.

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong kiến trúc của **NestJS**, một HTTP request đi qua một chuỗi các tầng (Request Lifecycle) trước khi chạm tới Business Logic trong Service:

```
[HTTP Request] 
      │
      ▼
1. Global Middleware (Helmet, CORS, CookieParser)
      │
      ▼
2. Guards (Xác thực AuthGuard, Phân quyền RolesGuard)
      │
      ▼
3. Interceptors (Pre-controller: Logging, Metrics, Throttling)
      │
      ▼
4. Pipes (ValidationPipe: Kiểm tra & làm sạch dữ liệu đầu vào)
      │
      ▼
5. Controller & Service (Business Logic & Database access)
      │
      ▼
6. Interceptors (Post-controller: Định dạng dữ liệu trả về, ẩn trường nhạy cảm)
      │
      ▼
7. Exception Filters (Bắt lỗi toàn cục, ẩn Stack Trace trên Production)
```

Một hệ thống NestJS an toàn phải được thiết lập phòng thủ **ngay từ cổng vào (`main.ts`)** và xuyên suốt qua các tầng này.

---

## 3. Thực hành Code NestJS: Cấu hình Bảo mật Nền tảng trong `main.ts`

### ❌ Trường hợp Kém an toàn (Vulnerable / Default Setup)
Lập trình viên khởi tạo ứng dụng NestJS mặc định mà không thiết lập các chốt chặn an toàn:

```typescript
// main.ts - NGUY HIỂM: Thiếu hoàn toàn các chốt chặn bảo mật cơ bản
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  // Cho phép mọi origin gọi API mà không có ràng buộc
  app.enableCors(); 
  // Không có ValidationPipe: Dữ liệu bẩn có thể đi thẳng vào Controller
  // Không có Helmet: Thiếu các Security Headers bảo vệ trình duyệt
  await app.listen(3000);
}
bootstrap();
```

---

###  Trường hợp Chuẩn mực An toàn (Secure Production Setup)
Thiết lập đầy đủ các chốt chặn bảo vệ ứng dụng NestJS ở cấp độ toàn cục:

```typescript
// main.ts - CHUẨN BẢO MẬT PRODUCTION
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, VersioningType, Logger } from '@nestjs/common';
import { AppModule } from './app.module';
import helmet from 'helmet';
import * as cookieParser from 'cookie-parser';
import { AllExceptionsFilter } from './common/filters/all-exceptions.filter';

async function bootstrap() {
  const logger = new Logger('Bootstrap');
  const app = await NestFactory.create(AppModule, {
    // Tắt logger chi tiết ở production nếu cần bảo mật thông tin nội bộ
    logger: process.env.NODE_ENV === 'production' 
      ? ['error', 'warn', 'log'] 
      : ['error', 'warn', 'log', 'debug', 'verbose'],
  });

  // 1. HTTP Security Headers với Helmet (Ẩn X-Powered-By, chống Clickjacking...)
  app.use(helmet());

  // 2. Cookie Parser phục vụ việc đọc HttpOnly Cookies an toàn
  app.use(cookieParser(process.env.COOKIE_SECRET));

  // 3. Cấu hình CORS nghiêm ngặt - Chỉ cho phép domain của frontend hợp lệ
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS 
      ? process.env.ALLOWED_ORIGINS.split(',') 
      : ['https://myapp.com'],
    credentials: true, // Cho phép truyền cookie
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
  });

  // 4. API Versioning - Kiểm soát phiên bản API, tránh lộ các endpoint thử nghiệm
  app.enableVersioning({
    type: VersioningType.URI,
    defaultVersion: '1',
  });

  // 5. Global Validation Pipe: Chốt chặn dữ liệu độc hại đầu tiên
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,              // Tự động loại bỏ các field không khai báo trong DTO
      forbidNonWhitelisted: true,   // Ném lỗi 400 nếu client gửi kèm field lạ (chống Mass Assignment)
      transform: true,              // Tự động ép kiểu dữ liệu sang kiểu trong DTO
      disableErrorMessages: process.env.NODE_ENV === 'production', // Ẩn chi tiết lỗi validate ở production
    }),
  );

  // 6. Global Exception Filter: Giấu Stack Trace trên Production
  app.useGlobalFilters(new AllExceptionsFilter());

  const port = process.env.PORT || 3000;
  await app.listen(port);
  logger.log(`Ứng dụng đang chạy an toàn tại cổng: ${port}`);
}
bootstrap();
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Không bao giờ tin tưởng client:** Bất kể request đến từ Web App hay Mobile App của công ty, request đó đều có thể bị kẻ tấn công can thiệp.
- **Khai thác trọn vẹn Request Lifecycle của NestJS:** Sử dụng `Pipes` để làm sạch dữ liệu, `Guards` để chặn cửa truy cập trái phép, và `Filters` để che giấu thông tin lỗi nhạy cảm.

