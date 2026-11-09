# Module 2: Cấu trúc Hệ thống (System Structure)

> **Tư duy cốt lõi:**  
> *"Cấu trúc của một hệ thống quyết định hành vi của nó (System structure determines system behavior). Nếu một hệ thống liên tục sinh ra lỗi hay tắc nghẽn, việc thay người hay sửa từng đoạn code chỉ là phần ngọn; gốc rễ nằm ở cách các bộ phận được liên kết và phụ thuộc vào nhau."*

---

## 1. Mối quan hệ giữa Cấu trúc và Hành vi

Tại sao cùng một lượng máy chủ, cùng một lượng database, nhưng một kiến trúc này có thể chịu được hàng triệu người dùng trong khi kiến trúc khác lại sập ngay khi có 10,000 request?
- **Câu trả lời nằm ở CẤU TRÚC (Structure):** Cách các thành phần được sắp xếp, phân cấp, kết nối và phụ thuộc lẫn nhau.
- Cấu trúc tạo ra các luồng di chuyển của dữ liệu, quyết định nơi nào sẽ chịu tải, nơi nào có thể trở thành điểm nghẽn và nơi nào sẽ xảy ra phản ứng dây chuyền khi có sự cố.

---

## 2. Dependency (Sự phụ thuộc)

```
THIẾT KẾ PHỤ THUỘC CHẶT CHẼ (TIGHT COUPLING) - CỰC KỲ DỄ GÂY CASCADING FAILURE
┌──────────────┐  HTTP Sync   ┌──────────────┐  HTTP Sync   ┌──────────────┐
│  Service A   │─────────────►│  Service B   │─────────────►│  Service C   │
│  (Frontend)  │              │   (Order)    │              │  (Payment)   │
└──────────────┘              └──────────────┘              └──────┬───────┘
                                                                   │
                                                                   ▼
                                                            ┌──────────────┐
                                                            │  DB Payment  │
                                                            │   (DEADLOCK) │
                                                            └──────────────┘
Khi DB Payment bị chậm ──► Service C bị nghẽn thread ──► Service B cạn pool ──► Toàn hệ thống sập!
```

### 2.1. Bản chất của Dependency
- Một thành phần $A$ được coi là phụ thuộc vào thành phần $B$ ($A \rightarrow B$) nếu $A$ cần sự hiện diện, dữ liệu hoặc dịch vụ của $B$ để hoàn thành chức năng của chính mình.
- **Các dạng phụ thuộc trong phần mềm:**
  1. **Runtime Dependency (Phụ thuộc lúc chạy):** Service A gọi Service B qua mạng. Nếu B chết, A không thể xử lý xong request.
  2. **Compile-time / Code Dependency (Phụ thuộc mã nguồn):** Module A import class của Module B. Thay đổi trong code của B bắt buộc A phải compile lại.
  3. **Data Dependency (Phụ thuộc dữ liệu):** Hai service cùng chia sẻ một bảng trong Database. Thay đổi schema ở một nơi sẽ làm gãy service còn lại.
  4. **Infrastructure Dependency (Phụ thuộc hạ tầng):** Phụ thuộc vào DNS, NTP Server, AWS S3, Cloudflare.

### 2.2. Phân loại Coupling: Tight Coupling vs. Loose Coupling

| Đặc tính | Phụ thuộc Chặt chẽ (Tight Coupling) | Phụ thuộc Lỏng lẻo (Loose Coupling) |
| :--- | :--- | :--- |
| **Giao tiếp** | Gọi hàm trực tiếp, RPC đồng bộ (Sync HTTP REST). | Hướng sự kiện bất đồng bộ (Async Event Bus / Queue). |
| **Nhận biết (Awareness)** | Service A biết rõ địa chỉ IP, domain, schema của Service B. | Service A chỉ bắn event `OrderCreated` vào Kafka, không quan tâm ai nghe. |
| **Tính độc lập** | Phải deploy cùng nhau (Lock-step deployment). | Có thể deploy độc lập, rollback độc lập. |
| **Khả năng chịu lỗi** | Dễ sập lan truyền dây chuyền (**Cascading Failure**). | Đệm lỗi tốt (Fault Isolation) nhờ Message Queue lưu trữ tạm thời. |

### 2.3. Hiểm họa: Single Point of Failure (SPOF) & Cyclic Dependency
- **SPOF (Điểm lỗi đơn lẻ):** Một thành phần duy nhất mà nếu nó ngừng hoạt động, toàn bộ hệ thống sẽ tê liệt (ví dụ: Một con Master Database duy nhất không có replica, một Service Auth tập trung mà mọi request đều phải gọi qua).
- **Cyclic Dependency (Phụ thuộc vòng tròn):** $A \rightarrow B \rightarrow C \rightarrow A$. Đây là cơn ác mộng kiến trúc: rất khó khởi động hệ thống, dễ gây Deadlock phân tán và cực kỳ khó gỡ lỗi.

---

## 3. Hierarchy (Hệ thống Phân cấp)

```
┌────────────────────────────────────────────────────────────────────────┐
│                   HỆ THỐNG PHÂN CẤP (HIERARCHY)                        │
│                                                                        │
│   TẦNG DOANH NGHIỆP: E-Commerce Platform                              │
│   ├── SUB-SYSTEM 1: Core Commerce (Quản lý Hàng hóa & Đặt hàng)        │
│   │   ├── MODULE: Catalog Service                                      │
│   │   └── MODULE: Inventory Service                                    │
│   │                                                                    │
│   ├── SUB-SYSTEM 2: Financial (Thanh toán & Kế toán)                   │
│   │   ├── MODULE: Payment Gateway Adapter                              │
│   │   └── MODULE: Ledger & Accounting Service                          │
│   │                                                                    │
│   └── SUB-SYSTEM 3: Logistics (Giao nhận & Vận tải)                    │
│       ├── MODULE: Warehouse Management                                 │
│       └── MODULE: Shipping Partner Integrations                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 3.1. Tại sao Hệ thống cần Phân cấp?
- **Nguyên lý giới hạn nhận thức:** Não bộ con người không thể cùng một lúc nắm bắt hàng nghìn mối quan hệ phức tạp. Phân cấp (Hierarchy) giúp chia nhỏ bài toán thành các phân hệ (Subsystems) và các khối module độc lập.
- **Tính tự trị cục bộ (Local Subsystem Autonomy):** Mỗi hệ thống con có thể tự duy trì sự cân bằng của chính nó mà không cần làm phiền đến tầng cao hơn. Ví dụ: Bộ điều khiển máy lạnh trong phòng tự điều hòa nhiệt độ mà không cần hỏi ý kiến của ban quản lý toàn bộ tòa nhà.
- **Sự gắn kết cao và giảm tải giao tiếp:** Các thành phần bên trong cùng một hệ thống con trao đổi thông tin với tần suất dày đặc, trong khi giao tiếp giữa các hệ thống con với nhau diễn ra qua các Interface hoặc API chuẩn hóa với tần suất thấp hơn.

### 3.2. Hierarchy trong Kiến trúc Phần mềm
1. **Kiến trúc phân tầng ứng dụng (Clean / Hexagonal Architecture):**
   - *Frameworks & Drivers* $\rightarrow$ *Interface Adapters (Controllers)* $\rightarrow$ *Use Cases (Application Logic)* $\rightarrow$ *Entities (Domain Model Core)*.
   - Nguyên tắc phụ thuộc (The Dependency Rule): Chiều phụ thuộc luôn hướng vào trong, bảo vệ phần lõi nghiệp vụ độc lập với database hay web framework.
2. **Kiến trúc hạ tầng Cloud:**
   - *Region* $\rightarrow$ *Availability Zone (AZ)* $\rightarrow$ *Virtual Private Cloud (VPC)* $\rightarrow$ *Kubernetes Cluster* $\rightarrow$ *Node* $\rightarrow$ *Pod* $\rightarrow$ *Container* $\rightarrow$ *Thread*.

---

## 4. Interconnection (Mạng lưới Liên kết)

Cách các thành phần kết nối với nhau định hình nên topology của toàn bộ hệ thống backend:

```
    A. POINT-TO-POINT                B. HUB-AND-SPOKE             C. EVENT-DRIVEN BUS
    
     [S1]───────[S2]                       [S1]                         [S1]       [S2]
       \       /                             \                            │          │
        \     /                               ▼                           ▼          ▼
         [S3]                               [BUS]                 ═════════════════════════
        /     \                               ▲                           KAFKA BUS
       /       \                             /                    ═════════════════════════
     [S4]───────[S5]                       [S2]                         ▲          ▲
                                                                        │          │
 (Mớ hỗn độn n*(n-1)/2)           (Dễ dính SPOF tại Bus)               [S3]       [S4]
                                                                (Loose Coupling, Scalable)
```

1. **Point-to-Point (Mạng lưới trực tiếp):** Mỗi service tự gọi trực tiếp tới các service khác.
   - *Hạn chế:* Số lượng liên kết tăng theo cấp số nhân $\frac{n(n-1)}{2}$. Rất khó kiểm soát luồng dữ liệu khi hệ thống scale lên hàng chục service.
2. **Hub-and-Spoke (Trục tập trung):** Mọi giao tiếp đều đi qua một bộ điều phối trung tâm (API Gateway hoặc Enterprise Service Bus - ESB).
   - *Hạn chế:* Điểm trung tâm dễ trở thành nút cổ chai (Bottleneck) và SPOF nếu không được thiết kế phân tán.
3. **Event-Driven Mesh (Hướng sự kiện qua Message Bus):** Các service giao tiếp phi tập trung thông qua việc phát và lắng nghe các sự kiện bất đồng bộ.
   - *Ưu điểm:* Khả năng mở rộng tuyệt vời, các thành phần không cần biết về sự tồn tại của nhau.

---

## 5. System Mapping (Bản đồ hóa Hệ thống)

Để tư duy hệ thống hiệu quả, kỹ sư bắt buộc phải có khả năng **vẽ bản đồ hệ thống (System Mapping)** nhằm đưa các trừu tượng phức tạp ra mặt phẳng quan sát trực quan.

### Mô hình Bản đồ hóa C4 Model (Cực kỳ phổ biến trong Software Architecture)

Mô hình C4 do Simon Brown khởi xướng ví như Google Maps cho phần mềm, gồm 4 mức thu phóng:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 1: SYSTEM CONTEXT DIAGRAM (Bối cảnh Hệ thống)                     │
│ Nhìn từ vệ tinh: Ai dùng hệ thống này? Nó tương tác với hệ thống nào?   │
│ [Users/Khách hàng] ──► [E-COMMERCE SYSTEM] ──► [Cổng Thanh toán Stripe] │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (Thu phóng vào hệ thống)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 2: CONTAINER DIAGRAM (Các Ứng dụng & Kho dữ liệu)                 │
│ Hệ thống gồm những process nào chạy độc lập? (App, DB, File Store...)    │
│ [Web SPA] ──► [API Gateway] ──► [Order API Container] ──► [Postgres DB] │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (Thu phóng vào 1 Container)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 3: COMPONENT DIAGRAM (Các Module nội bộ của 1 Container)          │
│ Bên trong Order API gồm có những Controller, Service, Repository nào?   │
│ [OrderController] ──► [OrderService] ──► [PaymentClient, OrderRepo]     │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (Thu phóng vào 1 Component)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 4: CODE DIAGRAM (Class / Entity Diagram)                          │
│ Chi tiết cấu trúc code: Các class, method, interface cài đặt cụ thể.    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Tổng kết & Bài học Thực chiến cho Kỹ sư Backend

1. **Cẩn trọng với Dependency ngầm:** Luôn phân tích cây phụ thuộc (Dependency Graph). Một sự cố mạng nhỏ ở một dịch vụ phân tích dữ liệu không quan trọng có thể kéo sập API thanh toán cốt lõi nếu hai bên có quan hệ phụ thuộc đồng bộ.
2. **Ứng dụng mẫu thiết kế Circuit Breaker:** Khi Service A phụ thuộc vào Service B qua mạng, bắt buộc phải bọc lời gọi bằng Circuit Breaker (ngắt mạch khi lỗi liên tiếp) và Timeout nghiêm ngặt để cắt đứt sự phụ thuộc khi hệ thống có sự cố.
3. **Luôn bắt đầu bằng việc vẽ Bản đồ Hệ thống (Map before Code):** Trước khi tối ưu hiệu năng hay refactor một hệ thống cũ, hãy vẽ sơ đồ bối cảnh và luồng dữ liệu (C4 Model Level 1 & 2). Bạn không thể tối ưu hóa thứ mà bạn không nhìn thấy cấu trúc toàn cảnh của nó!
