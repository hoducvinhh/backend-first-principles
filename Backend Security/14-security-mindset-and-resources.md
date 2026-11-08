# Phần 14: Tư duy Bảo mật & Tài nguyên Học tập (Security Mindset & Resources)

> **Tư duy cốt lõi:**  
> *"Bảo mật không phải là một đích đến hay một tính năng làm xong là xong. Bảo mật là một tư duy phản biện liên tục trước mọi ranh giới dữ liệu và thiết lập hệ thống phòng thủ đa tầng (Defense in Depth) để không một mắt xích yếu nào có thể đánh sập toàn bộ ứng dụng."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Triết lý Cốt lõi: Ranh giới & Giả định (Boundaries & Assumptions)
Mọi lỗ hổng bảo mật trên thế giới đều bắt nguồn từ một hiện tượng: **Dữ liệu vượt qua ranh giới (Data crossing boundaries)** giữa các hệ sinh thái, giữa các ngôn ngữ, hoặc giữa các phân cấp quyền hạn:
- **SQL Injection:** Dữ liệu người dùng vượt qua ranh giới vào bộ phân tích cú pháp SQL.
- **Command Injection:** Dữ liệu người dùng vượt qua ranh giới vào trình thông dịch Shell của Hệ điều hành.
- **XSS:** Dữ liệu người dùng vượt qua ranh giới vào cấu trúc DOM/HTML của trình duyệt.
- **BOLA/BFLA:** Dữ liệu vượt qua ranh giới quyền hạn giữa User A và User B, hoặc giữa Member và Admin.

### 3 Câu hỏi Vàng trước khi Viết từng Dòng Code
Mỗi khi tiếp nhận một luồng dữ liệu mới, kỹ sư luôn phải tự chất vấn:
1. **Dữ liệu này đang vượt qua những ranh giới nào?** *(Browser $\rightarrow$ Backend $\rightarrow$ Database $\rightarrow$ OS)*.
2. **Mình đang đưa ra những giả định gì về dữ liệu này?** *(Nó có đúng định dạng không? Người gửi có thực sự là chủ sở hữu không?)*.
3. **Điều gì sẽ xảy ra nếu tất cả những giả định đó hoàn toàn sai lệch hoặc bị cố tình phá vỡ?**

---

### Mô hình Phòng thủ Đa tầng (Defense in Depth - 5 Lớp)

```
┌────────────────────────────────────────────────────────────────────────┐
│ TẦNG 1: INPUT VALIDATION (LÀM SẠCH ĐẦU VÀO)                            │
│ Kiểm tra kiểu dữ liệu, độ dài, regex, whitelist ngay tại Gateway/Pipe. │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ TẦNG 2: PARAMETERIZED OPERATIONS (CÔ LẬP LỆNH & DỮ LIỆU)               │
│ Tách biệt Code và Data: Parameterized Queries, Argument Arrays.        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ TẦNG 3: AUTHORIZATION AT POINT OF ACCESS (PHÂN QUYỀN TẬN GỐC)          │
│ Kiểm tra quyền sở hữu tại Repository/DB (WHERE user_id = $uid).        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ TẦNG 4: SECURITY HEADERS & BROWSER POLICIES (CHỐT CHẶN TRÌNH DUYỆT)    │
│ Helmet (CSP, X-Frame-Options), Cookie HttpOnly, Secure, SameSite.      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ TẦNG 5: MONITORING, AUDIT LOGGING & RATE LIMITING (THEO DÕI & GIỚI HẠN)│
│ Throttler giới hạn tần suất, ghi log kiểm toán các hành vi rà quét.   │
└────────────────────────────────────────────────────────────────────────┘
```

> **Ý nghĩa:** Không một tầng phòng thủ đơn lẻ nào là hoàn hảo. Nhưng khi xây dựng cả 5 tầng độc lập, kẻ tấn công buộc phải vượt qua toàn bộ 5 lớp cùng lúc mới có thể xâm nhập hệ thống — xác suất này gần như bằng 0!

---

## 2. Ánh xạ vào Hệ sinh thái NestJS: Bản đồ Kiến trúc Phòng thủ Toàn diện

Dưới đây là bức tranh tổng hợp cách 5 tầng phòng thủ ánh xạ trực tiếp vào các thành phần của **NestJS**:

| Tầng Phòng thủ | Thành phần tương ứng trong NestJS |
| :--- | :--- |
| **Tầng 1: Input Validation** | `ValidationPipe` + `class-validator` + `class-transformer` |
| **Tầng 2: Parameterized Operations** | `TypeORM QueryBuilder` (`:param`) / `Prisma.$queryRaw` |
| **Tầng 3: Authorization at Access** | `RolesGuard` + `@CurrentUser()` + Scoped Repository (`userId`) |
| **Tầng 4: Browser Policies** | `helmet()` + `enableCors()` + `res.cookie(..., { httpOnly, secure })` |
| **Tầng 5: Monitoring & Throttling** | `@nestjs/throttler` (Redis) + `Winston/Pino Logger` + Audit Interceptor |

---

## 3. Thực hành Code NestJS: Kiến trúc Mẫu Phòng thủ 5 Tầng Chuẩn Production

Dưới đây là một ví dụ tích hợp đầy đủ cả 5 tầng bảo mật trong một module nghiệp vụ của NestJS:

```typescript
// orders/orders.controller.ts - MÔ THỨC PHÒNG THỦ 5 TẦNG HOÀN CHỈNH
import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  UseGuards,
  UseInterceptors,
  ParseUUIDPipe,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { Throttle } from '@nestjs/throttler';
import { JwtAuthGuard } from '../common/guards/jwt-auth.guard';
import { RolesGuard } from '../common/guards/roles.guard';
import { CurrentUser } from '../common/decorators/current-user.decorator';
import { AuditLogInterceptor } from '../common/interceptors/audit-log.interceptor';
import { CreateOrderDto } from './dto/create-order.dto';
import { OrdersService } from './orders.service';

@Controller('orders')
@UseGuards(JwtAuthGuard, RolesGuard) // TẦNG 3: Guard kiểm tra danh tính và vai trò
@UseInterceptors(AuditLogInterceptor) // TẦNG 5: Audit Log ghi lại mọi thao tác
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}

  /**
   * TẦNG 1: DTO Validation chặn đứng dữ liệu bẩn
   * TẦNG 5: Rate Limiting chống spam tạo đơn hàng
   */
  @Post()
  @HttpCode(HttpStatus.CREATED)
  @Throttle({ short: { limit: 2, ttl: 1000 } }) // Tối đa 2 đơn/giây
  async createOrder(
    @Body() createOrderDto: CreateOrderDto,
    @CurrentUser('userId') currentUserId: string,
  ) {
    return this.ordersService.create(createOrderDto, currentUserId);
  }

  /**
   * TẦNG 3: BOLA Protection - Sử dụng ParseUUIDPipe chặn quét ID
   */
  @Get(':id')
  async getOrder(
    @Param('id', ParseUUIDPipe) orderId: string,
    @CurrentUser('userId') currentUserId: string,
  ) {
    return this.ordersService.findOneScoped(orderId, currentUserId);
  }
}
```

```typescript
// orders/orders.service.ts - TẦNG 2 & TẦNG 3 TẠI SERVICE LAYER
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Order } from './entities/order.entity';
import { CreateOrderDto } from './dto/create-order.dto';

@Injectable()
export class OrdersService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
  ) {}

  async create(dto: CreateOrderDto, userId: string): Promise<Order> {
    const order = this.orderRepository.create({
      ...dto,
      userId: userId, // Backend chủ động gán userId, không nhận từ client
      status: 'pending',
    });
    return this.orderRepository.save(order);
  }

  /**
   * TẦNG 2: Parameterized Query qua QueryBuilder (:orderId, :userId)
   * TẦNG 3: Ràng buộc quyền sở hữu và trả về 404 Not Found chống rò rỉ thông tin
   */
  async findOneScoped(orderId: string, userId: string): Promise<Order> {
    const order = await this.orderRepository
      .createQueryBuilder('order')
      .where('order.id = :orderId AND order.userId = :userId', { orderId, userId })
      .getOne();

    if (!order) {
      // Ẩn sự tồn tại của đơn hàng nếu không thuộc về user này
      throw new NotFoundException('Không tìm thấy đơn hàng yêu cầu');
    }

    return order;
  }
}
```

---

## 4. Các Tài nguyên Học tập Bảo mật Hàng đầu Khuyến nghị

1. **PortSwigger Web Security Academy (`https://portswigger.net/web-security`):**
   - Nền tảng thực hành bảo mật số 1 thế giới của công ty sáng tạo ra **Burp Suite**.
   - Cung cấp kiến thức lý thuyết chuyên sâu kèm hàng trăm phòng Lab thực hành tương tác miễn phí 100% cho từng loại lỗ hổng: SQLi, XSS, CSRF, SSRF, JWT Attacks, OS Command Injection, OAuth2 vulnerabilities...
2. **OWASP Top 10 (`https://owasp.org/www-project-top-ten/`):**
   - Bảng xếp hạng cập nhật định kỳ về 10 nhóm nguy cơ bảo mật ứng dụng phổ biến và nghiêm trọng nhất toàn cầu (Broken Access Control, Cryptographic Failures, Injection, Security Misconfiguration...).
3. **OWASP Cheat Sheet Series (`https://cheatsheetseries.owasp.org/`):**
   - Bộ cẩm nang tra cứu nhanh cực kỳ thực chiến cho lập trình viên Backend theo từng chuyên đề:
     - *Node.js / Express Security Cheat Sheet*
     - *Authentication Cheat Sheet*
     - *Password Storage Cheat Sheet*
     - *Session Management Cheat Sheet*
     - *REST Security Cheat Sheet*
4. **Lucia Auth Documentation (`https://lucia-auth.com/`):**
   - Cẩm nang hướng dẫn tuyệt vời về các nguyên lý xác thực hiện đại, giải thích sâu về session management, hashing mật khẩu và bảo mật cookie.

