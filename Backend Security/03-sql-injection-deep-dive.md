# Phần 3: Phân tích chuyên sâu SQL Injection (SQL Injection Deep Dive)

> **Tư duy cốt lõi:**  
> *"Dấu nháy đơn `'`, dấu chấm phẩy `;` và hai dấu gạch nối `--` không hề nguy hiểm khi đứng một mình. Chúng chỉ trở thành vũ khí phá hoại khi bạn nối chuỗi dữ liệu người dùng trực tiếp vào câu lệnh SQL."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Bản chất của SQL Injection (SQLi)
- Xảy ra khi lập trình viên xây dựng câu lệnh SQL bằng phương pháp **nối chuỗi (String Concatenation / Template Literals)** với dữ liệu do người dùng nhập vào.
- Trình phân tích cú pháp (SQL Parser) của cơ sở dữ liệu bị đánh lừa: ký tự điều khiển trong chuỗi nhập vào bị hiểu thành cú pháp cấu trúc câu lệnh.

### Giải phẫu các Kịch bản Tấn công Điển hình

#### 1. Đòn đánh Bypass Đăng nhập & Rò rỉ Dữ liệu (`' OR '1'='1' -- `)
- Câu lệnh gốc của backend:
  ```sql
  SELECT * FROM users WHERE email = '$userInput';
  ```
- Kẻ tấn công nhập: `' OR '1'='1' -- `
- Câu lệnh thực thi:
  ```sql
  SELECT * FROM users WHERE email = '' OR '1'='1' -- ';
  ```
- **Hệ quả:** Dấu `'` đầu tiên kết thúc chuỗi email. Mệnh đề `OR '1'='1'` luôn đúng. Dấu `-- ` chú thích loại bỏ dấu nháy đơn đóng còn lại. Câu lệnh tương đương `SELECT * FROM users;` $\rightarrow$ Trả về toàn bộ người dùng, kẻ tấn công chiếm quyền tài khoản đầu tiên.

#### 2. Đòn đánh Phá hủy Cấu trúc DDL (`'; DROP TABLE users; -- `)
- Sử dụng dấu `;` để ngắt câu lệnh hiện tại và kích hoạt câu lệnh thứ hai: `DROP TABLE users;`.
- **Nguyên tắc phân quyền DB:** Tài khoản kết nối DB của backend ứng dụng production **CHỈ ĐƯỢC PHÉP CÓ QUYỀN DML** (`SELECT`, `INSERT`, `UPDATE`, `DELETE`), **TUYỆT ĐỐI CẤM QUYỀN DDL** (`DROP`, `ALTER`, `TRUNCATE`).

#### 3. Các dạng khai thác nâng cao khác
- **`UNION SELECT`:** Trích xuất trái phép dữ liệu từ bảng thẻ tín dụng, giao dịch thanh toán sang kết quả trả về.
- **File System / OS Execution:** Khai thác các hàm DB nội tại (`LOAD_FILE()`, `pg_read_file`, `xp_cmdshell`) để đọc mã nguồn hoặc chạy shell command trên máy chủ chứa database.

---

## 2. Ánh xạ vào Hệ sinh thái NestJS (TypeORM / Prisma)

Nhiều lập trình viên NestJS lầm tưởng rằng: *"Tôi dùng TypeORM / Prisma rồi nên tôi hoàn toàn miễn nhiễm với SQL Injection"*.  
**Đây là một sai lầm chết người!** Cả TypeORM và Prisma đều có thể bị dính SQL Injection nếu lập trình viên sử dụng:
1. `dataSource.query(...)` hoặc `manager.query(...)` kèm nối chuỗi.
2. `repository.createQueryBuilder(...)` nhưng lại nối chuỗi trực tiếp vào các mệnh đề `.where()`, `.andWhere()`, `.orderBy()`.

---

## 3. Thực hành Code NestJS: Nhận diện & Tránh Lỗ hổng SQLi

### ❌ Trường hợp 1: Dùng Raw Query với Template Literal (RẤT NGUY HIỂM)
Lập trình viên dùng `DataSource` để chạy câu query thuần và dùng dấu backtick để nhúng biến:

```typescript
// auth.service.ts - VULNERABLE: Lỗ hổng SQL Injection kinh điển trong NestJS
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class AuthService {
  constructor(private readonly dataSource: DataSource) {}

  async loginVulnerable(email: string) {
    // NGUY HIỂM TỘT CÙNG: Dùng Template String nối biến trực tiếp
    const rawQuery = `SELECT * FROM "user" WHERE "email" = '${email}'`;
    
    // Nếu email là: ' OR '1'='1' --
    // Query thực thi: SELECT * FROM "user" WHERE "email" = '' OR '1'='1' --'
    const users = await this.dataSource.query(rawQuery);

    if (!users || users.length === 0) {
      throw new UnauthorizedException('Không tìm thấy tài khoản');
    }

    return users[0]; // Trả về tài khoản đầu tiên trong database!
  }
}
```

---

### ❌ Trường hợp 2: Dùng TypeORM QueryBuilder nhưng vẫn nối chuỗi
Kể cả khi sử dụng QueryBuilder chuyên nghiệp, nếu không dùng placeholder thì vẫn dính SQLi như thường:

```typescript
// users.service.ts - VULNERABLE: Dùng QueryBuilder nhưng nối chuỗi sai lầm
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async searchUsersByEmail(email: string) {
    // SAI LẦM: Nối chuỗi bên trong mệnh đề where()
    return this.userRepository
      .createQueryBuilder('user')
      .where(`user.email = '${email}'`) // <-- SQL Injection vẫn xảy ra tại đây!
      .getMany();
  }
}
```

---

### ❌ Trường hợp 3: SQL Injection tại mệnh đề `ORDER BY`
Nhiều dev biết bảo vệ mệnh đề `WHERE` nhưng lại quên mất mệnh đề sắp xếp `ORDER BY`:

```typescript
// products.service.ts - VULNERABLE: SQLi tại orderBy
async getProducts(sortBy: string, order: 'ASC' | 'DESC') {
  // sortBy do user truyền lên: "price; DROP TABLE products; --"
  return this.productRepository
    .createQueryBuilder('product')
    .orderBy(`product.${sortBy}`, order) // NGUY HIỂM!
    .getMany();
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Không bao giờ dùng dấu cộng `+` hoặc backtick `` `${}` `` để ghép chuỗi vào câu lệnh SQL**, dù là raw query hay trong các method của ORM.
- **QueryBuilder không tự động bảo vệ bạn:** Bạn phải cung cấp tham số theo đúng cơ chế parameter mapping của nó (sẽ được phân tích chi tiết ở Phần 4).
- **Cấu hình Database Role hạn chế:** Trong file cấu hình kết nối TypeORM (`ormconfig` hoặc `TypeOrmModule.forRoot`), hãy chắc chắn rằng user DB chỉ có các quyền hạn tối thiểu cần thiết để thao tác dữ liệu.

