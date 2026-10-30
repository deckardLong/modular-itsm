# Kế hoạch thiết kế sơ đồ ITSM (phạm vi mức C)

> Tất cả sơ đồ được vẽ bằng **draw.io trên web** (app.diagrams.net).
> Mọi điểm cần "họp để chốt" đã được **chốt sẵn trong tài liệu này** (mục 2). Cả nhóm chỉ cần đọc và làm theo; nếu có tranh luận thì **M1 quyết định cuối cùng**.
> Cập nhật lần cuối: 30/10/2026

---

## 1. Ranh giới hệ thống (mức C: đầy đủ ITIL)

### Nằm trong hệ thống

| Nhóm | Thành phần |
|---|---|
| **Lõi** | Đăng nhập + phân quyền (RBAC), Incident, Service Request + Catalog (có phê duyệt), Knowledge Base, SLA cơ bản, thông báo email + trong app, Dashboard/báo cáo, Audit log, Asset dạng danh sách |
| **Mở rộng** | Problem + Known Error, Change + CAB, CMDB có quan hệ giữa các CI, SLA nâng cao (giờ làm việc, escalate), thông báo Telegram, AI Agent (cả hai hướng: phân loại ticket + chatbot RAG tra cứu KB) |

### Nằm ngoài hệ thống (không làm)

- Ứng dụng mobile (chỉ web responsive)
- Đa tổ chức (multi-tenant), đăng nhập SSO/LDAP
- Tích hợp hệ thống giám sát để tự tạo Incident
- Quản lý license phần mềm, mua sắm, tài chính
- Workflow designer kéo thả
- Nhận ticket qua email (chỉ gửi email đi)
- Đa ngôn ngữ, nhiều múi giờ

### Người dùng và hệ thống bên ngoài (xuất hiện trong sơ đồ Context)

**Người dùng (7 vai trò):** End User, Agent L1/L2/L3, Dept Manager, Change Manager, CAB Member, KB Editor, Admin.

**Hệ thống bên ngoài (3):**

| Hệ thống | Vai trò | Ghi chú |
|---|---|---|
| Máy chủ email (SMTP) | Nhận email từ hệ thống để gửi thông báo | Bắt buộc, chỉ gửi đi |
| Nhà cung cấp LLM/embedding (qua API) | AI Service gọi để phân loại ticket và trả lời chatbot | Cần vì làm cả hai hướng AI |
| Telegram Bot | Gửi thông báo đẩy | Tùy chọn, cắt đầu tiên nếu cần giảm |

### Các quyết định đã chốt

| Mục | Chốt |
|---|---|
| Bối cảnh giả định | Phòng IT của một trường đại học (sinh viên, giảng viên, nhân viên các phòng ban) |
| Quy mô | Một tổ chức, vài chục người dùng đồng thời, giao diện tiếng Việt, một múi giờ (Asia/Ho_Chi_Minh) |
| Kênh tạo ticket | Chỉ qua web portal |
| Đăng nhập | Tài khoản + mật khẩu (JWT); Admin tạo thủ công hoặc nhập CSV |
| AI | Làm cả hai hướng; gọi LLM qua API bên ngoài, không tự chạy model |
| Nơi chạy | Cloud (nhà cung cấp do M5 chọn trước khi vẽ Deployment) |
| Thông báo đẩy | Telegram (không dùng Zalo) |

---

## 2. Bộ khung chung đã chốt (không cần họp)

Mọi sơ đồ phải dùng đúng các quy ước dưới đây. **M1 giữ bản chính thức**; muốn đổi thì báo M1 và cả nhóm trước khi sửa.

### 2.1. Bảng thuật ngữ

| Tên dùng trong sơ đồ | Nghĩa | Không dùng các từ này |
|---|---|---|
| **Ticket** | Tên chung cho Incident, Service Request, Problem, Change | case, sự vụ, phiếu |
| **Incident** | Sự cố làm gián đoạn dịch vụ | sự cố (viết tiếng Việt thay cho tên) |
| **Service Request** (gọi tắt **Request**) | Yêu cầu dịch vụ có sẵn trong danh mục (cấp tài khoản, cài phần mềm...) | yêu cầu |
| **Problem** | Nguyên nhân gốc của nhiều Incident lặp lại | vấn đề |
| **Known Error** | Problem đã biết nguyên nhân và có cách xử lý tạm | lỗi biết trước |
| **Change** | Thay đổi có kiểm soát đối với hạ tầng/dịch vụ | RFC |
| **CAB** | Hội đồng duyệt Change | ban duyệt |
| **Assignment Group** | Nhóm xử lý ticket | team, queue |
| **Agent** | Người xử lý ticket (cấp L1/L2/L3) | nhân viên IT, support |
| **End User** | Người gửi ticket | khách hàng, user |
| **CI (Config Item)** | Một tài sản/thành phần trong CMDB (máy chủ, laptop, ứng dụng, dịch vụ) | asset (chỉ dùng "Asset" cho bản danh sách đơn giản) |
| **CMDB** | Cơ sở dữ liệu các CI và quan hệ giữa chúng | |
| **SLA Policy / SLA Timer** | Quy định hạn xử lý / bộ đếm thời gian của một ticket | |
| **Knowledge Article** (gọi tắt **bài KB**) | Bài hướng dẫn trong Knowledge Base | |
| **Catalog Item** | Một dịch vụ trong Service Catalog | |

### 2.2. Quy ước tham chiếu ticket chung

SLA, Notification, Audit, Attachment, Comment áp dụng cho nhiều loại ticket, nên **không tạo khóa ngoại riêng tới từng bảng ticket**. Thay vào đó mỗi bảng đó có hai cột:

- `ticket_type`: một trong `incident`, `request`, `problem`, `change`
- `ticket_id`: mã của ticket đó

Nhờ vậy M3 (Notification), M4 (SLA) và M1 (Audit) không phụ thuộc cứng vào bảng của M2.

### 2.3. Danh sách vai trò

| Mã vai trò | Tên | Quyền chính |
|---|---|---|
| `end_user` | End User | Tạo ticket, xem ticket của mình, tra cứu KB, dùng chatbot |
| `agent_l1` | Agent L1 | Tiếp nhận, phân loại, xử lý sự cố cơ bản |
| `agent_l2` | Agent L2 | Xử lý sâu hơn, xử lý Problem |
| `agent_l3` | Agent L3 | Chuyên gia, xử lý nguyên nhân gốc |
| `dept_manager` | Dept Manager | Duyệt Request của nhân viên phòng mình |
| `change_manager` | Change Manager | Lập, theo dõi, lên lịch Change |
| `cab_member` | CAB Member | Bỏ phiếu duyệt Change |
| `kb_editor` | KB Editor | Soạn, duyệt, xuất bản bài KB |
| `admin` | Admin | Quản lý người dùng, nhóm, danh mục, SLA, xem báo cáo |

### 2.4. Trạng thái chuẩn của từng loại

| Loại | Các trạng thái (dùng đúng tên này) |
|---|---|
| **Incident** | New → Assigned → In Progress ⇄ On Hold → Resolved → Closed; Resolved → Reopened → Assigned |
| **Request** | Submitted → Pending Approval → Approved / Rejected; Approved → In Fulfillment → Completed; Cancelled (từ Submitted hoặc Pending Approval) |
| **Problem** | New → Under Investigation → Known Error → Resolved → Closed |
| **Change** | Draft → Submitted → CAB Review → Approved / Rejected; Approved → Scheduled → Implementing → Review → Closed |
| **Bài KB** | Draft → In Review → Published → Archived |
| **SLA Timer** | Running ⇄ Paused → Met / Breached |

**Quy tắc liên quan:** Incident vào `On Hold` thì SLA Timer chuyển `Paused`; ra khỏi `On Hold` thì `Running` lại. Incident `Resolved` thì SLA Timer chuyển `Met` (hoặc `Breached` nếu đã quá hạn).

### 2.5. Ma trận ưu tiên và SLA mặc định

**Ưu tiên = Mức tác động (Impact) × Mức khẩn (Urgency):**

| Impact \ Urgency | High | Medium | Low |
|---|---|---|---|
| **High** | P1 | P2 | P3 |
| **Medium** | P2 | P3 | P4 |
| **Low** | P3 | P4 | P4 |

**SLA mặc định** (tính theo giờ làm việc thứ Hai đến thứ Sáu, 8:00 đến 17:00; P1 tính 24/7):

| Ưu tiên | Phản hồi trong | Giải quyết trong |
|---|---|---|
| P1 | 15 phút | 4 giờ |
| P2 | 30 phút | 8 giờ |
| P3 | 2 giờ | 3 ngày làm việc |
| P4 | 4 giờ | 5 ngày làm việc |

### 2.6. Danh sách sự kiện chuẩn

Dùng khi vẽ Sequence Diagram:

| Sự kiện | Bên phát | Bên lắng nghe |
|---|---|---|
| `ticket.created` | M2, M3, M4 | M4 (bắt đầu SLA), M3 (thông báo), M1 (audit), M6/ai-service (phân loại ticket) |
| `ticket.assigned` | M2 | M3 (thông báo) |
| `ticket.status_changed` | M2, M3, M4 | M4 (tạm dừng/tiếp tục SLA), M3, M1 |
| `ticket.resolved` / `ticket.closed` | M2 | M4 (dừng SLA), M3 |
| `sla.warning` / `sla.breached` | M4 | M3 (thông báo), M2 (escalate) |
| `change.approved` / `change.rejected` | M4 | M3 |
| `request.approval_needed` | M3 | M3 (thông báo cho Dept Manager) |

### 2.7. Quy ước trình bày chung

| Yếu tố | Quy ước |
|---|---|
| **Màu khối lõi** | Xanh dương (mặc định của draw.io) |
| **Khối mở rộng** | Viền nét đứt, nền xám nhạt |
| **Hệ thống bên ngoài** | Nền xám đậm |
| **Người dùng** | Hình người (thư viện sẵn của draw.io) |
| **Database / lưu trữ** | Hình trụ |
| **Mũi tên liền** | Gọi đồng bộ (HTTP, gọi hàm) |
| **Mũi tên đứt** | Bất đồng bộ (sự kiện, hàng đợi, Celery) |
| **Nhãn mũi tên** | Bắt buộc, ghi rõ gọi gì hoặc dữ liệu gì |
| **Khung tiêu đề mỗi tab** | Tên sơ đồ, người vẽ, ngày cập nhật |
| **Chú giải** | Mỗi tab có một chú giải theo bảng này |
| **Cỡ tối đa** | Khoảng 15 khối mỗi sơ đồ; nhiều hơn thì tách |

---

## 3. Danh sách sơ đồ (30 sơ đồ)

Cột **Nhóm**: *Lõi* = vẽ ngay từ sprint 1; *Mở rộng* = vẽ sau mốc quyết định cuối sprint 2 (mục 6).

| # | Loại sơ đồ | Mục đích | Phụ trách | Nhóm |
|---|---|---|---|---|
| 1 | **Use Case** | Cho biết ai dùng hệ thống để làm gì; là nguồn yêu cầu cho thiết kế | M1 | Lõi |
| 2 | **Context (C4 mức 1)** | Cho thấy hệ thống nằm ở đâu so với người dùng và hệ thống ngoài | M1 | Lõi |
| 3 | **Container (C4 mức 2)** | Liệt kê các khối chạy được (web, API, worker, AI service, database, cache, storage) và cách nối với nhau | M1 | Lõi (ai-service vẽ nét đứt) |
| 4 | **Component backend (C4 mức 3)** | Cho thấy backend chia module nào và module nào gọi module nào, để phát hiện phụ thuộc vòng | M1 | Lõi (module mở rộng nét đứt) |
| 5a | **ERD cụm lõi người dùng/tổ chức** | Bảng User, Department, Role, AssignmentGroup, AuditLog | M1 | Lõi |
| 5b | **ERD cụm Incident** | Bảng Incident, Comment, Worklog, liên kết Asset/CI | M2 | Lõi |
| 5c | **ERD cụm Catalog/Request/Knowledge/Notification** | Bảng danh mục dịch vụ, Request, bước phê duyệt, bài KB, thông báo | M3 | Lõi |
| 5d | **ERD cụm SLA** | Bảng SLAPolicy, BusinessCalendar, SLATimer | M4 | Lõi |
| 5e | **ERD cụm Problem + Known Error** | Bảng Problem, KnownError, liên kết Incident-Problem | M2 | Mở rộng |
| 5f | **ERD cụm Change** | Bảng Change, ChangeApproval (CAB), đánh giá rủi ro, liên kết CI | M4 | Mở rộng |
| 5g | **ERD cụm CMDB** | Bảng ConfigItem, CIType, CIRelationship | M1 | Lõi (danh sách) + Mở rộng (quan hệ giữa CI) |
| 5h | **ERD tổng** | Chỉ tên bảng và khóa ngoại, để kiểm tra mâu thuẫn giữa các cụm | M1 | Lõi, cập nhật khi thêm cụm mở rộng |
| 6a | **State: Incident** | Trạng thái, ai chuyển và khi nào; nguồn viết test nghiệp vụ | M2 | Lõi |
| 6b | **State: Service Request** | Vòng đời Request kèm nhánh phê duyệt | M3 | Lõi |
| 6c | **State: Bài KB** | Vòng đời bài KB | M3 | Lõi |
| 6d | **State: SLA Timer** | Chạy, tạm dừng, đạt, vi phạm | M4 | Lõi |
| 6e | **State: Problem** | Vòng đời Problem, bước tạo Known Error | M2 | Mở rộng |
| 6f | **State: Change** | Vòng đời Change qua đánh giá rủi ro và CAB | M4 | Mở rộng |
| 7a | **Sequence: đăng nhập** | Luồng cấp token và kiểm tra quyền | M1 | Lõi |
| 7b | **Sequence: tạo và xử lý Incident** | Tạo → gán nhóm → SLA bắt đầu đếm → thông báo | M2 | Lõi |
| 7c | **Sequence: Request có phê duyệt** | Gửi → Dept Manager duyệt → thực hiện → đóng | M3 | Lõi |
| 7d | **Sequence: cảnh báo vi phạm SLA** | Celery Beat quét → cảnh báo → thông báo | M4 | Lõi |
| 7e | **Sequence: duyệt Change qua CAB** | Gửi Change → đánh giá rủi ro → CAB bỏ phiếu → lên lịch | M4 | Mở rộng |
| 7f | **Sequence: chatbot RAG** | Hỏi → truy xuất KB → gọi LLM → trả lời kèm nguồn | M6 | Mở rộng |
| 7g | **Sequence: tự phân loại ticket** | Ticket mới → ai-service gợi ý danh mục và ưu tiên → Agent xác nhận | M6 | Mở rộng |
| 8 | **Deployment Diagram** | Cho thấy hệ thống chạy ở đâu trên cloud (máy chủ/dịch vụ chạy container, mạng, database, lưu trữ file, DNS/HTTPS, kết nối ra LLM API) | M5 | Lõi (bổ sung ai-service, storage khi có) |
| 9 | **Pipeline CI/CD** | Luồng từ commit đến test, build, deploy staging/production và rollback | M5 | Lõi |
| 10a | **Site Map** | Các màn hình, ai được truy cập, điều hướng giữa các trang | M6 | Lõi |
| 10b | **Wireframe màn hình lõi** | Đăng nhập, danh sách/chi tiết ticket, Catalog, KB, Dashboard | M6 | Lõi |
| 10c | **Wireframe màn hình mở rộng** | Problem, Change/CAB, CMDB, giao diện chat | M6 | Mở rộng |

**Tổng: 30 sơ đồ**, trong đó 22 lõi và 8 mở rộng (5e, 5f, 6e, 6f, 7e, 7f, 7g, 10c).

---

## 4. Chia file và tab trên draw.io

**Mỗi sơ đồ một tab, mỗi thành viên một file: 6 file, 30 tab.**

| File | Chủ | Tab | Số tab |
|---|---|---|---|
| `M1-architecture.drawio` | M1 | 01-UseCase · 02-Context · 03-Container · 04-Component · 5a-ERD-Core · 5g-ERD-CMDB · 5h-ERD-Tong · 7a-Seq-DangNhap | 8 |
| `M2-incident-problem.drawio` | M2 | 5b-ERD-Incident · 6a-State-Incident · 7b-Seq-TaoIncident · 5e-ERD-Problem (MR) · 6e-State-Problem (MR) | 5 |
| `M3-request-knowledge.drawio` | M3 | 5c-ERD-Catalog-Request-KB-Notif · 6b-State-Request · 6c-State-KB · 7c-Seq-Request-PheDuyet | 4 |
| `M4-sla-change.drawio` | M4 | 5d-ERD-SLA · 6d-State-SLA · 7d-Seq-CanhBaoSLA · 5f-ERD-Change (MR) · 6f-State-Change (MR) · 7e-Seq-CAB (MR) | 6 |
| `M5-deployment.drawio` | M5 | 08-Deployment-Cloud · 09-Pipeline-CICD | 2 |
| `M6-frontend-ai.drawio` | M6 | 10a-SiteMap · 10b-Wireframe-Loi · 10c-Wireframe-MoRong (MR) · 7f-Seq-Chatbot-RAG (MR) · 7g-Seq-PhanLoaiTicket (MR) | 5 |

**Quy ước tên tab:** mã + loại + tên ngắn; thêm **(MR)** cho sơ đồ mở rộng. Tạo sẵn tab trống có đuôi (MR) từ đầu để nhìn thanh tab biết cái nào đang chờ. Cả 6 file đặt chung một thư mục Google Drive hoặc OneDrive của nhóm.

---

## 5. Phụ thuộc giữa các thành viên

**Có phụ thuộc, nhưng chỉ ở một số điểm.** Phần còn lại vẽ song song được.

| Sơ đồ | Phụ thuộc vào | Lý do |
|---|---|---|
| ERD các cụm (M2, M3, M4) | ERD 5a và 5g của M1 (User, Department, Role, AssignmentGroup, ConfigItem) | Cần để nối khóa ngoại |
| ERD tổng 5h (M1) | Tất cả ERD cụm | Là bản ghép |
| State SLA Timer 6d (M4) | State Incident 6a (M2), đặc biệt `On Hold` | Tạm dừng và tiếp tục SLA (đã chốt ở mục 2.4) |
| State Change 6f (M4) | State Problem 6e (M2) | Problem có thể dẫn tới Change |
| ERD Change 5f (M4) | ERD CMDB 5g (M1) | Change liên kết với CI |
| Component backend (M1) | M2, M3, M4 báo module mình gọi những module nào | Để vẽ đúng mũi tên |
| Sequence 7b (M2) | M3 (thông báo), M4 (SLA) | Luồng đi qua cả ba module |
| Sequence 7d (M4) | M3 (thông báo), M5 (Celery) | Đi qua thông báo và hạ tầng |
| Sequence 7e (M4) | M3 (thông báo), M1 (vai trò CAB) | Cần kênh thông báo và quyền |
| Deployment (M5) | Container (M1), Celery của M4, cách build của M6 | Phải khớp các khối đã định nghĩa |
| Wireframe (M6) | Use Case (M1), State Diagram và danh sách endpoint của M2, M3, M4 | Cần biết nút hành động theo trạng thái và các trường của form |
| Sequence 7f, 7g (M6) | Cấu trúc KB của M3, Container (M1) | RAG cần dữ liệu KB; ai-service là khối riêng |

**Làm song song được:** ERD cụm của M2 và M3, Pipeline CI/CD của M5, Site Map cấp cao của M6 (sau khi có Use Case).

---

## 6. Thứ tự thực hiện và mốc quyết định

### Giai đoạn lõi (Sprint 1)

| Đợt | Khi nào | Việc | Ai |
|---|---|---|---|
| **0** | Ngày 1 (sáng) | M1 công bố bộ khung ở mục 2; mọi người đọc và xác nhận trong ngày (không cần họp quyết định, chỉ họp ngắn nếu có thắc mắc). M5 chọn nhà cung cấp cloud | M1, M5 |
| **1** | Ngày 1 (chiều) đến sáng ngày 2 | Use Case, Context, Container bản sạch, ERD 5a, ERD 5g (danh sách CI) | M1 |
| **2** | Ngày 2 đến 3 (song song) | ERD 5b, 5c, 5d; State 6a, 6b, 6c, 6d; Deployment; Pipeline CI/CD; Site Map cấp cao | M2, M3, M4, M5, M6 |
| **3** | Ngày 3 đến 4 | Review chéo (mục 7), ghép ERD tổng 5h, vẽ Component backend | M1 chủ trì |
| **4** | Ngày 4 đến 5 | Sequence 7a, 7b, 7c, 7d; Wireframe lõi 10b | Chủ luồng, M6 |

### Mốc quyết định cuối Sprint 2 (áp dụng theo quy tắc, không cần tranh luận)

M1 kiểm tra các điều kiện sau và áp dụng bảng cắt giảm ngay:

| Tình hình cuối sprint 2 | Quyết định |
|---|---|
| Phần lõi (Auth, Incident, Request, SLA cơ bản) **đã qua Handoff Gate** và có **không quá 1 task trễ** | Giữ nguyên mức C, vẽ và làm toàn bộ phần mở rộng |
| Phần lõi đã qua Gate nhưng có **2 đến 3 task trễ** | Cắt bước 1 đến 4 trong bảng dưới |
| Phần lõi **chưa qua Gate** hoặc có **từ 4 task trễ** trở lên | Cắt cả bước 1 đến 6 (về mức A), dồn sức hoàn thiện phần lõi |

**Bảng cắt giảm (cắt theo thứ tự này, không đảo):**

| Bước | Bỏ gì | Sơ đồ không cần vẽ |
|---|---|---|
| 1 | Thông báo Telegram | Bỏ Telegram Bot khỏi Context |
| 2 | **Phân loại ticket tự động** (giữ chatbot vì dễ demo hơn) | Bỏ 7g |
| 3 | SLA nâng cao (escalate tự động, lịch phức tạp) | Rút gọn ERD 5d (bỏ BusinessCalendar nếu cần), State 6d |
| 4 | CMDB có quan hệ giữa các CI (còn lại Asset dạng danh sách) | Bỏ phần quan hệ trong ERD 5g |
| 5 | Change + CAB (từ C xuống B) | Bỏ 5f, 6f, 7e; bỏ màn hình Change trong 10c |
| 6 | Problem + Chatbot (từ B xuống A) | Bỏ 5e, 6e, 7f; bỏ màn hình Problem và giao diện chat trong 10c |

**Phần lõi không bao giờ cắt:** Auth/RBAC, Incident, Service Request + Catalog, Knowledge Base, SLA cơ bản, Dashboard.

### Giai đoạn mở rộng (đầu Sprint 3 trở đi)

Chỉ vẽ cho các module còn lại sau khi áp dụng bảng cắt:

| Thứ tự | Việc | Ai |
|---|---|---|
| 1 | ERD 5e, State 6e (Problem) | M2 |
| 2 | ERD CMDB quan hệ (bổ sung 5g); ERD 5f, State 6f (Change); Sequence 7e | M1, M4 |
| 3 | Cập nhật ERD tổng 5h, Component backend, Container (khối mở rộng) | M1 |
| 4 | Wireframe mở rộng 10c | M6 |
| 5 | Sequence 7f, 7g (AI); cập nhật Deployment (ai-service, storage) | M6, M5 |

---

## 7. Phân công và review chéo

### Phân công theo thành viên

| Thành viên | Sơ đồ lõi (vẽ ngay) | Sơ đồ mở rộng (vẽ sau mốc quyết định) |
|---|---|---|
| **M1** (Backend Lead) | Use Case, Context, Container, Component, 5a, 5g (danh sách CI), 5h, 7a | Bổ sung 5g (quan hệ CI), cập nhật 5h và Component |
| **M2** | 5b, 6a, 7b | 5e, 6e |
| **M3** | 5c, 6b, 6c, 7c | Hỗ trợ M6 về cấu trúc KB cho RAG |
| **M4** | 5d, 6d, 7d | 5f, 6f, 7e |
| **M5** (DevOps) | Deployment, Pipeline CI/CD | Bổ sung ai-service, storage, monitoring vào Deployment |
| **M6** (Frontend + AI) | 10a, 10b | 10c, 7f, 7g |

### Ai review sơ đồ của ai

| Sơ đồ của | Người review chính | Xem gì |
|---|---|---|
| M1 (Context, Container, Component) | M5 và M6 | Có khớp triển khai và giao diện không |
| M2 | M4 | Khớp SLA/Change; trạng thái liên quan |
| M3 | M2 | Khớp sự kiện ticket |
| M4 | M2 | Khớp Incident/Problem |
| M5 | M1 | Khớp Container diagram |
| M6 | M2, M3, M4 | Đủ API cho từng màn hình |

---

## 8. Tiêu chí cho mỗi sơ đồ

- Có khung tiêu đề (tên, người vẽ, ngày cập nhật) và chú giải theo mục 2.7
- Mọi mũi tên đều có nhãn
- Tên khối, bảng, trạng thái, vai trò khớp **đúng** mục 2 (thuật ngữ, trạng thái, vai trò, sự kiện)
- Không quá khoảng 15 khối; nhiều hơn thì tách thành sơ đồ con
- Đã được người review chính (mục 7) xem
- Sơ đồ mở rộng chưa chắc làm thì ghi (MR) trên tên tab và vẽ nét đứt trên các sơ đồ lõi (Container, Component)
