# Kế hoạch thực hiện đề tài ITSM (IT Service Management)

> Tài liệu phân công và lộ trình cho nhóm 6 thành viên.
> Cập nhật lần cuối: 30/10/2026

---

## 1. Mục tiêu và phạm vi

Xây dựng hệ thống ITSM theo hướng ITIL 4, kiến trúc **Modular Monolith** (Django REST Framework + PostgreSQL + Celery/Redis), giao diện web cho End User và Agent, kèm AI Agent hỗ trợ.

| Mức | Module |
|---|---|
| **MVP (bắt buộc)** | Auth/RBAC, Incident, Service Request + Catalog, SLA, Knowledge Base, Notification, Dashboard/Report |
| **Nên có** | Problem, Change (CAB), CMDB cơ bản, Audit log |
| **Mở rộng** | AI Agent: tự phân loại ticket, chatbot RAG tra cứu KB |

---

## 2. Phân công nhân sự

| Mã | Vai trò | Họ tên | Phụ trách chính |
|---|---|---|---|
| **M1** | Backend Lead | _(điền tên)_ | Core, Accounts (Auth, RBAC, Department, Assignment Group), Audit, CMDB |
| **M2** | Backend | _(điền tên)_ | Incident, Problem, Known Error, Reporting |
| **M3** | Backend | _(điền tên)_ | Service Catalog, Service Request (phê duyệt), Knowledge Base, Notification |
| **M4** | Backend | _(điền tên)_ | Change (CAB), SLA Engine (Celery/Beat) |
| **M5** | DevOps (CI/CD + Deployment) | _(điền tên)_ | Docker, CI/CD, môi trường dev/staging/prod, monitoring, hỗ trợ hạ tầng cho Celery/Redis/MinIO |
| **M6** | Frontend + AI Agent | _(điền tên)_ | Toàn bộ frontend, design system, AI Service (phân loại ticket, chatbot RAG) |

**Quy tắc sở hữu**

- Chỉ **M6** được sửa `frontend/` và `ai-service/` để giao diện đồng nhất.
- Chỉ **M5** được sửa `deploy/`, `.github/workflows/` (hoặc `.gitlab-ci.yml`), `docker-compose*.yml`.
- **M1 đến M4** chỉ làm backend, mỗi người chỉ sửa các thư mục `apps/` mình sở hữu.

### Thư mục sở hữu

| Thành viên | Thư mục |
|---|---|
| M1 | `backend/config/`, `apps/core/`, `apps/accounts/`, `apps/audit/`, `apps/cmdb/` |
| M2 | `apps/incidents/`, `apps/problems/`, `apps/reporting/` |
| M3 | `apps/service_catalog/`, `apps/service_requests/`, `apps/knowledge/`, `apps/notifications/` |
| M4 | `apps/changes/`, `apps/sla/` |
| M5 | `deploy/`, `docker-compose*.yml`, CI config, `docs/deployment/` |
| M6 | `frontend/`, `ai-service/` |

---

## 3. Cấu trúc thư mục dùng chung (monorepo)

Một repo duy nhất, mỗi thư mục gắn với một chủ sở hữu (M1 đến M6) để tránh xung đột. Mỗi model, serializer, view, service nằm trong một file riêng.

### 3.1. Cấp cao nhất

```
itsm-platform/
├── backend/                  # M1-M4 (theo từng app)
├── frontend/                 # M6
├── ai-service/               # M6
├── deploy/                   # M5
├── docs/                     # cả nhóm (xem 3.6)
├── scripts/                  # M5: script tiện ích (seed, backup, reset db)
├── .github/                  # M5
│   ├── workflows/
│   │   ├── ci-backend.yml
│   │   ├── ci-frontend.yml
│   │   ├── ci-ai-service.yml
│   │   ├── cd-staging.yml
│   │   └── cd-production.yml
│   ├── pull_request_template.md     # có sẵn mục Handoff Note
│   ├── ISSUE_TEMPLATE/
│   │   ├── feature.md
│   │   ├── bug.md
│   │   └── api-request.md           # backend dev yêu cầu màn hình cho M6
│   └── CODEOWNERS
├── docker-compose.yml        # M5 (chạy dev)
├── docker-compose.staging.yml
├── .env.example
├── .gitignore
├── .editorconfig
├── .pre-commit-config.yaml   # M5: lint/format trước khi commit
├── Makefile                  # lệnh tắt: make up, make test, make seed
└── README.md
```

### 3.2. Backend (Django)

```
backend/
├── manage.py
├── pytest.ini                         # M1
├── conftest.py                        # M1: fixture dùng chung
├── requirements/
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
├── config/                            # M1
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   ├── staging.py
│   │   └── prod.py
│   ├── urls.py
│   ├── asgi.py
│   ├── wsgi.py
│   └── celery.py
│
└── apps/
    ├── core/                          # M1: dùng chung toàn hệ thống
    │   ├── models/
    │   │   ├── timestamped.py
    │   │   ├── soft_delete.py
    │   │   └── attachment.py
    │   ├── permissions/
    │   ├── pagination.py
    │   ├── exceptions.py
    │   └── utils/
    ├── accounts/                      # M1
    ├── audit/                         # M1
    ├── cmdb/                          # M1
    ├── incidents/                     # M2
    ├── problems/                      # M2
    ├── reporting/                     # M2
    ├── service_catalog/               # M3
    ├── service_requests/              # M3
    ├── knowledge/                     # M3
    ├── notifications/                 # M3
    ├── changes/                       # M4
    └── sla/                           # M4
```

**Mẫu bên trong mỗi app** (copy từ `incidents/`):

```
apps/incidents/
├── models/
│   ├── __init__.py
│   ├── incident.py
│   ├── incident_comment.py
│   └── incident_worklog.py
├── serializers/
│   ├── __init__.py
│   ├── incident_serializer.py
│   └── incident_comment_serializer.py
├── views/
│   ├── __init__.py
│   ├── incident_view.py
│   └── incident_comment_view.py
├── services/                          # logic ghi/nghiệp vụ
│   ├── create_incident.py
│   ├── assign_incident.py
│   ├── resolve_incident.py
│   └── escalate_incident.py
├── selectors/                         # truy vấn đọc
│   └── incident_selectors.py
├── workflows/
│   └── incident_states.py             # state machine (django-fsm)
├── filters/
│   └── incident_filter.py
├── signals/
├── management/commands/
│   └── seed_incidents.py              # seed data cho Handoff Gate
├── migrations/
├── urls.py
├── apps.py
└── tests/
    ├── conftest.py
    ├── factories/
    │   └── incident_factory.py
    ├── api/
    │   ├── test_incident_create.py
    │   ├── test_incident_list.py
    │   ├── test_incident_transition.py
    │   └── test_incident_permissions.py
    └── services/
        └── test_assign_incident.py
```

### 3.3. Frontend (tổ chức theo feature, ví dụ Angular)

```
frontend/
├── src/
│   ├── app/
│   │   ├── core/                      # singleton: auth, interceptor, guard
│   │   │   ├── auth/
│   │   │   ├── interceptors/
│   │   │   └── guards/
│   │   ├── shared/                    # dùng chung toàn app
│   │   │   ├── components/
│   │   │   │   ├── ticket-status-badge/
│   │   │   │   ├── priority-tag/
│   │   │   │   ├── sla-countdown/
│   │   │   │   ├── data-table/        # generic table cho CRUD
│   │   │   │   └── dynamic-form/      # generic form
│   │   │   ├── pipes/
│   │   │   └── models/
│   │   ├── api/                       # client sinh tự động từ OpenAPI (không sửa tay)
│   │   ├── layouts/
│   │   │   ├── portal-layout/         # End User
│   │   │   └── console-layout/        # Agent/Admin
│   │   └── features/
│   │       ├── portal/
│   │       ├── incidents/
│   │       │   ├── pages/
│   │       │   ├── components/
│   │       │   ├── services/
│   │       │   └── incidents.routes.ts
│   │       ├── requests/
│   │       ├── problems/
│   │       ├── changes/
│   │       ├── cmdb/
│   │       ├── knowledge/
│   │       ├── sla/
│   │       ├── reports/
│   │       ├── chat/                  # giao diện chatbot (AI Agent)
│   │       └── admin/
│   ├── mocks/                         # mock API (MSW) khi backend chưa xong
│   ├── assets/
│   └── styles/                        # design tokens, theme
├── e2e/
├── package.json
└── Dockerfile
```

Nếu chọn React: giữ nguyên cách chia `features/`, `shared/`, `core/`, chỉ đổi phần framework.

### 3.4. AI service (FastAPI + LangChain/LangGraph)

```
ai-service/
├── app/
│   ├── main.py
│   ├── api/
│   │   ├── classify_router.py         # phân loại ticket
│   │   └── chat_router.py             # chatbot RAG
│   ├── chains/
│   │   ├── classify_chain.py
│   │   └── rag_chain.py
│   ├── graphs/
│   │   └── chat_graph.py              # LangGraph workflow
│   ├── retrievers/
│   │   └── kb_retriever.py
│   ├── prompts/
│   ├── schemas/
│   ├── config.py
│   └── clients/
│       └── backend_client.py          # gọi backend lấy KB
├── tests/
├── requirements.txt
└── Dockerfile
```

### 3.5. Deploy

```
deploy/
├── docker/
│   ├── backend.Dockerfile
│   ├── frontend.Dockerfile
│   └── ai-service.Dockerfile
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
├── k8s/                               # tùy chọn
├── monitoring/                        # Prometheus/Grafana (nếu dùng)
└── backup/                            # script backup/restore PostgreSQL
```

### 3.6. Tài liệu và quản lý dự án

```
docs/
├── architecture/
│   ├── software-architecture.md       # sơ đồ module, tầng
│   ├── erd.md
│   └── deployment-diagram.md
├── process-flows/                     # state diagram, BPMN
│   ├── incident-flow.md
│   ├── change-flow.md
│   └── request-flow.md
├── api/
│   ├── openapi.yaml                   # xuất từ drf-spectacular
│   └── collections/                   # Postman/Bruno theo module (Handoff Gate)
│       ├── accounts.json
│       ├── incidents.json
│       └── ...
├── deployment/
│   ├── local-setup.md
│   └── release-guide.md
├── decisions/                         # ADR: ghi lại quyết định kỹ thuật quan trọng
│   ├── 0001-modular-monolith.md
│   └── 0002-frontend-framework.md
├── planning/
│   ├── itsm-team-plan.md              # file kế hoạch này
│   ├── sprint-01.md                   # mục tiêu, task, kết quả từng sprint
│   └── meeting-notes/
│       └── 2026-11-03.md
├── reports/                           # báo cáo đề tài
└── CONTRIBUTING.md                    # quy ước Git, code, test
```

### 3.7. CODEOWNERS (tự động yêu cầu đúng người review)

```
# .github/CODEOWNERS
/backend/config/             @M1
/backend/apps/core/          @M1
/backend/apps/accounts/      @M1
/backend/apps/audit/         @M1
/backend/apps/cmdb/          @M1
/backend/apps/incidents/     @M2
/backend/apps/problems/      @M2
/backend/apps/reporting/     @M2
/backend/apps/service_*/     @M3
/backend/apps/knowledge/     @M3
/backend/apps/notifications/ @M3
/backend/apps/changes/       @M4
/backend/apps/sla/           @M4
/deploy/                     @M5
/.github/                    @M5
/docker-compose*.yml         @M5
/frontend/                   @M6
/ai-service/                 @M6
/docs/api/collections/       @M1 @M2 @M3 @M4
```

Thay `@M1`... bằng username GitHub của từng người. Kết hợp với branch protection (bắt buộc review từ code owner), quy tắc "chỉ M6 sửa `frontend/`" được thực thi tự động thay vì dựa vào tự giác.

### 3.8. Quy ước đặt tên

| Loại | Quy ước | Ví dụ |
|---|---|---|
| Thư mục, file Python | `snake_case` | `assign_incident.py` |
| Thư mục, file frontend | `kebab-case` | `sla-countdown/` |
| Nhánh Git | `feature/<module>-<mô-tả>` | `feature/incidents-assign` |
| Commit | Conventional Commits | `feat(incidents): add assign endpoint` |
| Test | `test_<đối tượng>_<hành vi>.py` | `test_incident_transition.py` |
| Collection | `<module>.json` | `incidents.json` |

### 3.9. Quy ước tầng trong mỗi app

Quy ước trong mỗi app: mỗi model, serializer, view, service một file; truy vấn đọc nằm ở `selectors/`, logic ghi nằm ở `services/`.

```
views → serializers (validate)
views → services (ghi) → models
views → selectors (đọc) → models
```

---

## 4. Lộ trình theo sprint (mỗi sprint 2 tuần, 5 sprint)

| Sprint | M1 | M2 | M3 | M4 | M5 | M6 |
|---|---|---|---|---|---|---|
| **1** | Khung Django, Auth JWT, RBAC, User/Department | Model + state machine Incident | Model Catalog, Request | Model SLAPolicy, BusinessCalendar | **Docker Compose dev, CI (lint + test), môi trường dev** | Chọn thư viện UI, design system, layout, Login, Portal khung |
| **2** | AssignmentGroup, Audit log | API Incident, comment, worklog | API Request + Approval | SLA timer, Celery/Beat | Redis, Celery worker/beat trong Compose, môi trường staging | Trang Incident, My Tickets, Service Catalog |
| **3** | CMDB (ConfigItem, CIRelationship) | Problem, Known Error | Knowledge Base, Notification (email, in-app) | Change + CAB, risk assessment | CD tự động lên staging, MinIO, backup DB | Trang Request, KB, Dashboard |
| **4** | Hoàn thiện phân quyền, review tích hợp | Escalate, Reporting API, liên kết CMDB | Duyệt/xuất bản KB, feedback | Lịch Change, cảnh báo vi phạm SLA | Monitoring/logging, HTTPS, tối ưu image | Trang Problem, Change, CMDB, Report |
| **5** | Tài liệu backend, hardening | Test, sửa lỗi | Test, sửa lỗi | Test, sửa lỗi | Triển khai bản demo, kịch bản rollback, tài liệu triển khai | **AI Agent** + giao diện chat, hoàn thiện UI |

### 4.1. Thứ tự bắt đầu (Sprint 1)

**Bắt đầu từ M1 (Backend Lead) và M5 (DevOps), chạy song song ngay ngày đầu.** Mọi người còn lại phụ thuộc vào hai người này, nên đây là đường găng (critical path) của dự án.

```
Ngày 1-2   M1: dựng khung Django, base model, cấu hình pytest, conftest
           M5: Dockerfile + docker-compose (postgres, redis), CI skeleton (lint + pytest)
           M6: chọn framework/UI lib, dựng design system, cấu hình mock API   (không phụ thuộc backend)
           M2, M3, M4: viết ERD, state diagram, danh sách endpoint của module mình (chưa code vào repo)

Ngày 3     M1 merge khung dự án vào `develop`  ← mốc mở khóa cho M2, M3, M4
           M5 gắn khung vào Compose và CI, cả nhóm chạy được `docker compose up`

Ngày 3-7   M2, M3, M4: viết model, state machine, test đầu tiên trên khung của M1
           M1: Auth JWT, User/Department/Role
           M5: hoàn thiện CI chặn merge (test + coverage), môi trường dev

Ngày 8-10  M1: RBAC dùng được → M2, M3, M4 gắn phân quyền vào API
           M6: dựng Login, layout, trang đầu tiên, nối API thật khi M1 bàn giao Auth
```

| Thứ tự | Thành viên | Lý do |
|---|---|---|
| 1 | **M1** | Khung dự án, base model, nền tảng test, Auth/RBAC là thứ mọi module backend phải dùng |
| 1 (song song) | **M5** | Không có Docker Compose và CI thì cả nhóm không chạy và không kiểm thử thống nhất được |
| 1 (song song) | **M6** | Design system và mock API không cần chờ backend, tận dụng thời gian chờ |
| 2 | **M2, M3, M4** | Bắt đầu viết code từ ngày 3, sau khi M1 mở khóa khung dự án; trước đó chuẩn bị thiết kế |

---

## 5. Nhiệm vụ chi tiết từng thành viên

### M1: Backend Lead (Core, Accounts, Audit, CMDB)

- [ ] Khởi tạo project Django, cấu hình `settings/{base,dev,prod}`
- [ ] Model: `User`, `Department`, `Role`, `AssignmentGroup`
- [ ] Auth JWT (`simplejwt`), refresh token, đổi mật khẩu
- [ ] RBAC: User → Role → Permission, phân quyền theo đối tượng (`django-guardian` hoặc `django-rules`)
- [ ] Base class dùng chung: `TimeStampedModel`, `SoftDeleteModel`, pagination, exception handler
- [ ] Audit log (`django-simple-history`), API xem lịch sử thay đổi
- [ ] CMDB: `ConfigItem`, `CIType`, `CIRelationship`, API gắn CI vào ticket
- [ ] Viết quy ước code, template PR, review PR của M2 đến M4
- [ ] Cấu hình OpenAPI (`drf-spectacular`) cho cả hệ thống
- [ ] Dựng nền tảng test dùng chung ở sprint 1: `conftest.py`, factory cho User/Role/Group, helper đăng nhập theo role, ngưỡng coverage
- [ ] Viết API test + Postman/Bruno collection cho Auth, Accounts, Audit, CMDB trước khi bàn giao (Handoff Gate, mục 7)

### M2: Backend (Incident, Problem, Reporting)

- [ ] Model: `Incident`, `IncidentComment`, `IncidentWorklog`
- [ ] State machine (`django-fsm`): New → Assigned → In Progress → On Hold → Resolved → Closed (Reopened)
- [ ] Priority matrix: Impact × Urgency → P1 đến P4
- [ ] Tự động gán nhóm xử lý theo category
- [ ] Services: tạo, gán, giải quyết, đóng, escalate
- [ ] Problem + Known Error, liên kết nhiều Incident → 1 Problem
- [ ] Reporting API: số lượng ticket theo trạng thái, MTTR, tỷ lệ đạt SLA, ticket theo nhóm
- [ ] Filter, tìm kiếm, sắp xếp (`django-filter`)
- [ ] Viết API test + collection cho Incident, Problem, Reporting; qua Handoff Gate trước khi giao cho M6 và M3

### M3: Backend (Catalog, Request, Knowledge, Notification)

- [ ] Service Catalog: danh mục, item, form động
- [ ] Service Request + luồng phê duyệt (Dept Manager)
- [ ] Knowledge Base: bài viết, danh mục, duyệt, xuất bản, feedback hữu ích
- [ ] Tìm kiếm KB (full-text PostgreSQL), chuẩn bị dữ liệu cho RAG
- [ ] Notification: kênh email, in-app (WebSocket hoặc polling), template thông báo
- [ ] Celery task gửi email bất đồng bộ
- [ ] Viết API test + collection cho Catalog, Request, Knowledge, Notification; qua Handoff Gate trước khi giao cho M6 (và giao dữ liệu KB cho M6 làm RAG)

### M4: Backend (Change, SLA)

- [ ] SLA: `SLAPolicy`, `BusinessCalendar`, `SLATimer`
- [ ] Tính `response_due`, `resolution_due` theo ưu tiên và giờ làm việc
- [ ] Celery Beat quét ticket sắp vi phạm, cảnh báo và escalate
- [ ] Change: model, state machine (Draft → Submitted → CAB Review → Approved/Rejected → Scheduled → Implementing → Review → Closed)
- [ ] Đánh giá rủi ro, bỏ phiếu CAB, lịch Change
- [ ] Liên kết Change với CI (CMDB của M1)
- [ ] Viết API test + collection cho SLA và Change (gồm test timer, giờ làm việc, vi phạm SLA, luồng CAB); qua Handoff Gate trước khi giao cho M6

### M5: DevOps (CI/CD + Deployment)

- [ ] `docker-compose.yml`: nginx, backend, frontend, postgres, redis, celery-worker, celery-beat, minio
- [ ] Dockerfile multi-stage cho backend, frontend, ai-service
- [ ] CI: lint (ruff/flake8, eslint), test, build image, quét bảo mật cơ bản
- [ ] CD: tự động deploy lên staging khi merge vào `develop`; deploy production khi tag release
- [ ] Quản lý biến môi trường và secret (`.env.example`, secret của CI)
- [ ] Môi trường: dev (local), staging, production (VPS hoặc cloud)
- [ ] Nginx: reverse proxy, HTTPS, rate limit
- [ ] Backup/restore PostgreSQL, kịch bản rollback
- [ ] Logging và monitoring cơ bản (healthcheck, Prometheus/Grafana hoặc tương đương)
- [ ] Tài liệu: hướng dẫn chạy local, hướng dẫn triển khai, sơ đồ deployment
- [ ] CI làm cổng chặn merge: chạy test API, kiểm tra độ phủ ≥ ngưỡng, kiểm tra schema OpenAPI
- [ ] Sau mỗi lần deploy staging, tự chạy Newman/Bruno CLI với collection của các module và báo kết quả vào kênh chung
- [ ] Cung cấp mẫu PR có sẵn mục Handoff Note (`.github/pull_request_template.md`)

> **Lưu ý:** Sprint 1 của M5 là nhiệm vụ chặn tiến độ của cả nhóm. Cần có Docker Compose dev chạy được trong tuần đầu tiên.

### M6: Frontend + AI Agent

**Frontend**

- [ ] Chọn framework (Angular hoặc React) và thư viện UI (Material/PrimeNG/MUI/Ant Design), chốt ở sprint 1
- [ ] Design system: layout, bảng, form, badge trạng thái, tag ưu tiên, đồng hồ đếm SLA
- [ ] Sinh client TypeScript tự động từ OpenAPI, không viết tay
- [ ] Mock API (MSW hoặc json-server) khi backend chưa sẵn sàng
- [ ] Portal End User: tạo ticket, My Tickets, Service Catalog, tìm KB
- [ ] Agent Console: danh sách và chi tiết Incident, Problem, Change, CMDB
- [ ] Admin Panel: người dùng, nhóm, SLA, danh mục
- [ ] Dashboard và Report
- [ ] Generic table/form component cho các trang CRUD đơn giản
- [ ] Với vai trò người nhận: smoke test từng module backend bằng collection trên staging trong 1 ngày làm việc, trả lại kèm lỗi cụ thể nếu không đạt (Handoff Gate, mục 7.3)

**AI Agent (sprint 5, chỉ làm MVP)**

- [ ] Phân loại ticket tự động (category, priority) từ nội dung tiếng Việt
- [ ] Chatbot RAG tra cứu Knowledge Base (dùng LangChain/LangGraph có sẵn)
- [ ] Endpoint FastAPI cho `ai-service`, backend gọi qua HTTP
- [ ] Giao diện chat trong Portal

---

## 6. Phụ thuộc giữa các thành viên

| Bên cần | Cần từ | Nội dung | Hạn |
|---|---|---|---|
| Tất cả | M5 | Docker Compose dev chạy được | Tuần 1 |
| M2, M3, M4 | M1 | Auth, RBAC, base model | Cuối sprint 1 |
| M6 | M1 đến M4 | Schema OpenAPI của từng module | Đầu mỗi sprint |
| M2, M4 | M4, M1 | SLA timer, CI để gắn vào ticket | Sprint 2 đến 3 |
| M3 | M2 | Sự kiện ticket để gửi notification | Sprint 2 |
| M6 (AI) | M3 | Dữ liệu KB đủ lớn để làm RAG | Sprint 4 |
| M6 | M5 | Môi trường staging để test tích hợp | Sprint 2 |

---

## 7. Cổng kiểm thử API trước khi bàn giao (Handoff Gate)

**Nguyên tắc: chưa qua test API thì chưa được bàn giao cho người kế tiếp.** Áp dụng cho mọi lần bàn giao:

| Bàn giao | Người giao | Người nhận |
|---|---|---|
| Backend → Frontend | M1, M2, M3, M4 | M6 |
| Backend → Backend (module phụ thuộc nhau) | M1 → M2, M3, M4; M4 ↔ M2; M3 ← M2 | Module phụ thuộc |
| KB → AI Agent | M3 | M6 |
| Code → Staging | M1 đến M4, M6 | M5 |

### 7.1. Quy trình

```
Dev code xong
  → Tự chạy test API (local)
  → Mở PR kèm Handoff Note
  → CI xanh (unit + API test + contract test)
  → Reviewer duyệt
  → Merge vào develop, tự deploy lên staging (M5)
  → Người nhận chạy smoke test trên staging
  → Chấp nhận (bàn giao hoàn tất)  hoặc  Trả lại (kèm lỗi cụ thể)
```

### 7.2. Checklist của người giao (bắt buộc trước khi bàn giao)

Với **mỗi endpoint**, phải có test API cho các trường hợp:

- [ ] **Thành công** (200/201/204) với dữ liệu hợp lệ, kiểm tra cả cấu trúc response
- [ ] **Dữ liệu sai** (400): thiếu trường, sai kiểu, sai giá trị
- [ ] **Chưa đăng nhập** (401)
- [ ] **Sai quyền** (403): kiểm thử từng role liên quan (End User, Agent, Manager...)
- [ ] **Không tồn tại** (404)
- [ ] **Phân trang, lọc, sắp xếp** đối với endpoint danh sách
- [ ] **Chuyển trạng thái sai** bị từ chối (đối với Incident, Change, Request...)
- [ ] **Quy tắc nghiệp vụ** của module (ví dụ: không đóng Incident khi chưa Resolved; Change chưa qua CAB không được Implement)

Ngoài test, người giao phải bàn giao đủ:

- [ ] Độ phủ test của `services/` và `selectors/` ≥ **80%**
- [ ] Schema OpenAPI đã cập nhật (`drf-spectacular`), không có cảnh báo
- [ ] **Postman/Bruno collection** của module, lưu tại `docs/api/collections/<module>.json`
- [ ] **Seed data/fixture** để người nhận có dữ liệu thử ngay
- [ ] **Handoff Note** trong mô tả PR (mẫu ở mục 7.5)

### 7.3. Checklist của người nhận (smoke test trong 1 ngày làm việc)

- [ ] Import collection, chạy toàn bộ trên staging, tất cả request mẫu đều đạt
- [ ] Kiểm tra response khớp schema OpenAPI (client TypeScript sinh ra không lỗi, đối với M6)
- [ ] Thử ít nhất 1 luồng nghiệp vụ end-to-end bằng dữ liệu seed
- [ ] Kết quả: **Chấp nhận** (đóng task) hoặc **Trả lại** (ghi rõ endpoint, request, kết quả mong đợi, kết quả thực tế)

Nếu quá 1 ngày làm việc người nhận không phản hồi, coi như chấp nhận ngầm và người giao đi tiếp việc khác.

### 7.4. Công cụ và cấu trúc test

| Mục đích | Công cụ |
|---|---|
| Test API | `pytest-django` + DRF `APIClient` |
| Dữ liệu test | `factory_boy` |
| Đo độ phủ | `pytest-cov` |
| Contract test với OpenAPI (tùy chọn) | `schemathesis` |
| Chạy collection tự động trong CI/staging | Newman (Postman) hoặc Bruno CLI |

Mỗi app tổ chức test thành các file nhỏ, mỗi file một nhóm endpoint:

```
apps/incidents/tests/
├── factories/
│   └── incident_factory.py
├── api/
│   ├── test_incident_create.py
│   ├── test_incident_list.py
│   ├── test_incident_transition.py
│   └── test_incident_permissions.py
├── services/
│   └── test_assign_incident.py
└── conftest.py
```

M5 cấu hình CI để **chặn merge** nếu: test API fail, độ phủ dưới ngưỡng, hoặc schema OpenAPI lỗi. Sau khi deploy staging, CI chạy Newman/Bruno với collection của các module để phát hiện lỗi tích hợp sớm.

### 7.5. Mẫu Handoff Note (dán vào mô tả PR)

```
## Handoff Note
**Module:** incidents
**Người giao:** @M2        **Người nhận:** @M6
**Endpoint thêm/sửa:**
- POST /api/v1/incidents/            (role: End User, Agent)
- PATCH /api/v1/incidents/{id}/assign (role: Agent L1+)
**Thay đổi so với schema cũ:** (breaking change? có / không)
**Collection:** docs/api/collections/incidents.json
**Seed data:** python manage.py seed_incidents
**Độ phủ test:** 86%
**Lưu ý nghiệp vụ / trường hợp biên:** ...
**Kết quả smoke test của người nhận:** [ ] Chấp nhận  [ ] Trả lại (lý do: ...)
```

### 7.6. Hạn bàn giao trong sprint

- Bàn giao API **chậm nhất thứ Tư của tuần thứ 2** mỗi sprint, để người nhận còn thứ Năm và thứ Sáu tích hợp và smoke test.
- Task chưa qua Handoff Gate vào cuối sprint **không được tính hoàn thành** và chuyển sang sprint sau.

### 7.7. Cấu hình pytest tối thiểu (M1 dựng ở sprint 1)

`backend/requirements/dev.txt`:

```
pytest
pytest-django
pytest-cov
factory-boy
schemathesis
```

`backend/pytest.ini`:

```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.dev
python_files = test_*.py
addopts = --reuse-db --cov=apps --cov-report=term-missing --cov-fail-under=80
```

`backend/conftest.py` cung cấp các fixture dùng chung: `api_client`, và một fixture đăng nhập theo role cho mỗi vai trò (`end_user_client`, `agent_client`, `manager_client`...), để mọi module test phân quyền theo cùng một cách.

---

## 8. Quy ước cộng tác

### Git

- Nhánh: `main` (ổn định), `develop` (tích hợp), `feature/<module>-<mô-tả>`
- Mọi thay đổi qua Pull Request, ít nhất 1 người review (backend: M1 review; frontend: một backend dev xem lại tính đúng của API; hạ tầng: M1 review)
- Commit theo Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`)

### API-first

- Thay đổi schema API phải báo M6 ngay trong kênh chung.
- Mỗi endpoint mới phải có docstring/schema trong `drf-spectacular` trước khi merge.
- Backend dev muốn một màn hình mới thì mở issue kèm wireframe, không tự viết giao diện.

### Quan hệ giữa các module

- Module này cần dữ liệu module khác thì gọi qua `services/` hoặc `selectors/`, không import trực tiếp `models/` của module khác.

### Họp

| Cuộc họp | Tần suất | Nội dung |
|---|---|---|
| Stand-up ngắn | 2 lần/tuần | Tiến độ, vướng mắc |
| Đối chiếu API | Đầu mỗi sprint | Chốt schema giữa backend và M6 |
| Sprint review/retro | Cuối mỗi sprint | Demo, rút kinh nghiệm |

---

## 9. Định nghĩa hoàn thành (Definition of Done)

Một task chỉ được tính xong khi:

- [ ] Code đã merge vào `develop` qua PR đã được review
- [ ] **Test API đầy đủ theo mục 7.2** (thành công, 400, 401, 403, 404, nghiệp vụ) và CI xanh
- [ ] Độ phủ test `services/` và `selectors/` ≥ 80%
- [ ] API có mô tả trong OpenAPI, có Postman/Bruno collection và seed data
- [ ] Có tài liệu ngắn trong `docs/` nếu là tính năng mới
- [ ] Đã chạy được trên môi trường staging
- [ ] **Người nhận đã smoke test và chấp nhận (Handoff Gate, mục 7)**

---

## 10. Rủi ro và cách giảm thiểu

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| M6 quá tải do làm cả frontend và AI | Cao | AI để sprint cuối và chỉ làm MVP; dùng generic component; nếu trễ sau sprint 2, giao phần tích hợp AI phía backend cho M3 |
| Backend chưa xong thì frontend bị chặn | Trung bình | API-first, mock theo schema, client sinh tự động |
| Hạ tầng chậm làm cả nhóm không chạy được | Cao | M5 ưu tiên Docker Compose dev ở tuần 1 |
| Xung đột khi hai người cùng sửa một module | Trung bình | Quy tắc sở hữu thư mục, review PR bắt buộc |
| M1 quá tải (Core + CMDB + review) | Trung bình | Nếu cần, chuyển CMDB sang M4 sau sprint 2; M5 hỗ trợ Audit |
| Dữ liệu demo nghèo nàn | Thấp | Viết seed data/fixture từ sprint 2 (mỗi người seed module của mình) |
| Bàn giao API lỗi làm M6 và các module phụ thuộc bị chặn | Cao | Handoff Gate (mục 7): test API đầy đủ, collection, smoke test của người nhận, CI chặn merge |
| Test API làm chậm tiến độ backend | Trung bình | Viết test song song với code theo từng endpoint; dùng factory và fixture dùng chung do M1 dựng ở sprint 1 |

---

## 11. Việc cần chốt ngay trong tuần đầu

1. Điền tên thành viên cho M1 đến M6 (M1 nên là người vững backend nhất).
2. Chốt framework frontend (Angular hay React) và thư viện UI.
3. Chốt nơi chạy deployment (VPS, cloud hay chỉ máy cục bộ) và nền tảng CI (GitHub Actions hay GitLab CI).
4. Chốt bối cảnh tổ chức giả định (trường đại học, ngân hàng, công ty phần mềm) để định nghĩa phòng ban, danh mục dịch vụ và SLA mẫu.
5. Chốt ngày bắt đầu sprint 1 và thời hạn nộp đề tài để quy đổi ra lịch cụ thể.
6. Chốt công cụ test API (Postman hay Bruno) và ngưỡng độ phủ test (đề xuất 80%).