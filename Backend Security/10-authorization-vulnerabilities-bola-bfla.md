# Phần 10: Lỗ hổng Phân quyền: BOLA & BFLA (Authorization Vulnerabilities)

> **Tư duy cốt lõi:**  
> *"Đừng bao giờ để việc người dùng đã đăng nhập đánh lừa bạn. Xác thực (Authentication) chỉ trả lời câu hỏi 'Họ là ai?'. Tại từng điểm truy cập cơ sở dữ liệu, bạn bắt buộc phải kiểm tra Phân quyền (Authorization): 'Họ có thực sự sở hữu dòng dữ liệu này hay không?'"*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Phân biệt Rõ ràng: Authentication vs Authorization
- **Authentication (Xác thực):** Xác minh danh tính của người dùng (Token hợp lệ, mật khẩu đúng).
- **Authorization (Phân quyền):** Xác định quyền hạn của người dùng trên từng tài nguyên hoặc chức năng cụ thể.

### Bẫy tư duy của Kỹ sư Backend
Nhiều dev lầm tưởng rằng chỉ cần đặt middleware xác thực ở Router là toàn bộ hệ thống đã an toàn:
```
Request ──► [Router / AuthGuard] ──► [Controller] ──► [Service] ──► [Repository/DB]
```
Thực tế, vượt qua `AuthGuard` không có nghĩa là người dùng được phép xem hóa đơn, sửa đơn hàng hay xóa bài viết của một người dùng khác!

---

### Hai Dạng Tấn công Phân quyền Điển hình

```
          ▲
          │  [VERTICAL ESCALATION / BFLA]
          │  Leo thang quyền lực theo CHIỀU DỌC
          │  User bình thường (Member) ──► Chiếm quyền Chức năng Admin (/admin/...)
          │
──────────┼──────────────────────────────────────────────────────────►
          │
          ▼  [HORIZONTAL ESCALATION / BOLA / IDOR]
             Mở rộng phạm vi dữ liệu theo CHIỀU NGANG
             User A ──► Đọc trộm hoặc chỉnh sửa Tài nguyên của User B
```

#### 1. BOLA (Broken Object Level Authorization) / IDOR
- **Kịch bản:** User A sở hữu hóa đơn 1 và 2. Nhưng User A gọi API `GET /invoices?id=5`.
- **Lỗi code:** Backend chạy `SELECT * FROM invoices WHERE id = 5;` $\rightarrow$ Trả về hóa đơn số 5 của User B! Hacker chỉ cần viết vòng lặp tăng ID từ 1 đến 100,000 để lấy trộm toàn bộ dữ liệu tài chính của khách hàng.
- **Khắc phục:** Luôn ràng buộc quyền sở hữu tới tận câu query DB:
  ```sql
  SELECT * FROM invoices WHERE id = $id AND user_id = $currentUserId;
  ```

#### 2. Nghệ thuật Xử lý Mã Lỗi HTTP: Tránh Rò rỉ Thông tin (403 vs 404)
- **Cách làm SAI (Rò rỉ thông tin):**
  1. Query DB: Thấy hóa đơn số 5 tồn tại trong bảng.
  2. Kiểm tra code: `if (invoice.userId !== currentUser.id) return 403 Forbidden;`
  3. **Hậu quả:** Kẻ tấn công biết chắc chắn ID 5 tồn tại trong hệ thống (dù không xem được). Chúng sẽ lập danh sách ID hợp lệ để tấn công lừa đảo Social Engineering.
- **Cách làm ĐÚNG (Bảo vệ thông tin tuyệt đối):**
  1. Query DB: `WHERE id = 5 AND user_id = currentUser.id`.
  2. Database trả về 0 dòng (No rows).
  3. Trả về **`404 Not Found`**.
  4. **Kết quả:** Kẻ tấn công hoàn toàn không thể phân biệt được ID 5 không tồn tại hay thuộc về người khác!

#### 3. BFLA (Broken Function Level Authorization)
- Lỗ hổng cho phép người dùng quyền thấp truy cập vào các chức năng quản trị cấp cao (ví dụ: `GET /admin/users`, `DELETE /admin/system-reset`).
- **Cạm bẫy "Bảo mật bằng sự che giấu" (Security through Obscurity):** Nghĩ rằng không hiển thị nút Admin trên màn hình thì user không biết. Thực tế hacker mở Network tab hoặc đọc file JS bundle là tìm ra URL ngay!

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong NestJS, giải pháp chuẩn mực để xử lý BFLA và BOLA bao gồm:
1. **Chống BFLA:** Sử dụng **Custom Decorator** `@Roles('admin')` kết hợp với **`RolesGuard`** để chặn đứng ngay từ tầng Controller.
2. **Chống BOLA:** Sử dụng Custom Decorator `@CurrentUser()` để lấy ID của người dùng từ token, sau đó **ép buộc điều kiện `userId` tại tầng Service & Repository**. Trả về `NotFoundException` (404).

---

## 3. Thực hành Code NestJS: Phòng thủ BOLA và BFLA Toàn diện

### 1. Chống BFLA: Xây dựng Role-Based Access Control (RBAC) với Guard

#### Decorator khai báo quyền: `@Roles()`
```typescript
// common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

#### Guard kiểm tra vai trò: `RolesGuard`
```typescript
// common/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // Nếu route không yêu cầu role cụ thể -> cho phép qua
    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();

    // Kiểm tra role của user có nằm trong danh sách được phép hay không
    const hasRole = requiredRoles.includes(user?.role);
    if (!hasRole) {
      throw new ForbiddenException('Bạn không có quyền hạn quản trị để truy cập chức năng này!');
    }

    return true;
  }
}
```

---

### 2. Custom Decorator trích xuất Người dùng: `@CurrentUser()`
```typescript
// common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user; // Được JwtStrategy hoặc SessionGuard gán vào
    return data ? user?.[data] : user;
  },
);
```

---

### 3. Chống BOLA: Xử lý tại Service & Repository với Mã lỗi 404

#### Controller định tuyến an toàn
```typescript
// invoices/invoices.controller.ts
import { Controller, Get, Param, Delete, UseGuards, ParseUUIDPipe } from '@nestjs/common';
import { JwtAuthGuard } from '../common/guards/jwt-auth.guard';
import { RolesGuard } from '../common/guards/roles.guard';
import { Roles } from '../common/decorators/roles.decorator';
import { CurrentUser } from '../common/decorators/current-user.decorator';
import { InvoicesService } from './invoices.service';

@Controller('invoices')
@UseGuards(JwtAuthGuard, RolesGuard)
export class InvoicesController {
  constructor(private readonly invoicesService: InvoicesService) {}

  /**
   * BẢO VỆ BOLA: User chỉ xem được hóa đơn của chính mình
   * Dùng UUID Pipe để chặn kẻ gian đoán ID tự tăng
   */
  @Get(':id')
  async getInvoice(
    @Param('id', ParseUUIDPipe) id: string,
    @CurrentUser('userId') currentUserId: string,
  ) {
    return this.invoicesService.findOneForUser(id, currentUserId);
  }

  /**
   * BẢO VỆ BFLA: Chỉ Admin mới được xem toàn bộ hóa đơn hệ thống
   */
  @Get('admin/all')
  @Roles('admin')
  async getAllInvoicesForAdmin() {
    return this.invoicesService.findAll();
  }
}
```

#### Service thực thi truy vấn Scoped Data (Chốt chặn BOLA cốt lõi)
```typescript
// invoices/invoices.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Invoice } from './entities/invoice.entity';

@Injectable()
export class InvoicesService {
  constructor(
    @InjectRepository(Invoice)
    private readonly invoiceRepository: Repository<Invoice>,
  ) {}

  /**
   * PHÒNG CHỐNG BOLA TRIỆT ĐỂ:
   * Luôn luôn kèm điều kiện userId trong câu query DB!
   */
  async findOneForUser(invoiceId: string, userId: string): Promise<Invoice> {
    const invoice = await this.invoiceRepository.findOne({
      where: {
        id: invoiceId,
        userId: userId, // BẮT BUỘC: Ràng buộc quyền sở hữu dữ liệu
      },
    });

    // NẾU KHÔNG TÌM THẤY: NÉM LỖI 404 NOT FOUND (KHÔNG DÙNG 403 FORBIDDEN!)
    if (!invoice) {
      throw new NotFoundException('Không tìm thấy hóa đơn yêu cầu');
    }

    return invoice;
  }

  /**
   * XÓA HÓA ĐƠN AN TOÀN
   */
  async deleteInvoiceForUser(invoiceId: string, userId: string): Promise<{ success: boolean }> {
    const result = await this.invoiceRepository.delete({
      id: invoiceId,
      userId: userId, // Ràng buộc chỉ xóa nếu đúng chủ sở hữu
    });

    if (result.affected === 0) {
      throw new NotFoundException('Không tìm thấy hóa đơn để xóa');
    }

    return { success: true };
  }

  async findAll(): Promise<Invoice[]> {
    return this.invoiceRepository.find();
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Không bao giờ tin tưởng client-side ID:** Dù client có gửi `invoiceId` lên URL, điều kiện `userId` bắt buộc phải được lấy từ **Authentication Token (`req.user.id`)**, tuyệt đối không lấy `userId` từ query hay body của request!
- **Sử dụng UUID thay cho Sequential ID:** Thay vì dùng `id: number` (1, 2, 3), hãy cấu hình `@PrimaryGeneratedColumn('uuid')` trong Entity TypeORM để vô hiệu hóa việc kẻ tấn công tự động dò quét ID.
- **Quy tắc 404 thay vì 403:** Không tiết lộ cho kẻ tấn công biết một tài nguyên có tồn tại hay không.
- **Áp dụng cho mọi hành động:** Cả `SELECT`, `UPDATE`, `DELETE` đều phải kèm `where: { id, userId }`.

