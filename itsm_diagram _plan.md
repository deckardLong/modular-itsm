# Kế hoạch thiết kế sơ đồ ITSM (phạm vi mức C)

> Tất cả sơ đồ được vẽ bằng **draw.io trên web** (app.diagrams.net).
> Tài liệu nêu: ranh giới hệ thống, loại sơ đồ và mục đích, người phụ trách, phụ thuộc giữa các thành viên, và cách cắt giảm nếu cần.
> Cập nhật lần cuối: 30/10/2026

---

## 1. Ranh giới hệ thống (mức C: đầy đủ ITIL)

Nhóm thử sức mức C trước; nếu không ổn thì cắt giảm theo mục 6.

### Nằm trong hệ thống

| Nhóm | Thành phần |
|---|---|
| **Lõi** | Đăng nhập + phân quyền (RBAC), Incident, Service Request + Catalog (có phê duyệt), Knowledge Base, SLA cơ bản, thông báo email + trong app, Dashboard/báo cáo, Audit log |
| **Mở rộng** | Problem + Known Error, Change + CAB, CMDB có quan hệ giữa các CI, SLA nâng cao (giờ làm việc, escalate), thông báo Telegram, AI Agent (cả hai hướng: phân loại ticket + chatbot RAG tra cứu KB) |

### Nằm ngoài hệ thống (không làm)

- Ứng dụng mobile (chỉ web responsive)
- Đa tổ chức (multi-tenant), đăng nhập SSO/LDAP
- Tích hợp hệ thống giám sát để tự tạo Incident
- Quản lý license phần mềm, mua sắm, tài chính
- Workflow designer kéo thả
- Nhận ticket qua email (chỉ gửi email đi)

### Hệ thống bên ngoài xuất hiện trong sơ đồ Context

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
| Quy mô | Một tổ chức, vài chục người dùng đồng thời, giao diện tiếng Việt, một múi giờ và một lịch làm việc SLA |
| Kênh tạo ticket | Chỉ qua web portal |
| Đăng nhập | Tài khoản + mật khẩu (JWT); Admin tạo thủ công hoặc nhập CSV |
| AI | **Làm cả hai hướng**: tự phân loại ticket và chatbot RAG tra cứu KB; gọi LLM qua API bên ngoài, không tự chạy model |
| Nơi chạy | **Cloud** (nhà cung cấp cụ thể do nhóm chọn, xem sơ đồ Deployment) |
| Thông báo đẩy | Telegram (không dùng Zalo) |

---

## 2. Danh sách sơ đồ

Cột **Nhóm**: *Lõi* = vẽ ngay từ sprint 1; *Mở rộng* = vẽ sau mốc quyết định cuối sprint 2 (mục 5), để nếu phải cắt giảm thì không mất công vẽ.

| # | Loại sơ đồ | Mục đích | Phụ trách | Nhóm |
|---|---|---|---|---|
| 1 | **Use Case** | Cho biết ai dùng hệ thống để làm gì; là nguồn yêu cầu cho toàn bộ thiết kế | M1 | Lõi (phần mở rộng bổ sung sau) |
| 2 | **Context (C4 mức 1)** | Cho thấy hệ thống nằm ở đâu so với người dùng và hệ thống bên ngoài | M1 | Lõi |
| 3 | **Container (C4 mức 2)** | Liệt kê các khối chạy được (web, API, worker, AI service, database, cache, storage) và cách nối với nhau | M1 | Lõi (ai-service ghi là khối mở rộng) |
| 4 | **Component backend (C4 mức 3)** | Cho thấy backend chia module nào và module nào gọi module nào, để phát hiện phụ thuộc vòng | M1 | Lõi (module mở rộng vẽ nét đứt, hoàn thiện sau) |
| 5a | **ERD cụm lõi người dùng/tổ chức** | Bảng User, Department, Role, AssignmentGroup, AuditLog | M1 | Lõi |
| 5b | **ERD cụm Incident** | Bảng Incident, Comment, Worklog, liên kết với Asset/CI | M2 | Lõi |
| 5c | **ERD cụm Catalog/Request/Knowledge/Notification** | Bảng danh mục dịch vụ, Request, bước phê duyệt, bài KB, thông báo | M3 | Lõi |
| 5d | **ERD cụm SLA** | Bảng SLAPolicy, BusinessCalendar, SLATimer | M4 | Lõi |
| 5e | **ERD cụm Problem + Known Error** | Bảng Problem, KnownError, liên kết Incident-Problem | M2 | Mở rộng |
| 5f | **ERD cụm Change** | Bảng Change, ChangeApproval (CAB), đánh giá rủi ro, liên kết CI | M4 | Mở rộng |
| 5g | **ERD cụm CMDB** | Bảng ConfigItem, CIType, CIRelationship | M1 | Lõi (bản danh sách) + Mở rộng (quan hệ giữa CI) |
| 5h | **ERD tổng** | Chỉ có tên bảng và khóa ngoại, để thấy toàn cảnh và kiểm tra mâu thuẫn giữa các cụm | M1 | Lõi, cập nhật khi thêm cụm mở rộng |
| 6a | **State: Incident** | Trạng thái, ai được chuyển và khi nào; nguồn để viết test nghiệp vụ | M2 | Lõi |
| 6b | **State: Service Request** | Vòng đời Request kèm nhánh phê duyệt | M3 | Lõi |
| 6c | **State: Knowledge Article** | Vòng đời bài KB (soạn, duyệt, xuất bản, lưu trữ) | M3 | Lõi |
| 6d | **State: SLA Timer** | Chạy, tạm dừng, vi phạm, đạt | M4 | Lõi |
| 6e | **State: Problem** | Vòng đời Problem, bước tạo Known Error | M2 | Mở rộng |
| 6f | **State: Change** | Vòng đời Change qua đánh giá rủi ro và CAB | M4 | Mở rộng |
| 7a | **Sequence: đăng nhập** | Luồng cấp token và kiểm tra quyền | M1 | Lõi |
| 7b | **Sequence: tạo và xử lý Incident** | Tạo → gán nhóm → SLA bắt đầu đếm → thông báo | M2 | Lõi |
| 7c | **Sequence: Service Request có phê duyệt** | Gửi → Dept Manager duyệt → thực hiện → đóng | M3 | Lõi |
| 7d | **Sequence: cảnh báo vi phạm SLA** | Celery Beat quét → cảnh báo → thông báo | M4 | Lõi |
| 7e | **Sequence: duyệt Change qua CAB** | Gửi Change → đánh giá rủi ro → CAB bỏ phiếu → lên lịch | M4 | Mở rộng |
| 7f | **Sequence: chatbot RAG và phân loại ticket** | Hỏi → truy xuất KB → trả lời kèm nguồn; ticket mới → tự gợi ý danh mục/ưu tiên | M6 | Mở rộng |
| 8 | **Deployment Diagram** | Cho thấy hệ thống chạy ở đâu trên cloud (máy chủ/dịch vụ chạy container, mạng và cổng, database, lưu trữ file, DNS/HTTPS, kết nối ra LLM API) | M5 | Lõi (bổ sung ai-service, MinIO/object storage khi có) |
| 9 | **Pipeline CI/CD** | Luồng từ commit đến test, build, deploy staging/production và rollback | M5 | Lõi |
| 10a | **Site Map** | Các màn hình, ai được truy cập, điều hướng giữa các trang | M6 | Lõi (trang mở rộng bổ sung sau) |
| 10b | **Wireframe màn hình lõi** | Đăng nhập, danh sách/chi tiết ticket, Catalog, KB, Dashboard | M6 | Lõi |
| 10c | **Wireframe màn hình mở rộng** | Problem, Change/CAB, CMDB, giao diện chat | M6 | Mở rộng |

---

## 3. Phân công theo thành viên

| Thành viên | Sơ đồ lõi (vẽ ngay) | Sơ đồ mở rộng (vẽ sau mốc quyết định) |
|---|---|---|
| **M1** (Backend Lead) | Use Case, Context, Container, Component backend, ERD 5a, ERD 5g (danh sách CI), ERD tổng 5h, Sequence 7a | Bổ sung ERD 5g (quan hệ CI), cập nhật ERD tổng và Component |
| **M2** | ERD 5b, State 6a, Sequence 7b | ERD 5e, State 6e |
| **M3** | ERD 5c, State 6b, State 6c, Sequence 7c | Hỗ trợ M6 về cấu trúc KB cho RAG |
| **M4** | ERD 5d, State 6d, Sequence 7d | ERD 5f, State 6f, Sequence 7e |
| **M5** (DevOps) | Deployment, Pipeline CI/CD | Bổ sung ai-service, MinIO, monitoring vào Deployment |
| **M6** (Frontend + AI) | Site Map, Wireframe lõi 10b | Wireframe mở rộng 10c, Sequence 7f |

---

## 4. Phụ thuộc giữa các thành viên

**Có phụ thuộc, nhưng chỉ ở một số điểm.** Phần còn lại vẽ song song được.

| Sơ đồ | Phụ thuộc vào | Lý do |
|---|---|---|
| Context, Container (M1) | Họp cả nhóm ngày đầu | Cần chốt ranh giới hệ thống |
| ERD các cụm (M2, M3, M4) | ERD 5a và 5g của M1 (User, Department, Role, AssignmentGroup, ConfigItem) | Cần để nối khóa ngoại |
| ERD tổng 5h (M1) | Tất cả ERD cụm | Là bản ghép |
| State SLA Timer 6d (M4) | State Incident 6a (M2), đặc biệt `On Hold` | Tạm dừng và tiếp tục SLA phụ thuộc trạng thái này |
| State Change 6f (M4) | State Problem 6e (M2) | Problem có thể dẫn tới Change |
| ERD Change 5f (M4) | ERD CMDB 5g (M1) | Change liên kết với CI |
| Component backend (M1) | M2, M3, M4 khai báo module mình gọi những module nào | Để vẽ đúng mũi tên |
| Sequence 7b (M2) | M3 (thông báo), M4 (SLA) | Luồng đi qua cả ba module |
| Sequence 7d (M4) | M3 (thông báo), M5 (Celery) | Đi qua thông báo và hạ tầng |
| Sequence 7e (M4) | M3 (thông báo), M1 (vai trò CAB) | Cần kênh thông báo và quyền |
| Deployment (M5) | Container (M1), Celery của M4, cách build của M6 | Phải khớp các khối đã định nghĩa |
| Wireframe (M6) | Use Case (M1), State Diagram và danh sách endpoint của M2, M3, M4 | Cần biết nút hành động theo trạng thái và các trường của form |
| Sequence 7f (M6) | Cấu trúc KB của M3, Container (M1) | RAG cần dữ liệu KB; ai-service là một khối riêng |

**Làm song song được:** ERD cụm của M2 và M3, Pipeline CI/CD của M5, Site Map cấp cao của M6 (sau khi có Use Case).

### Điểm cần chốt chung trong buổi họp đầu

1. **Bảng thuật ngữ**: một từ cho mỗi khái niệm (ticket, Incident, Request, Problem, Change, Assignment Group).
2. **Cách tham chiếu ticket chung** cho SLA, Notification, Audit: dùng cặp (loại ticket, id) thay vì khóa ngoại riêng tới từng bảng.
3. **Danh sách sự kiện chuẩn**: `ticket.created`, `ticket.assigned`, `ticket.status_changed`, `ticket.resolved`, `sla.warning`, `sla.breached`, `change.approved`, `request.approval_needed`.
4. **Danh sách vai trò**: End User, Agent L1/L2/L3, Dept Manager, Change Manager, CAB Member, KB Editor, Admin.
5. **Danh sách sơ đồ lõi và mở rộng** theo mục 2, để cả nhóm biết cái nào vẽ ngay, cái nào chờ.

---

## 5. Thứ tự thực hiện

### Giai đoạn lõi (Sprint 1)

| Đợt | Khi nào | Việc | Ai |
|---|---|---|---|
| **0** | Ngày 1 (sáng) | Họp chốt Context, Container sơ bộ và 5 điểm chung ở mục 4 | Cả nhóm |
| **1** | Ngày 1 (chiều) đến sáng ngày 2 | Use Case, Context, Container bản sạch, ERD 5a, ERD 5g (danh sách CI) | M1 |
| **2** | Ngày 2 đến 3 (song song) | ERD 5b, 5c, 5d; State 6a, 6b, 6c, 6d; Deployment; Pipeline CI/CD; Site Map cấp cao | M2, M3, M4, M5, M6 |
| **3** | Ngày 3 đến 4 | Review chéo, ghép ERD tổng 5h, vẽ Component backend | M1 chủ trì |
| **4** | Ngày 4 đến 5 | Sequence 7a, 7b, 7c, 7d; Wireframe lõi 10b | Chủ luồng, M6 |

### Mốc quyết định (cuối Sprint 2)

Họp cả nhóm để quyết định có làm tiếp các module mở rộng hay cắt giảm. Xét theo:

- Phần lõi (Auth, Incident, Request, SLA cơ bản) đã **qua Handoff Gate** chưa?
- Có từ 2 task trở lên trễ hạn bàn giao trong sprint 1 hoặc 2 không?
- M6 có theo kịp giao diện của các module đã xong không?
- Độ phủ test và CI có ổn định không?

**Nếu phần lõi chưa xong thì không bắt đầu module mở rộng**, dù còn thời gian trên lịch.

### Giai đoạn mở rộng (đầu Sprint 3 trở đi)

Chỉ vẽ cho các module nhóm quyết định giữ:

| Thứ tự | Việc | Ai |
|---|---|---|
| 1 | ERD 5e, State 6e (Problem) | M2 |
| 2 | ERD CMDB quan hệ (5g bổ sung); ERD 5f, State 6f (Change); Sequence 7e | M1, M4 |
| 3 | Cập nhật ERD tổng 5h, Component backend, Container (khối mở rộng) | M1 |
| 4 | Wireframe mở rộng 10c | M6 |
| 5 | Sequence 7f (AI), cập nhật Deployment (ai-service, MinIO) | M6, M5 |

---

## 6. Cắt giảm nếu mức C không ổn

Cắt từ phần ít ảnh hưởng luồng demo nhất đến nhiều nhất:

| Thứ tự cắt | Bỏ gì | Sơ đồ không cần vẽ |
|---|---|---|
| 1 | Thông báo Telegram | Bỏ Telegram Bot khỏi Context |
| 2 | Một trong hai hướng AI | Bỏ phần tương ứng trong 7f; có thể bỏ luôn 7f nếu bỏ cả hai |
| 3 | SLA nâng cao (giờ làm việc phức tạp, escalate tự động) | Rút gọn ERD 5d (bỏ BusinessCalendar) và State 6d |
| 4 | CMDB có quan hệ giữa các CI (còn lại danh sách Asset) | Bỏ phần quan hệ trong ERD 5g |
| 5 | Change + CAB (từ C xuống B) | Bỏ ERD 5f, State 6f, Sequence 7e, màn hình Change trong 10c |
| 6 | Problem (từ B xuống A) | Bỏ ERD 5e, State 6e, màn hình Problem trong 10c |

**Phần lõi không cắt:** Auth/RBAC, Incident, Service Request + Catalog, Knowledge Base, SLA cơ bản, Dashboard.

---

## 7. Tiêu chí cho mỗi sơ đồ

- Có tiêu đề, ngày cập nhật, người vẽ
- Có chú giải màu và hình dạng; mọi mũi tên đều có nhãn
- Tên khối, bảng, trạng thái khớp bảng thuật ngữ chung
- Không quá khoảng 15 khối; nếu nhiều hơn thì tách thành sơ đồ con
- Đã được ít nhất một người khác review (ví dụ M4 review State của M2, M5 và M6 review Container của M1)
- Sơ đồ mở rộng chưa chắc làm thì ghi rõ "mở rộng" hoặc vẽ nét đứt trên các sơ đồ lõi (Container, Component)