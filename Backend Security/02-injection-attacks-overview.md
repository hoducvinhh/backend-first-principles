# Phần 2: Tổng quan về Tấn công Tiêm nhiễm (Injection Attacks Overview)

> **Tư duy cốt lõi:**  
> *"Kẻ tấn công không quan tâm bạn dùng framework gì. Họ chỉ tìm câu trả lời cho duy nhất một câu hỏi: Lập trình viên đã đưa ra những giả định ngây thơ nào ở đây?"*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Bản chất của Tấn công Tiêm nhiễm (Injection)
- Ứng dụng Backend là một phiên dịch viên đa ngôn ngữ: giao tiếp với Database qua **SQL**, giao tiếp với Trình duyệt qua **HTML/JS/CSS**, giao tiếp với Hệ điều hành qua **Shell commands**.
- Mỗi ngôn ngữ có hệ thống cú pháp, ký tự điều khiển (`'`, `;`, `--`, `|`, `&&`, `<script>`) để phân biệt giữa:
  - **Lệnh điều khiển (CODE / COMMAND)**
  - **Dữ liệu thuần túy (DATA)**
- **Nguyên nhân gốc rễ:** Lỗ hổng Injection phát sinh khi dữ liệu người dùng gửi lên vượt qua ranh giới giữa hai ngôn ngữ mà không được cô lập, khiến hệ thống **nhầm lẫn giữa Code và Data** — thực thi dữ liệu của người dùng như một phần của câu lệnh hệ thống.

### Cạm bẫy "Con đường màu hồng" (Happy Path Trap)
- Dev thường giả định người dùng chỉ gửi các trường hợp lệ, đúng kiểu dữ liệu, không ai mở F12 để gửi thêm trường `isAdmin: true` hay các ký tự đặc biệt.
- Attacker thì ngược lại: thử mọi ký tự đặc biệt để phá vỡ cấu trúc cú pháp của hệ thống.

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong NestJS, **Tuyến phòng thủ đầu tiên** chống lại Injection và dữ liệu bẩn là:
1. **DTO (Data Transfer Object):** Định nghĩa hợp đồng dữ liệu chuẩn xác.
2. **`class-validator` & `class-transformer`:** Kiểm tra ràng buộc kiểu, độ dài, regex của từng trường.
3. **`ValidationPipe`:** Tự động loại bỏ các trường không mong muốn và ném lỗi 400 nếu dữ liệu vi phạm.

Ngoài ra, cạm bẫy lớn nhất trong NestJS liên quan đến giả định đầu vào là **Mass Assignment (Gán dữ liệu hàng loạt)**: Dev lấy trực tiếp `req.body` đưa vào Database update, vô tình cho phép kẻ tấn công sửa quyền hạn của chính họ!

---

## 3. Thực hành Code NestJS: Phòng thủ Dữ liệu Đầu vào

### ❌ Trường hợp Kém an toàn (Vulnerable: Nhận dữ liệu không kiểm soát)
Lập trình viên tin tưởng dữ liệu từ client và không sử dụng DTO hoặc không bật kiểm tra whitelist:

```typescript
// users.controller.ts - NGUY HIỂM: Nhận trực tiếp object bất kỳ từ body
import { Controller, Post, Body } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post('register')
  async register(@Body() rawBody: any) {
    // Kẻ tấn công gửi: { "email": "attacker@evil.com", "password": "123", "role": "admin" }
    // Giả định sai lầm: Nghĩ rằng form chỉ có email và password
    return this.usersService.create(rawBody); // role 'admin' bị lưu thẳng vào DB!
  }
}
```

---

###  Trường hợp Chuẩn mực An toàn (Secure: Ràng buộc DTO nghiêm ngặt)
Định nghĩa DTO rõ ràng, kiểm tra từng kiểu dữ liệu và cấu hình `ValidationPipe`:

#### 1. Định nghĩa DTO với `class-validator`
```typescript
// dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, MaxLength, Matches } from 'class-validator';

export class CreateUserDto {
  @IsEmail({}, { message: 'Email không đúng định dạng chuẩn' })
  @MaxLength(100, { message: 'Email không được vượt quá 100 ký tự' })
  email: string;

  @IsString({ message: 'Mật khẩu phải là chuỗi ký tự' })
  @MinLength(8, { message: 'Mật khẩu tối thiểu phải có 8 ký tự' })
  @MaxLength(64, { message: 'Mật khẩu không được dài quá 64 ký tự' })
  @Matches(/((?=.*\d)|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/, {
    message: 'Mật khẩu quá yếu: cần chứa chữ hoa, chữ thường và chữ số/ký tự đặc biệt',
  })
  password: string;

  // TUYỆT ĐỐI KHÔNG KHAI BÁO CÁC TRƯỜNG NHẠY CẢM NHƯ role, balance, isVerified Ở ĐÂY!
}
```

#### 2. Áp dụng DTO trong Controller
```typescript
// users.controller.ts - AN TOÀN
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post('register')
  @HttpCode(HttpStatus.CREATED)
  async register(@Body() createUserDto: CreateUserDto) {
    // Nhờ ValidationPipe với { whitelist: true, forbidNonWhitelisted: true },
    // nếu client cố tình gửi thêm { role: 'admin' }, request sẽ bị chặn đứng với mã lỗi 400 Bad Request!
    return this.usersService.create(createUserDto);
  }
}
```

#### 3. Xử lý an toàn trong Service (Chống Mass Assignment)
```typescript
// users.service.ts
import { Injectable } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  async create(dto: CreateUserDto) {
    // Chỉ trích xuất tường minh các trường được phép, gán role mặc định phía backend
    const newUser = {
      email: dto.email,
      password: dto.password, // Sau này sẽ được băm bằng Argon2/Bcrypt
      role: 'member',         // Backend chủ động kiểm soát giá trị này
      createdAt: new Date(),
    };

    // Lưu vào database
    return newUser;
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Mọi dữ liệu đầu vào đều là độc hại (Untrusted Input):** Không bao giờ dùng kiểu `any` cho `@Body()`, `@Query()` hay `@Param()`.
- **Luôn kích hoạt bộ 3 của ValidationPipe:**
  - `whitelist: true`: Tự động gạt bỏ các thuộc tính rác.
  - `forbidNonWhitelisted: true`: Báo động ngay khi client gửi trường không cho phép.
  - `transform: true`: Ép kiểu dữ liệu an toàn theo DTO.
- **Tách biệt ranh giới:** Dữ liệu từ Request (HTTP) $\rightarrow$ DTO (Validation) $\rightarrow$ Domain Entity (Business Logic) $\rightarrow$ Database (Persistence). Tuyệt đối không để một luồng dữ liệu thô đi thẳng từ HTTP vào DB!

