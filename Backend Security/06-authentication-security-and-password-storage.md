# Phần 6: Bảo mật Xác thực & Lưu trữ Mật khẩu (Authentication & Password Storage)

> **Tư duy cốt lõi:**  
> *"Đừng bao giờ phát minh lại mật mã học (Don't roll your own crypto). Hãy sử dụng các thuật toán băm chậm có tham số độ khó bộ nhớ (Memory-hard slow hashing) như Argon2id để biến đòn đánh brute-force từ vài ngày thành hàng thế kỷ."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Xác thực (Authentication) là gì?
- Là quá trình xác minh danh tính: *"Người này thực sự là ai trong bảng `users`?"*.
- Thất bại trong xác thực dẫn đến **Chiếm đoạt tài khoản (Account Takeover)**, rò rỉ dữ liệu cá nhân và thiệt hại tài chính.

### Lời khuyên cho Production: Sử dụng Managed Auth Providers
- Tự viết hệ thống Auth không đơn giản là kiểm tra mật khẩu rồi tạo token. Nó bao gồm:
  - Quản lý phiên đa thiết bị, thu hồi từ xa.
  - Tích hợp OAuth 2.0 / OIDC (Google, GitHub, Apple) và giải quyết bài toán **Account Linking** (khi cùng 1 email đăng nhập bằng cả password và Google).
  - Multi-factor Authentication (MFA), chống Brute-force.
- **Khuyến nghị:** Nếu có ngân sách, hãy dùng Clerk, Auth0, Supabase Auth. Khi quy mô chạm mốc hàng triệu users và chi phí auth quá lớn, doanh nghiệp mới nên tự build hệ thống riêng.

---

### Tiến hóa của các Phương pháp Lưu trữ Mật khẩu

```
1. PLAINTEXT (VĂN BẢN THÔ)
   -> Lộ DB là lộ toàn bộ tài khoản internet của user (do 70% user dùng chung pass).

2. FAST HASHING (MD5, SHA-256)
   -> Bị đánh sập bởi RAINBOW TABLES (bảng tính toán sẵn hàng tỷ mật khẩu phổ biến).

3. HASHING + RANDOM SALT
   -> Thêm chuỗi ngẫu nhiên (CSPRNG) cho TỪNG user.
   -> Triệt tiêu tính giống nhau (sameness), vô hiệu hóa 100% Rainbow Tables.
   -> Nhưng vẫn dễ bị bẻ khóa bởi OFFLINE GPU BRUTE-FORCE (hàng tỷ hash SHA/giây)!

4. SLOW HASHING (ARGON2ID, BCRYPT)
   -> Có tham số Cost Factor (Work Factor) và tốn bộ nhớ RAM (Memory-hard).
   -> Ép thời gian băm mất khoảng 300ms - 500ms.
   -> Bóp nghẹt tốc độ brute-force của hacker từ 1 tỷ lần/giây xuống còn 2-3 lần/giây.
```

- **Tiêu chuẩn công nghiệp số 1 hiện nay:** **Argon2id** (chiến thắng cuộc thi Password Hashing Competition, chống tấn công bằng phần cứng chuyên dụng ASIC/GPU).

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong NestJS, việc xây dựng module xác thực (`AuthModule`) thường tích hợp thư viện:
- Thư viện khuyến nghị hàng đầu: `argon2` (hỗ trợ Argon2id nguyên bản, cực kỳ bảo mật).
- Thư viện thay thế phổ biến: `bcrypt` (chuẩn công nghiệp lâu đời).

Dưới đây là kiến trúc chuẩn mực để lưu trữ và xác thực mật khẩu trong NestJS:

---

## 3. Thực hành Code NestJS: Xây dựng `AuthService` với Argon2id

### Cài đặt thư viện
```bash
npm install argon2
```

### 1. DTO Đăng ký Tài khoản (Ràng buộc độ phức tạp của mật khẩu)
```typescript
// dto/register.dto.ts
import { IsEmail, IsString, MinLength, MaxLength, Matches } from 'class-validator';

export class RegisterDto {
  @IsEmail({}, { message: 'Email không hợp lệ' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Mật khẩu phải từ 8 ký tự trở lên' })
  @MaxLength(64, { message: 'Mật khẩu không được dài quá 64 ký tự' })
  @Matches(/((?=.*\d)|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/, {
    message: 'Mật khẩu phải chứa ít nhất 1 chữ hoa, 1 chữ thường và 1 số/ký tự đặc biệt',
  })
  password: string;
}
```

---

### 2. Service Xử lý Băm Mật khẩu & Xác thực với Argon2id
```typescript
// auth.service.ts - CHUẨN BẢO MẬT MẬT KHẨU VỚI ARGON2ID
import { 
  Injectable, 
  ConflictException, 
  UnauthorizedException, 
  InternalServerErrorException 
} from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as argon2 from 'argon2';
import { User } from './entities/user.entity';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  /**
   * ĐĂNG KÝ: Băm mật khẩu bằng Argon2id kèm tham số Cost Factor chuẩn
   */
  async register(dto: RegisterDto): Promise<{ message: string; userId: string }> {
    // 1. Kiểm tra email đã tồn tại hay chưa
    const existingUser = await this.userRepository.findOne({ where: { email: dto.email } });
    if (existingUser) {
      throw new ConflictException('Email này đã được đăng ký');
    }

    try {
      // 2. Băm mật khẩu với Argon2id (Argon2 tự động sinh Salt ngẫu nhiên bằng CSPRNG)
      const hashedPassword = await argon2.hash(dto.password, {
        type: argon2.argon2id,  // Tiêu chuẩn Argon2id
        memoryCost: 2 ** 16,     // 64 MB bộ nhớ RAM (Memory-hard chống GPU)
        timeCost: 3,             // 3 lần lặp (Iterations)
        parallelism: 1,          // 1 luồng xử lý
      });

      // 3. Lưu vào Database (Chuỗi hash lưu trữ đã bao gồm cả Salt và Metadata)
      const newUser = this.userRepository.create({
        email: dto.email,
        password: hashedPassword, // TUYỆT ĐỐI KHÔNG LƯU DTO.PASSWORD GỐC
        role: 'member',
      });

      await this.userRepository.save(newUser);

      return {
        message: 'Đăng ký tài khoản thành công',
        userId: newUser.id,
      };
    } catch (error) {
      throw new InternalServerErrorException('Lỗi trong quá trình tạo tài khoản');
    }
  }

  /**
   * ĐĂNG NHẬP: Xác thực mật khẩu an toàn, chống Timing Attack
   */
  async validateUser(loginDto: LoginDto): Promise<User> {
    const { email, password } = loginDto;

    // 1. Tìm user theo email
    const user = await this.userRepository.findOne({ where: { email } });

    // CẢNH BÁO TIMING ATTACK:
    // Nếu user không tồn tại, vẫn nên chạy 1 phép verify giả để hacker không đo lường được
    // thời gian phản hồi (Response Time) để đoán xem email có tồn tại trong hệ thống hay không!
    if (!user) {
      // Chạy một hash giả tốn thời gian tương đương
      await argon2.verify('$argon2id$v=19$m=65536,t=3,p=1$c29tZXNhbHQ$fakehash', password).catch(() => {});
      throw new UnauthorizedException('Thông tin đăng nhập không chính xác');
    }

    // 2. So sánh mật khẩu bằng argon2.verify (tự động so sánh trong thời gian cố định - Constant Time)
    const isPasswordValid = await argon2.verify(user.password, password);
    if (!isPasswordValid) {
      throw new UnauthorizedException('Thông tin đăng nhập không chính xác');
    }

    // 3. Trả về user (nhưng đã ẩn password)
    delete user.password;
    return user;
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Không bao giờ dùng MD5, SHA-1, SHA-256 cho mật khẩu:** Các thuật toán này chỉ dành cho checksum tính toàn vẹn file.
- **Lựa chọn hàng đầu:** Dùng `argon2` với biến thể `argon2.argon2id`.
- **Thông báo lỗi đăng nhập đồng nhất:** Luôn trả về thông báo chung *"Thông tin đăng nhập không chính xác"* cho cả hai trường hợp (sai email hoặc sai password) để chống kỹ thuật rà quét tài khoản (Username Enumeration).
- **Phòng chống Timing Attack:** Kẻ tấn công có thể đo microsecond để biết email có tồn tại hay không. Hãy đảm bảo thời gian xử lý khi user không tồn tại xấp xỉ thời gian khi user tồn tại.

