# Phần 4: Phòng chống SQL Injection bằng Parameterized Queries

> **Tư duy cốt lõi:**  
> *"Tách biệt hoàn toàn Cấu trúc câu lệnh (Code) khỏi Dữ liệu tham số (Data). Khi cây cú pháp đã được biên dịch xong, dữ liệu người dùng gửi vào sẽ mãi mãi chỉ là dữ liệu, không bao giờ có thể trở thành lệnh thực thi."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Cơ chế hoạt động của Parameterized Queries (Prepared Statements)
Khi sử dụng Truy vấn Tham số hóa (Parameterized Query / Prepared Statement), hệ thống chia quá trình thực thi thành 2 giai đoạn gửi tới Database Engine:

```
GIAI ĐOẠN 1: GỬI CẤU TRÚC (STRUCTURE)
Client ──► DB Parser: "SELECT * FROM users WHERE email = $1"
          DB biên dịch thành Cây cú pháp trừu tượng (Abstract Syntax Tree - AST).
          Cấu trúc điều khiển đã được CHỐT VĨNH VIỄN.

GIAI ĐOẠN 2: GỬI DỮ LIỆU (DATA)
Client ──► DB Engine: ["' OR '1'='1' --"]
          DB đưa giá trị vào slot $1 dưới dạng THUẦN CHUỖI KÝ TỰ (Plain String).
          Không chạy lại Parser! Mọi ký tự đặc biệt chỉ là ký tự thông thường.
```

- **Kết quả:** Database tìm kiếm một bản ghi có email bằng đúng chuỗi ký tự `' OR '1'='1' -- `. Vì không có tài khoản nào có email quái dị như vậy, DB trả về 0 kết quả $\rightarrow$ Cuộc tấn công bị vô hiệu hóa 100%.

### Cạm bẫy NoSQL / MongoDB Operator Injection
- Nhiều người lầm tưởng dùng MongoDB thì không bao giờ bị Injection.
- **Thực tế:** MongoDB dùng query object JSON. Nếu nhận trực tiếp JSON từ người dùng, kẻ tấn công có thể chèn toán tử `$ne` (not equal), biến câu tìm kiếm thành:
  ```json
  { "email": { "$ne": null } }
  ```
  Truy vấn này sẽ khớp với tất cả tài khoản trong cơ sở dữ liệu!

---

## 2. Ánh xạ vào Hệ sinh thái NestJS (TypeORM, Prisma, Mongoose)

Trong NestJS, việc áp dụng Parameterized Queries phụ thuộc vào công cụ truy xuất dữ liệu bạn lựa chọn:
1. **TypeORM QueryBuilder:** Luôn dùng cú pháp `:paramName` kèm object tham số `{ paramName: value }`.
2. **TypeORM Raw Query:** Luôn truyền mảng tham số vào vị trí tham số thứ hai của `query(sql, [param1, param2])`.
3. **Prisma ORM:** Sử dụng tagged template literal `$queryRaw` chuẩn (tự động tham số hóa), **TUYỆT ĐỐI TRÁNH `$queryRawUnsafe`**.
4. **Mongoose / NoSQL:** Sử dụng `class-validator` với kiểu nguyên thủy tường minh `@IsString()` hoặc middleware `express-mongo-sanitize`.

---

## 3. Thực hành Code NestJS: Cách viết Truy vấn An toàn Chuẩn mực

###  1. TypeORM: QueryBuilder chuẩn với Parameterized Slots
```typescript
// users.service.ts - CHUẨN AN TOÀN VỚI TYPEORM QUERYBUILDER
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async findByEmailSafe(email: string): Promise<User> {
    // SỬ DỤNG PARAMETERIZED SLOT :email VÀ TRUYỀN VALUE RIÊNG BIỆT
    const user = await this.userRepository
      .createQueryBuilder('user')
      .where('user.email = :email', { email }) // <-- AN TOÀN TUYỆT ĐỐI
      .getOne();

    if (!user) {
      throw new NotFoundException('Không tìm thấy người dùng');
    }

    return user;
  }

  // Xử lý an toàn với ORDER BY (Whitelist các trường được phép sắp xếp)
  async getProductsSorted(sortBy: string, order: 'ASC' | 'DESC') {
    const allowedSortFields = ['price', 'createdAt', 'name'];
    
    // Kiểm tra Whitelist: Nếu client truyền trường lạ -> fallback về mặc định
    const safeSortField = allowedSortFields.includes(sortBy) ? sortBy : 'createdAt';
    const safeOrder = order === 'DESC' ? 'DESC' : 'ASC';

    return this.userRepository
      .createQueryBuilder('product')
      .orderBy(`product.${safeSortField}`, safeOrder) // An toàn vì đã qua Whitelist
      .getMany();
  }
}
```

---

###  2. TypeORM: Raw Query với Placeholder `$1, $2`
Nếu bắt buộc phải viết câu lệnh SQL thuần phức tạp, hãy truyền tham số qua mảng:

```typescript
// report.service.ts - CHUẨN AN TOÀN VỚI RAW SQL
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class ReportService {
  constructor(private readonly dataSource: DataSource) {}

  async getUserReport(status: string, minAge: number) {
    // Sử dụng placeholder $1, $2 (PostgreSQL) hoặc ? (MySQL)
    const sql = `
      SELECT id, full_name, email 
      FROM "user" 
      WHERE status = $1 AND age >= $2
    `;

    // Truyền mảng giá trị vào tham số thứ hai
    return this.dataSource.query(sql, [status, minAge]); // AN TOÀN!
  }
}
```

---

###  3. Prisma ORM: Phân biệt `$queryRaw` vs `$queryRawUnsafe`
```typescript
// prisma-user.service.ts - CHUẨN AN TOÀN VỚI PRISMA
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { Prisma } from '@prisma/client';

@Injectable()
export class PrismaUserService {
  constructor(private readonly prisma: PrismaService) {}

  async findUserSafe(email: string) {
    // Prisma $queryRaw tự động biến các biến bên trong template string thành parameterized query!
    return this.prisma.$queryRaw<User[]>`
      SELECT * FROM "User" WHERE email = ${email}
    `; // <-- AN TOÀN: Prisma chuyển thành: SELECT * FROM "User" WHERE email = $1

    // CẢNH BÁO: TUYỆT ĐỐI KHÔNG DÙNG $queryRawUnsafe nối chuỗi:
    // return this.prisma.$queryRawUnsafe(`SELECT * FROM "User" WHERE email = '${email}'`); // <-- DÍNH SQLi!
  }
}
```

---

###  4. Mongoose (MongoDB): Chống NoSQL Operator Injection
Để ngăn chặn kẻ tấn công gửi object `{ "$ne": null }` thay vì string:

#### Cách 1: Ràng buộc chặt chẽ kiểu dữ liệu bằng DTO trong NestJS
```typescript
// dto/login.dto.ts
import { IsString, IsNotEmpty } from 'class-validator';

export class LoginDto {
  // BẮT BUỘC PHẢI LÀ STRING: Nếu attacker gửi object {"$ne": null},
  // ValidationPipe sẽ ném lỗi 400 Bad Request ngay lập tức!
  @IsString({ message: 'Email phải là chuỗi ký tự' })
  @IsNotEmpty()
  email: string;

  @IsString()
  @IsNotEmpty()
  password: string;
}
```

#### Cách 2: Tích hợp Middleware `mongo-sanitize` trong `main.ts`
```typescript
// main.ts
import mongoSanitize from 'express-mongo-sanitize';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Tự động tìm và loại bỏ tất cả các key bắt đầu bằng dấu $ trong req.body, req.query, req.params
  app.use(mongoSanitize());

  await app.listen(3000);
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **QueryBuilder:** Cú pháp `:param` + `{ param: value }` là chân ái.
- **Raw SQL:** Luôn luôn dùng placeholder (`$1`, `?`) + array tham số.
- **Mệnh đề không thể Parameterize (như `ORDER BY`, tên bảng):** Bắt buộc phải dùng kỹ thuật **Whitelist Validation** (so sánh với một danh sách các chuỗi cố định được cho phép).
- **NoSQL:** Luôn dùng `@IsString()` trong DTO của NestJS để ngăn kẻ gian gửi query operator object.

