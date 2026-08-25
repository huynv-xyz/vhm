> **TÀI LIỆU NỘI BỘ** — Tài liệu mô tả kiến trúc mục tiêu L2 của năng lực quản lý hồ sơ Nhà ở Xã hội. Không chia sẻ ra ngoài phạm vi dự án khi chưa được phê duyệt.

# L2 - VHMKDO2O - Dịch vụ quản lý hồ sơ Nhà ở Xã hội

| **Trường** | **Nội dung** |
| --- | --- |
| **Trạng thái** | **ĐANG THẨM ĐỊNH (UNDER REVIEW)** |
| **Phiên bản & lịch sử thay đổi** | `v0.9.2` — 25/08/2026 — Chuẩn hóa scope diagram, integration/DFD/sequence, production topology và risk idempotency/Redis.<br>`v0.9.1` — 25/08/2026 — Bổ sung ownership và consent gate tại bước upload/submit hồ sơ NOXH trực tuyến.<br>`v0.9.0` — 15/08/2026 — Thiết lập kiến trúc mục tiêu cho dịch vụ quản lý hồ sơ NOXH. |
| **Chủ sở hữu tài liệu** | TBD |
| **Chủ sở hữu hệ thống** | TBD |
| **Hệ thống** | `vhm-dossier-core` — modular monolith quản lý hồ sơ và pipeline NOXH |
| **Hệ thống liên quan** | Market Landing Page, Agent/Market BFF, `vhm-ocr-ekyc`, PostgreSQL, Redis, Kafka, File Management, Market/Project, Message Delivery, TTOL |
| **Đội ngũ/PIC** | Backend: TBD · Kiến trúc: TBD · Tích hợp: TBD · ANBM: TBD · Quyền riêng tư dữ liệu: TBD · Vận hành: TBD |
| **Người rà soát/phê duyệt** | Sản phẩm/BA: TBD · Kiến trúc: TBD · ANBM: TBD · DBA/Vận hành: TBD · QA: TBD |
| **Mốc thiết kế** | Kiến trúc mục tiêu phục vụ thẩm định giải pháp và làm đầu vào cho thiết kế L3 |
| **Phạm vi hệ thống** | Dossier Core và các ranh giới tích hợp thể hiện trong sơ đồ ngữ cảnh |
| **Tài liệu nguồn** | SRS/BRD NOXH: TBD · [TDD gốc Dossier](https://vin3s.atlassian.net/wiki/spaces/VARW/pages/3009027308/L2+-+VHMKDO2O+-+Nh+x+h+i) · [L2 OCR/eKYC dùng chung](https://vin3s.atlassian.net/wiki/spaces/VARW/pages/3014268156/L2+-+VHMKDO2O+-+D+ch+v+OCR+eKYC) |
| **Lần rà soát gần nhất** | 25/08/2026 |

## Approval & Review Gates

| **Vai trò rà soát/phê duyệt** | **Phạm vi rà soát** | **Quyết định** | **Ngày xác nhận** |
| --- | --- | --- | --- |
| Chủ sở hữu Sản phẩm/Nghiệp vụ | Luồng đăng ký, checklist, duyệt PKD/PTT/SXD, SLA nhắc bổ sung | Chờ rà soát | — |
| Kiến trúc Ứng dụng/Giải pháp | Ranh giới modular monolith, API, pipeline và tính nhất quán | Chờ rà soát | — |
| Kiến trúc Tích hợp | BFF, File, Market, OCR, Message Delivery, TTOL, Kafka | Chờ rà soát | — |
| ANBM | IAM nội bộ, actor context, chống phát lại, PII và file | Chờ rà soát | — |
| Quyền riêng tư/Pháp chế | Consent cho hành trình NOXH, dữ liệu CCCD, lưu giữ, truy cập và xóa dữ liệu | Chờ rà soát | — |
| DBA/Vận hành/QA | Migration, dung lượng, quan sát, DR và bằng chứng kiểm thử | Chờ rà soát | — |

## Governance Gates

| **Chuyển trạng thái** | **Điều kiện đầu vào** |
| --- | --- |
| `DRAFT → UNDER REVIEW` | Scope, requirement, sơ đồ và decision đủ điều kiện rà soát; mọi deviation, phụ thuộc và rủi ro có định danh. |
| `UNDER REVIEW → APPROVED` | Có owner/reviewer đích danh; contract Checklist/File/Pipeline được chốt; security, privacy, NFR và vận hành có phê duyệt. |
| `APPROVED → IMPLEMENTATION BASELINE` | OpenAPI, migration, contract test, E2E, load test, DR/runbook và production configuration có bằng chứng. |

## L3 Artefact Register

| **Tài liệu L3** | **Trạng thái** | **Chủ sở hữu** | **Cổng bắt buộc** | **Tham chiếu** |
| --- | --- | --- | --- | --- |
| OpenAPI BFF ↔ Dossier Core | DRAFT | Backend/Tích hợp | Trước duyệt API | Liên kết tài liệu chính thức: TBD |
| Form Data Contract Social Housing v1 | DRAFT | Backend/BA | Trước UAT | Liên kết tài liệu chính thức: TBD |
| Pipeline Definition Social Housing v1 | DRAFT | Backend/BA | Trước UAT | Liên kết tài liệu chính thức: TBD |
| Contract Checklist chuẩn | **CHƯA CÓ** | BA/Tích hợp/Backend | Trước production | Chưa chốt nguồn authority và version |
| Contract sở hữu/quyền upload file | **CHƯA ĐỦ** | File Team/ANBM/Backend | Trước production | Chưa có owner/upload-grant contract |
| Consent Notice, Evidence, UI & Withdrawal Contract | **PENDING — DESIGN REQUIRED** | Product/Market/Backend/Privacy/Legal | Trước UAT dữ liệu cá nhân | L2 chỉ chốt ownership/gate; copy, UI, schema và workflow thuộc L3 |
| Agent Create Idempotency Decision | **PENDING** | Product/Agent/Backend/Kiến trúc | Trước duyệt API | Yêu cầu key đồng nhất hoặc deviation/risk acceptance có kiểm soát |
| Production Landing Zone, Network & Redis HA Design | **PENDING** | Platform/SRE/ANBM/Backend | Trước OAT | Chốt account/region/CIDR, AZ, ingress/egress, replica, Redis mode và failover SLO |
| Runbook migration/rollback/DR | Planned | DBA/Vận hành | Trước OAT | TBD |
| Runbook dashboard/cảnh báo/sự cố | Planned | Vận hành | Trước go-live | TBD |

## Quy ước trạng thái thiết kế

| **Nhãn** | **Ý nghĩa** |
| --- | --- |
| `BẮT BUỘC` | Phải hoàn thành và có bằng chứng trước production. |
| `ĐỀ XUẤT` | Quyết định kiến trúc đang chờ phê duyệt. |
| `BÊN NGOÀI` | Do dependency quy định; cần contract test và SLA. |
| `TBD` | Cần owner xác nhận trước cổng tương ứng. |

Tài liệu này mô tả **kiến trúc mục tiêu L2**. Dossier được thiết kế theo modular monolith với pipeline thực thi nội bộ.

# 1. Business Objectives & Scope

### Business Context & Objectives

`vhm-dossier-core` số hóa việc tiếp nhận, cập nhật, kiểm tra và phê duyệt hồ sơ đăng ký Nhà ở Xã hội. Khách hàng có thể tự tạo và nộp hồ sơ trên Market Landing Page; đại lý vẫn có thể thao tác trên kênh nghiệp vụ. Market hiển thị/thu consent, còn Domain lưu và kiểm tra consent tại bước upload và submit hồ sơ trực tuyến. Hồ sơ được xử lý qua các nhóm PKD, PTT và đầu mối SXD; mọi hành động hợp lệ được quyết định bởi pipeline nội bộ và actor context đã ký.

#### Current Business Problem

- Hồ sơ giấy/Excel/email khó kiểm soát tính đầy đủ, phiên bản và lịch sử xử lý.
- Nhập tay CCCD và thông tin khách hàng dễ sai; tài liệu thiếu hoặc không tồn tại chỉ được phát hiện muộn.
- Nhiều người có thể tạo hồ sơ cho cùng khách hàng và dự án, dẫn đến trùng nghiệp vụ và tranh chấp căn.
- Quyền xem/xử lý theo đại lý, đội nhóm, dự án và cấp duyệt phải được thực thi nhất quán giữa BFF và core.
- Luồng trả bổ sung cần SLA, nhắc hẹn, lịch sử và khả năng tiếp tục đúng cấp duyệt.
- Các tích hợp File, OCR, Market, TTOL và Message Delivery có độ sẵn sàng khác nhau; không được làm mất trạng thái đã commit.

#### Business Objectives

- Tạo hồ sơ `DRAFT` nhanh với form rỗng hoặc một phần; create không tự động submit.
- Duy trì một snapshot `formData`/`metadata` có version, lịch sử trạng thái và projection pipeline.
- Ngăn hồ sơ active trùng theo `(applicant.idNumber, projectRegistration.projectId)` kể cả khi có race.
- Bảo đảm checklist bắt buộc đã được upload trước `SUBMIT`.
- Thực thi pipeline PKD → PTT → SXD, bao gồm trả bổ sung, phân công, nhận xử lý, cấp/thu hồi căn và hồ sơ giấy.
- Cung cấp list/detail/statistics/export và progress checklist cho kênh nghiệp vụ qua BFF.
- Tách việc gửi sự kiện và notification khỏi transaction nghiệp vụ bằng outbox.
- Bảo vệ PII bằng actor context, visibility, ownership, masking và xác thực nội bộ có chữ ký.
- Lưu bằng chứng consent có thể kiểm chứng, hỗ trợ rút consent và chặn upload/submit khi consent không hợp lệ.

## 1.1 In Scope

| **Capability** | **Phạm vi** | **Yêu cầu thiết kế** |
| --- | --- | --- |
| Hồ sơ | Create/read/list/update/delete DRAFT, statistics, lookup theo contact | `BẮT BUỘC` |
| Luồng đăng ký công khai | Create DRAFT → prepare upload → PATCH snapshot → submit | `BẮT BUỘC` |
| Consent hành trình NOXH | Market hiển thị/thu lựa chọn; Domain lưu và kiểm tra consent tại upload/submit, đồng thời xử lý rút consent | `BẮT BUỘC` |
| Checklist | Snapshot từ nguồn chuẩn, progress, missing/invalid, readiness submit | `BẮT BUỘC` |
| Pipeline | State/action/role/ownership trong modular monolith | `BẮT BUỘC` |
| Phân công reviewer | Manual, round-robin PKD và roster TTOL cho PTT/SXD | `BẮT BUỘC` |
| Quyền dự án | Grant/revoke/list/group theo team/project/scope | `BẮT BUỘC` |
| OCR CCCD | Tích hợp capability dùng chung `vhm-ocr-ekyc` theo mô hình bất đồng bộ | `BẮT BUỘC` |
| File | Chuẩn bị upload, validate tồn tại và quyền attach | `BẮT BUỘC` |
| Notification/reminder | Transactional outbox và nhắc bổ sung T+6/T+18 | `BẮT BUỘC` |
| Báo cáo/tải file | Export danh sách NOXH; tải hợp đồng/tệp đính kèm | `BẮT BUỘC` |
| Notes/hardcopy | Ghi chú và theo dõi hồ sơ giấy | `BẮT BUỘC` |

## 1.2 Out of Scope

- Camunda/Zeebe hoặc BPMN engine bên ngoài; pipeline hiện chạy trong cùng ứng dụng và transaction.
- Sở hữu master data dự án, người dùng, đội nhóm, ngày nghỉ hoặc file binary.
- Thuật toán, provider credential, queue/worker và dữ liệu raw OCR; toàn bộ thuộc `vhm-ocr-ekyc`.
- Việc `vhm-ocr-ekyc` tự hiển thị hoặc lưu consent gốc; capability không có kênh tương tác trực tiếp với khách hàng.
- Quyết định pháp lý về đủ điều kiện NOXH dựa riêng vào OCR.
- Quản trị checklist chuẩn ở trình duyệt; `isRequired` do client gửi chưa thể là authority production.
- Thanh toán, ký điện tử và tích hợp tự động trực tiếp với cơ quan SXD.
- Chứng minh người upload sở hữu file nếu File Service chưa trả owner/upload-grant.

### Assumptions, Constraints & Dependencies

| **ID** | **Giả định/Ràng buộc** | **Trạng thái** | **Ảnh hưởng** |
| --- | --- | --- | --- |
| A-01 | BFF là public boundary; core chỉ cung cấp `/internal/v1/**` | Quyết định hiện hành | Client không gọi core trực tiếp. |
| A-02 | PostgreSQL là nguồn sự thật của hồ sơ, checklist, projection pipeline và outbox | Quyết định hiện hành | Mọi mutation trọng yếu dùng cùng transaction DB. |
| A-03 | Create chỉ tạo `DRAFT`; submit là lệnh riêng | Contract bắt buộc | BFF phải điều phối đủ bốn bước đăng ký. |
| A-04 | Kafka có thể giao lặp; relay/consumer phải idempotent | Giả định nền tảng | Outbox chấp nhận publish lặp, không làm lặp quyết định nghiệp vụ. |
| A-05 | File path là opaque; namespace upload độc lập với dossier ID | Đã xác minh STG | Không áp dụng kiểm tra prefix `registrations/{dossierId}/`. |
| A-06 | Mỗi dossier phải nhận một pipeline ID/version xác định | `BẮT BUỘC` | Không lựa chọn pipeline theo thứ tự cấu hình. |
| A-07 | Structural guard và Form Data Contract phải được thực thi trước persistence | `BẮT BUỘC` | Production không được bỏ qua schema validation. |
| A-08 | OCR tài liệu đi qua `vhm-ocr-ekyc`; dossier không gọi provider trực tiếp | Quyết định kiến trúc mục tiêu | Cần OpenAPI L3, workload IAM và migration khỏi client OCR legacy. |
| A-09 | External services có contract/SLA riêng | `BÊN NGOÀI` | Cần timeout, retry hữu hạn, monitoring và contract test. |
| A-10 | `vhm-dossier-core` là Domain Backend Service sở hữu consent cho hành trình NOXH; BFF không là consent authority | Quyết định kiến trúc | Cách lưu bằng chứng và workflow chi tiết thuộc contract L3. |
| A-11 | Không có consent hợp lệ thì Core không cho phép upload hoặc submit hồ sơ trực tuyến | `BẮT BUỘC` | Quy tắc được thực thi tại Domain, không tin trạng thái do client tự khai. |

### Stakeholders & Personas

| **Nhóm** | **Trách nhiệm/quyền** |
| --- | --- |
| Khách hàng / `APPLICANT` | Cung cấp hoặc rút consent khi upload/submit hồ sơ NOXH trực tuyến. |
| Đại lý / `APPLICANT_AGENT` | Tạo và cập nhật DRAFT, upload, submit/resubmit, xem hồ sơ trong phạm vi. |
| PKD / `PKD`, `PKD_LEAD` | Phân công/nhận hồ sơ, cấp căn, duyệt, trả bổ sung, từ chối, hồ sơ giấy. |
| PTT / `PTT`, `PTT_LEAD` | Kiểm tra thủ tục, duyệt/trả bổ sung/từ chối, chuyển SXD. |
| Đầu mối SXD | Được mô hình hóa bằng stage `SXD` nhưng xử lý qua roster/role PTT hiện hành. |
| BO/Admin | Quản lý quyền dự án, tra cứu/báo cáo và vận hành. |
| Market Landing Page / Market BFF | Xác thực khách hàng, hiển thị notice, thu/chuyển lựa chọn và gọi Core; không tự quyết consent hợp lệ hoặc là nguồn bằng chứng cuối. |
| Agent/BO BFF | Xác thực kênh nghiệp vụ, map DTO, ký request và actor context khi gọi core. |
| Privacy/Legal | Phê duyệt notice/copy, purpose, retention/deletion và điều kiện phải xin lại consent. |
| File/Market/`vhm-ocr-ekyc`/Message Delivery/TTOL | Cung cấp năng lực tích hợp không thuộc sở hữu core. |

### Personal Data Processing Summary

| **Dữ liệu** | **Mục đích** | **Vị trí** | **Kiểm soát hiện hành/yêu cầu** |
| --- | --- | --- | --- |
| CCCD, họ tên, ngày sinh, địa chỉ | Định danh và xử lý hồ sơ | `dossier.form_data` JSONB | Actor visibility, masking, XSS guard, mã hóa hạ tầng/TBD retention. |
| SĐT/email người nộp và vợ/chồng | Liên hệ, tìm kiếm, notification | JSONB và notification outbox | PKD masking; không ghi body nhạy cảm vào log. |
| Đường dẫn file | Gắn tài liệu và yêu cầu OCR | JSONB/checklist và OCR service | Chỉ lưu `s3PathFile`; không persist presigned URL; ownership còn thiếu. |
| Bằng chứng notice/consent | Chứng minh nội dung đã hiển thị, lựa chọn theo từng mục đích và việc rút consent | Domain store; mô hình vật lý thuộc L3 | Có version/thời điểm/chủ thể/phạm vi kiểm chứng được; không lưu CCCD, media hoặc raw OCR trong bằng chứng consent. |
| Kết quả/phê duyệt tài liệu | Readiness và quyết định | JSONB/checklist/history | Chỉ actor có quyền được mutation; lịch sử append trong snapshot. |
| Actor/reviewer | Audit và routing | audit columns, status history, stage reviewer | Actor context có chữ ký; không tin actor do client truyền. |

### System Criticality

Đề xuất **Cấp 2 — nghiệp vụ trọng yếu, xử lý dữ liệu cá nhân nhạy cảm**. Phân loại chính thức, RTO/RPO, retention và yêu cầu mã hóa phải được System Owner, ANBM, Privacy và Vận hành ký duyệt trước production.

# 2. Architecture Overview & Principles

## 2.1 Nguyên tắc thiết kế

| **Mã** | **Nguyên tắc** |
| --- | --- |
| ARCH-01 | Dossier domain và pipeline được quản lý thống nhất trong cùng ranh giới giao dịch. |
| ARCH-02 | PostgreSQL là nguồn sự thật; mutation hồ sơ, checklist, history và outbox cùng transaction. |
| ARCH-03 | Create và submit tách biệt; mọi create thành công trả về `DRAFT`. |
| ARCH-04 | Pipeline được cấu hình bằng phiên bản và thực thi trong cùng process/transaction với dossier. |
| ARCH-05 | Không tin actor/role từ request body; dùng actor context có chữ ký. |
| ARCH-06 | Dùng optimistic lock cho update/command và DB constraint cho race cuối. |
| ARCH-07 | Idempotency replay được kiểm tra trước validation và được serialize bằng advisory lock. |
| ARCH-08 | Event/notification dùng transactional outbox, chấp nhận at-least-once. |
| ARCH-09 | File path là tham chiếu opaque; không suy luận ownership từ prefix. |
| ARCH-10 | Mọi khoảng trống production phải được quản lý bằng risk/open issue, không mô tả như đã triển khai. |
| ARCH-11 | Dossier chỉ tích hợp contract chuẩn của `vhm-ocr-ekyc`; không biết provider, credential, queue hoặc raw response. |
| ARCH-12 | Market Landing/BFF hiển thị và thu lựa chọn; Dossier Domain lưu/kiểm tra consent; `vhm-ocr-ekyc` không sở hữu consent gốc. |
| ARCH-13 | Core chỉ cho phép upload và submit hồ sơ trực tuyến khi consent còn hiệu lực. |
| ARCH-14 | Market BFF không gọi OCR trực tiếp; luồng bắt buộc đi qua Core. |

## 2.2 Sơ đồ kiến trúc ứng dụng

### 2.2.1 Sơ đồ ngữ cảnh hệ thống

```mermaid
flowchart LR
    classDef inScope fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#0D3B1E
    classDef external fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,stroke-dasharray:5 5,color:#1F2937

    Customer([Khách hàng]) --> Landing[Market Landing Page]
    Landing --> MBFF[Market BFF]
    Agent([Đại lý / BO / Reviewer]) --> Channel[Ứng dụng nghiệp vụ]
    Channel --> ABFF[Agent / BO BFF]

    subgraph SCOPE[IN SCOPE — vhm-dossier-core]
        direction TB
        Core[Dossier Core<br/>API · Domain · Consent · Pipeline]
        PG[(PostgreSQL<br/>dossier_db)]
        Core <--> PG
    end

    MBFF -->|signed actor + consent decision| Core
    ABFF -->|Basic + HMAC + signed actor context| Core
    Core --> Redis[(Redis / Redisson)]
    Core --> Kafka[(Kafka)]
    Core --> File[Private File Service]
    Core --> Project[Market / Project Service]
    Core -->|authorized request + workload identity| OCR[vhm-ocr-ekyc]
    OCR --> File
    Core --> Msg[Message Delivery]
    Core --> TTOL[TTOL roster / holiday]

    class Core,PG inScope
    class Customer,Landing,MBFF,Agent,Channel,ABFF,Redis,Kafka,File,Project,OCR,Msg,TTOL external
    style SCOPE fill:#F5FBF6,stroke:#137333,stroke-width:2px
```

Khung xanh liền là phạm vi thiết kế của `vhm-dossier-core`; các node xám nét đứt là
kênh, nền tảng hoặc capability bên ngoài được sử dụng qua contract.

### 2.2.2 Sơ đồ thành phần

```mermaid
flowchart LR
    classDef inScope fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#0D3B1E
    classDef external fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,stroke-dasharray:5 5,color:#1F2937

    Landing[Market Landing Page] --> MBFF[Market BFF]
    Channel[Kênh Agent / Back Office] --> ABFF[Agent / BO BFF]

    subgraph SCOPE[IN SCOPE — vhm-dossier-core]
        direction TB
        API[Dossier Core API]
        DOMAIN[Dossier Domain<br/>Hồ sơ · Consent · Checklist · Pipeline]
        RELAY[Outbox & Scheduler]
        DB[(PostgreSQL<br/>dossier_db)]

        API --> DOMAIN
        DOMAIN --> DB
        DOMAIN --> RELAY
        RELAY --> DB
    end

    MBFF --> API
    ABFF --> API
    DOMAIN --> Redis[(Redis / Redisson)]
    RELAY --> Kafka[(Kafka)]
    RELAY --> Message[Message Delivery]
    API --> OCR[vhm-ocr-ekyc]
    API --> File[File Management]
    API --> Project[Market / Project]
    API --> TTOL[TTOL]

    class API,DOMAIN,RELAY,DB inScope
    class Landing,MBFF,Channel,ABFF,Redis,Kafka,Message,OCR,File,Project,TTOL external
    style SCOPE fill:#F5FBF6,stroke:#137333,stroke-width:2px
```

Các component trong khung xanh cùng nằm trong một deployable
`vhm-dossier-core`; đường biên trong hình là ranh giới logic, không phải service
hoặc deployment độc lập.

### 2.2.3 Sơ đồ kiến trúc pipeline nội bộ

Pipeline Social Housing là cấu hình có phiên bản được nạp cùng ứng dụng. Khi tạo hồ sơ, core ghi phiên bản pipeline và trạng thái khởi tạo; mỗi command được kiểm tra role, ownership và guard nghiệp vụ rồi cập nhật projection, history, reviewer và outbox trong một transaction. Không có process instance Camunda và không có ranh giới eventual consistency với workflow engine ngoài.

### 2.2.4 Phân định trách nhiệm module

`Dossier Domain` là module logic nằm bên trong `vhm-dossier-core` như thể hiện tại
mục 2.2.2; đây không phải một service hoặc deployable độc lập.

| **Khối kiến trúc** | **Trách nhiệm** | **Dữ liệu quản lý** | **Không chịu trách nhiệm** |
| --- | --- | --- | --- |
| Market/Agent BFF | Public contract, xác thực kênh, hiển thị/thu consent và truyền actor context/decision | Không sở hữu hồ sơ hoặc consent | Business invariant, đánh giá consent hợp lệ và persistence cuối. |
| Dossier Core API | Internal contract, xác thực workload/actor context, validation request và điều phối command/query | Không sở hữu aggregate riêng | Quyết định nghiệp vụ cuối và dữ liệu của dependency. |
| Dossier Domain | Vòng đời hồ sơ, consent gate/bằng chứng, validation, checklist, pipeline và phân công | Aggregate dossier và dữ liệu consent liên quan | Identity kênh, file binary, OCR raw. |
| Outbox/Scheduler | Phát sự kiện, gửi notification và nhắc SLA sau commit | Trạng thái delivery/dedup | Thay đổi quyết định nghiệp vụ đã commit. |
| `vhm-ocr-ekyc` | Nhận yêu cầu OCR CCCD đã được Domain cho phép, quản lý lifecycle và chuẩn hóa kết quả | OCR lifecycle, media reference, kết quả chuẩn | Hiển thị/thu consent, sở hữu consent gốc, cung cấp phương thức thay thế hoặc tự áp kết quả vào form. |
| Enterprise services | File, Market, TTOL, Message Delivery | Dữ liệu thuộc từng miền | Sở hữu aggregate dossier. |

### 2.2.5 Ranh giới tin cậy

| **Ranh giới** | **Mức tin cậy** | **Kiểm soát** | **Khoảng trống** |
| --- | --- | --- | --- |
| Client → BFF | Không tin cậy | Auth kênh, role, validation, rate limit tầng gateway | Thuộc phạm vi kênh/platform. |
| BFF → core | Zero Trust nội bộ | Basic Auth, HMAC, timestamp/nonce/body hash, actor signature | Cần vận hành secret rotation. |
| Actor context → business | Chỉ tin sau verify | `subject`, role, visibility, expiry, JTI | JTI replay đang có thể tắt theo môi trường. |
| Market BFF → Core | Zero Trust nội bộ | Signed actor, consent decision và idempotency | UI copy, evidence schema và OpenAPI thuộc L3. |
| Core → `vhm-ocr-ekyc` | Zero Trust nội bộ | Workload identity và object context | OpenAPI/IAM thuộc L3. |
| Core → File/Market/TTOL/Message | Dependency ngoài process | Credential server-side, timeout/retry/config | Contract/SLA và ownership file cần chốt. |
| Core → Kafka | At-least-once | Transactional outbox, retry, idempotent consumer | Publish mặc định có thể tắt theo config. |

## 2.3 Vòng đời hồ sơ

### 2.3.1 Trạng thái công bố

Pipeline Social Housing v1 sử dụng các business status chính: `DRAFT`, `SUBMITTED`, `UNDER_REVIEW`, `ADD_INFO_REQUESTED`, `APPROVED`, `REJECTED`. `APPROVED` vẫn cho phép thu hồi căn theo pipeline hiện tại; `REJECTED` là terminal. Enum tổng quát có thêm trạng thái legacy nhưng không mặc nhiên thuộc flow NOXH v1.

### 2.3.2 State machine

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> SALES_REVIEW: SUBMIT
    SALES_REVIEW --> PROCEDURE_REVIEW: APPROVE
    SALES_REVIEW --> AGENT_UPDATE_SALES: REQUEST_REVISION
    SALES_REVIEW --> REJECTED: REJECT
    AGENT_UPDATE_SALES --> SALES_REVIEW: UPDATE + RESUBMIT
    PROCEDURE_REVIEW --> SXD_REVIEW: APPROVE
    PROCEDURE_REVIEW --> SALES_REVISION_INTAKE: REQUEST_REVISION
    PROCEDURE_REVIEW --> SALES_REVIEW: RETURN_TO_SALES
    PROCEDURE_REVIEW --> REJECTED: REJECT / REVOKE_UNIT
    SALES_REVISION_INTAKE --> PROCEDURE_REVIEW: APPROVE
    SALES_REVISION_INTAKE --> AGENT_UPDATE_SALES: REQUEST_REVISION
    SALES_REVISION_INTAKE --> REJECTED: REJECT
    SXD_REVIEW --> APPROVED: APPROVE
    SXD_REVIEW --> PROCEDURE_REVISION_INTAKE: REQUEST_REVISION
    SXD_REVIEW --> REJECTED: REJECT / REVOKE_UNIT
    PROCEDURE_REVISION_INTAKE --> SXD_REVIEW: APPROVE
    PROCEDURE_REVISION_INTAKE --> SALES_REVISION_INTAKE: REQUEST_REVISION
    PROCEDURE_REVISION_INTAKE --> REJECTED: REJECT
    APPROVED --> REJECTED: REVOKE_UNIT
    REJECTED --> [*]
```

## 2.4 Tính nhất quán và Idempotency

### 2.4.1 Tạo tài nguyên và idempotency

- Nguyên tắc mục tiêu: mọi lệnh create có thể bị retry do timeout mạng phải idempotent; kênh gọi không phải là cơ sở để miễn trừ.
- Hiện trạng API: `source` mặc định `AGENT`; `MARKET` bắt buộc header `Idempotency-Key` (`10509`), còn `AGENT` chưa bắt buộc vì hành vi tương thích hiện hữu. Chưa có lý do nghiệp vụ hoặc risk acceptance được phê duyệt cho sự khác biệt này, nên đây là **deviation đang mở**, không phải baseline production.
- Trước production, Agent BFF phải gửi key và Core phải enforce như MARKET. Nếu Product giữ miễn trừ, quyết định phải nêu lý do, owner/SLA phát hiện–đối soát–xóa DRAFT mồ côi, metric/cảnh báo và risk acceptance có thời hạn theo AR-014.
- Với idempotency key, core lấy PostgreSQL transaction advisory lock trên key trước khi kiểm tra replay.
- Replay được tra theo `(idempotencyKey, createdBy)` **trước** form/schema/file validation; cùng actor nhận lại dossier đã tạo.
- Nếu key toàn cục đã thuộc actor khác, request bị từ chối; DB có unique constraint cuối trên `idempotency_key`.
- Sau pipeline initialization, entity được `flush` trước khi dựng response để `version` trả về bằng version trong DB.

### 2.4.2 Transactional outbox

Create/update/transition/delete ghi business row và `outbox_event` trong cùng transaction. Relay khóa theo batch (`FOR UPDATE SKIP LOCKED`), publish Kafka khi tính năng được bật và cập nhật trạng thái gửi. `notification_outbox` là outbox riêng cho ý định gửi thông báo; relay hiện dispatch kênh `EMAIL`, retry exponential có giới hạn và chuyển `FAILED` khi hết số lần thử.

### 2.4.3 Ranh giới nhất quán

| **Thao tác** | **Cùng transaction** | **Ngoài transaction** |
| --- | --- | --- |
| Create/update | Dossier, pipeline projection, status history, checklist, outbox | Kafka publish, downstream processing |
| Command pipeline | Dossier/status, reviewer, history, unit, outbox, notification intent | Gửi message và consumer ngoài hệ thống |
| Reminder | Claim/dedup reminder và notification intent | Message Delivery |
| OCR bất đồng bộ | Không thay đổi dossier khi tạo/poll OCR | `vhm-ocr-ekyc` quản lý media, lifecycle và kết quả chuẩn |

### 2.4.4 Concurrency guards

- `@Version` và `If-Match` ngăn lost update khi caller cung cấp version; bỏ header đồng nghĩa chấp nhận last-write-wins ở API hiện hành.
- Unique partial index ngăn race hồ sơ active trùng CCCD+dự án; conflict ánh xạ `11011`.
- Unique partial index căn được cấp ngăn hai hồ sơ active giữ cùng căn.
- Reviewer assignment PKD dùng row lock/rotation trong DB; PTT/SXD dùng atomic counter Redis và sticky assignment khi quay lại.
- Command pipeline kiểm tra current state/action trong transaction; mọi side effect chỉ commit khi toàn bộ guard thành công.

# 3. Functional Requirements

## 3.1 Ma trận năng lực chức năng

| **ID** | **Năng lực/yêu cầu** | **Thiết kế** | **Mức bắt buộc** |
| --- | --- | --- | --- |
| FR-01 | Tạo hồ sơ nháp | BFF map sang core create `SOCIAL_HOUSING`; trả `DRAFT` | `BẮT BUỘC` |
| FR-02 | Upload tài liệu | Đọc quyền dossier trước; cấp path opaque; client PUT File Management | `BẮT BUỘC` |
| FR-03 | Cập nhật snapshot | Full update ở DRAFT/ADD_INFO; contact-only khi đã submit/review | `BẮT BUỘC` |
| FR-04 | Nộp hồ sơ | Command `SUBMIT`, duplicate guard, checklist readiness, sinh mã | `BẮT BUỘC` |
| FR-05 | Chống trùng | Application precheck + database race guard | `BẮT BUỘC` |
| FR-06 | Checklist/progress | Snapshot theo template+group; counts/missing/invalid | `BẮT BUỘC` |
| FR-07 | Phê duyệt nhiều cấp | Pipeline versioned thực thi trong core | `BẮT BUỘC` |
| FR-08 | Phân công reviewer | Manual/reassign/claim + auto round-robin | `BẮT BUỘC` |
| FR-09 | Cấp/thu hồi căn | Atomic allocation/approve; unique active unit | `BẮT BUỘC` |
| FR-10 | Hồ sơ giấy | Submit/confirm hardcopy và timeline | `BẮT BUỘC` |
| FR-11 | OCR CCCD | Core gọi `vhm-ocr-ekyc`: tạo `202`, poll trạng thái/kết quả, người dùng xác nhận trước khi áp dụng | `BẮT BUỘC` |
| FR-12 | Notification/reminder | Outbox, kênh được phê duyệt, T+6/T+18 và manual trigger | `BẮT BUỘC` |
| FR-13 | Tra cứu/báo cáo | List/detail/statistics/by-contact/export | `BẮT BUỘC` |
| FR-14 | Xóa bản nháp | Chỉ DRAFT; xóa checklist và dữ liệu phụ thuộc | `BẮT BUỘC` |
| FR-15 | PIC hồ sơ | Use case, permission và audit semantics | `TBD` |
| FR-16 | Consent | Market hiển thị/thu consent; Core lưu và kiểm tra consent trước upload/submit, đồng thời xử lý rút consent | `BẮT BUỘC` |

## 3.2 Quy tắc nghiệp vụ

| **ID** | **Quy tắc** |
| --- | --- |
| BR-01 | Public create không được submit; trạng thái sau create luôn là `DRAFT`. |
| BR-02 | Create từ `MARKET` và `AGENT` phải có cơ chế replay idempotent. `MARKET` hiện đã bắt buộc `Idempotency-Key`; `AGENT` chưa bắt buộc là deviation tương thích đang mở theo AR-014, không phải miễn trừ nghiệp vụ đã được chấp thuận. |
| BR-03 | Một cặp CCCD người nộp + dự án chỉ có một hồ sơ chưa terminal. |
| BR-04 | Create `{}` không tạo checklist; create có `documents[]` và mọi full update DRAFT/ADD_INFO phải synchronize checklist. |
| BR-05 | Identity checklist là `(dossierId, documentTemplateId, groupCode)`. |
| BR-06 | Submit cần tồn tại ít nhất một required item và mọi required item ở `COMPLETE`. |
| BR-07 | Full update chỉ ở DRAFT/ADD_INFO; SUBMITTED/UNDER_REVIEW chỉ cho sửa contact email/phone. |
| BR-08 | Update không được ghi đè `source`, `assignedUnitCode` hoặc `assignedUnitId` do server quản lý. |
| BR-09 | File identity applicant/spouse và `documents[].s3PathFile` phải tồn tại khi file-validation được bật. |
| BR-10 | Không kiểm tra file path prefix theo dossier ID. |
| BR-11 | Mọi command phải hợp lệ theo state, role và ownership (`OWNER`, `CLAIMER`, `NONE`). |
| BR-12 | Lần submit đầu sinh mã `<sapId>-<agencyId>-<ddMMyy>-<sequence4>`; PKD approve chuẩn hóa thành `<sapId>-<sequence5>`. |
| BR-13 | Reject/revoke unit phải giải phóng căn đang cấp. |
| BR-14 | DRAFT delete xóa checklist tường minh; FK checklist cũng có `ON DELETE CASCADE`. |
| BR-15 | Comment hiện là optional theo pipeline/config; không mô tả là bắt buộc nếu chưa bật guard. |
| BR-16 | Core từ chối prepare-upload và submit khi consent không hợp lệ hoặc đã bị rút. |
| BR-17 | Xác nhận submit là xác nhận nộp hồ sơ, không được dùng để suy diễn consent. |

# 4. Non-Functional Requirements

Các mục tiêu `200 req/s`, P95 cụ thể và availability `99.9%` chưa có workload/capacity evidence được phê duyệt nên chưa được coi là baseline. Trước production phải chốt NFR theo workload thực tế.

| **ID** | **Nhóm** | **Yêu cầu** | **Cổng** |
| --- | --- | --- | --- |
| NFR-01 | Availability | Core stateless và có replica multi-AZ. Redis nonce/replay là dependency đồng bộ fail-closed; mọi availability target, gồm mức đề xuất `99.9%`, phải đo end-to-end và chỉ được baseline khi Redis HA/failover có bằng chứng | `BẮT BUỘC` |
| NFR-02 | Consistency | Không mất mutation đã commit; outbox cho event/notification | `BẮT BUỘC` |
| NFR-03 | Concurrency | Idempotency, optimistic lock, unique index, row/Redis lock | `BẮT BUỘC` |
| NFR-04 | Performance | P95/P99 theo endpoint và peak TPS phải được đo | TBD |
| NFR-05 | Scalability | Scale ngang API/relay; không dùng session cục bộ làm authority | `BẮT BUỘC` |
| NFR-06 | Security | Signed request/actor, least privilege, PII masking, secret rotation | `BẮT BUỘC` |
| NFR-07 | Privacy | Consent evidence, purpose, retention/deletion và audit access cho dữ liệu hồ sơ | TBD/BẮT BUỘC |
| NFR-08 | Recoverability | PostgreSQL PITR, outbox replay, backup/restore drill | TBD/BẮT BUỘC |
| NFR-09 | Observability | Metrics/log/trace/correlation và alert có owner | `BẮT BUỘC` |
| NFR-10 | Maintainability | Versioned migration/schema/pipeline; backward-compatible API | `BẮT BUỘC` |

# 5. Technology Stack & Justification

| **Công nghệ** | **Vai trò** | **Cơ sở lựa chọn/hệ quả** |
| --- | --- | --- |
| Java 25, Spring Boot 4.1 | Runtime/service framework | Stack nền tảng của dịch vụ; cần image/JVM production được chứng nhận. |
| Spring Data JPA/Hibernate | Aggregate persistence, optimistic lock | Phù hợp transaction domain; cần tránh N+1 và giữ `open-in-view=false`. |
| PostgreSQL, Liquibase | Source of truth, JSONB, constraint, advisory lock | Hỗ trợ transaction/partial index; migration phải forward-safe. |
| Redis/Redisson | Replay/nonce, counter phân công, cache/lock hỗ trợ | Không được là source of truth của dossier. |
| Kafka | Phát domain event từ outbox | At-least-once; consumer cần idempotent. |
| Caffeine | Cache cục bộ cho dữ liệu tham chiếu | Giảm latency; cần invalidation/TTL rõ ràng. |
| JSON Schema 2020-12 | Validate form Social Housing v1 | Cho phép evolution schema; enforcement hiện phụ thuộc config. |
| Thrift/HTTP clients | File, `vhm-ocr-ekyc` và tích hợp nội bộ | Contract bên ngoài cần timeout/retry/test. |
| Syncfusion/Apache POI | Export/tạo tài liệu | Cần quản lý license, template và kiểm thử output. |

## 5.1 ADR Log

ADR chi tiết nằm tại Phụ lục D. Các quyết định nền tảng: modular monolith, pipeline nội bộ bằng YAML, PostgreSQL source of truth, snapshot JSONB có schema version, transactional outbox, signed actor context, database constraint là race guard cuối.

# 6. Integration Architecture

## 6.1 Danh mục component và giao diện tích hợp

### 6.1.1 Danh sách component

| **ID** | **Component** | **Phạm vi** | **Trách nhiệm trong tích hợp** | **Authority/Dữ liệu chính** |
| --- | --- | --- | --- | --- |
| CMP-01 | Market Landing Page / Market BFF | Bên ngoài | Kênh khách hàng; xác thực, hiển thị/thu consent và map public contract | Không là authority của hồ sơ hoặc consent |
| CMP-02 | Agent / BO BFF | Bên ngoài | Kênh nghiệp vụ cho Agent/BO/Reviewer; xác thực và truyền signed actor context | Không là authority của hồ sơ |
| CMP-03 | `vhm-dossier-core` | **IN SCOPE** | Business authorization, hồ sơ, consent, checklist, pipeline, outbox và điều phối dependency | Authority của aggregate hồ sơ và consent NOXH |
| CMP-04 | PostgreSQL `dossier_db` | **IN SCOPE — owned data store** | Lưu aggregate, projection, history, consent evidence và outbox | Source of truth của Dossier Domain |
| CMP-05 | Redis / Redisson | Shared platform | Nonce/replay security, counter, cache và coordination | Không là source of truth nghiệp vụ |
| CMP-06 | Kafka | Shared platform | Phân phối domain event sau commit | Outbox/PostgreSQL là nguồn replay |
| CMP-07 | Private File Service | Bên ngoài | Presigned upload, kiểm tra tồn tại, download và ownership contract | Authority của file binary/upload grant |
| CMP-08 | Market / Project Service | Bên ngoài | Project/SAP/unit và special-day reference | Authority của master data tương ứng |
| CMP-09 | `vhm-ocr-ekyc` | Bên ngoài | Capability OCR dùng chung; quản lý media/lifecycle/kết quả OCR | Authority của tài nguyên OCR |
| CMP-10 | Message Delivery | Bên ngoài | Gửi notification từ intent đã commit | Authority của delivery outcome |
| CMP-11 | TTOL | Bên ngoài | Roster reviewer và lịch nghỉ | Authority của roster/calendar được công bố |

### 6.1.2 Sơ đồ context/integration

```mermaid
flowchart LR
    classDef inScope fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#0D3B1E
    classDef external fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,stroke-dasharray:5 5,color:#1F2937

    MARKET[Market Landing / BFF]
    AGENT[Agent / BO BFF]

    subgraph SCOPE[IN SCOPE — vhm-dossier-core]
        direction TB
        CORE[Dossier Core<br/>API · Domain · Outbox/Scheduler]
        DB[(PostgreSQL<br/>dossier_db)]
        CORE <-->|transactional data| DB
    end

    REDIS[(Redis / Redisson)]
    KAFKA[(Kafka)]
    FILE[Private File Service]
    PROJECT[Market / Project]
    OCR[vhm-ocr-ekyc]
    MSG[Message Delivery]
    TTOL[TTOL]

    MARKET -->|HTTPS sync<br/>signed actor + consent| CORE
    AGENT -->|HTTPS sync<br/>signed actor context| CORE
    CORE <-->|nonce · cache · coordination| REDIS
    CORE -.->|domain event sau commit| KAFKA
    CORE -->|HTTPS sync<br/>file metadata/reference| FILE
    CORE -->|HTTPS sync<br/>project/unit/reference| PROJECT
    CORE -->|HTTPS async resource<br/>OCR command/query| OCR
    CORE -.->|notification intent| MSG
    CORE -->|HTTPS sync/cache<br/>roster/calendar| TTOL

    class CORE,DB inScope
    class MARKET,AGENT,REDIS,KAFKA,FILE,PROJECT,OCR,MSG,TTOL external
    style SCOPE fill:#F5FBF6,stroke:#137333,stroke-width:2px
```

Sơ đồ chỉ thể hiện contract mà `vhm-dossier-core` trực tiếp sở hữu hoặc sử dụng;
topology nội bộ phía sau các capability bên ngoài không thuộc phạm vi TDD này.

### 6.1.3 Danh mục giao diện tích hợp

| **ID** | **Tích hợp** | **Hướng** | **Kiểu** | **Mục đích** | **Failure policy** |
| --- | --- | --- | --- | --- | --- |
| INT-01 | Market/Agent/BO BFF | Inbound | HTTPS sync | Registration, consent, list/detail và action | Signature/actor fail closed |
| INT-02 | Private File Service | Outbound | Client sync | Prepare upload, existence, download/presign | Fail hard cho validation bắt buộc |
| INT-03 | `vhm-ocr-ekyc` | Core outbound | HTTP async resource | OCR CCCD và kết quả chuẩn sau Domain authorization | Idempotent create, polling hữu hạn, không retry mù |
| INT-04 | Market/Project | Outbound | HTTP sync + cache | SAP/project/unit/special days | Tùy use case: fail hard hoặc best effort |
| INT-05 | TTOL | Outbound | HTTP/cache | Roster reviewer/holiday | Auto-assign best effort; manual fallback |
| INT-06 | Message Delivery | Outbound | Outbox relay | Email notification | Retry/backoff → FAILED |
| INT-07 | Kafka | Outbound | Async | Domain event đã commit | Outbox retry; publish có feature flag |
| INT-08 | PostgreSQL `dossier_db` | Internal data | SQL transaction | Aggregate, projection, history, consent và outbox | Fail request; HA/PITR theo RPO/RTO |
| INT-09 | Redis / Redisson | Shared platform | Sync | Nonce/replay, cache, counter và coordination | Security path fail closed; nghiệp vụ phụ trợ degrade theo use case |

## 6.2 Contract API hồ sơ VHM

### 6.2.1 Public registration flow

| **Bước** | **API Agent** | **Kết quả bắt buộc** |
| --- | --- | --- |
| 1 | `POST /v1/social-housing/registrations` | Tạo duy nhất một `DRAFT`, nhận `dossierId`. |
| 2 | `POST /v1/social-housing/registrations/{id}/prepare-upload` rồi PUT file | Chỉ cấp URL sau khi caller đọc được dossier; giữ `s3PathFile`. |
| 3 | `PATCH /v1/social-housing/registrations/{id}` | Ghi snapshot form/documents đầy đủ. |
| 4 | `POST /v1/social-housing/registrations/{id}/submit` | Chuyển vào pipeline nếu mọi guard đạt. |

### 6.2.2 Nhóm năng lực API nội bộ

Core công bố API nội bộ có version cho các nhóm năng lực: quản lý hồ sơ, command pipeline, quyết định tài liệu, quyền dự án, notes/hardcopy, download/export, statistics và reminder. Public contract không phản chiếu nguyên xi endpoint nội bộ; BFF chịu trách nhiệm map DTO, HTTP semantics và ẩn cấu trúc nội bộ. Danh sách path/field đầy đủ thuộc OpenAPI L3, không lặp lại trong tài liệu L2 này.

### 6.2.3 Envelope và phân trang

Core trả `ServiceResponse { code, message, data }`, trong đó `code=0` là thành công. Danh sách dùng `PageDto { items, pagination }`, page number là 1-based. HTTP status và application error code phải cùng biểu đạt một kết quả; client không được chỉ dựa vào message tiếng Việt.

## 6.3 Contract presigned upload

### 6.3.1 Tài liệu đính kèm dossier

- Request phải chứa registration ID hợp lệ dạng UUID và metadata file được allowlist.
- Định dạng hỗ trợ trong core gồm JPEG, PNG, PDF, DOC/DOCX và XLS/XLSX; extension được dẫn xuất từ content type đã chấp nhận.
- Object key có dạng `registrations/{registrationUuid}/{slug}_{randomUuid}.{ext}` nhưng không dùng prefix này làm bằng chứng ownership khi attach.
- BFF phải đọc dossier bằng actor context trước khi xin URL.
- Core chỉ cấp upload reference khi consent của hành trình NOXH còn hiệu lực.
- Sau upload, create/update kiểm tra file tồn tại khi feature bật; contract File Service phải bổ sung uploader/upload-grant owner trước production.

### 6.3.2 Media dùng cho OCR

Media gửi OCR không dùng contract prepare-upload của dossier. Kênh gọi Market BFF, BFF gọi Core; Core gọi `vhm-ocr-ekyc` theo business authorization. OCR service kiểm tra context/media rồi xin presigned URL từ File Management; upload contract được trả ngược qua Core/BFF.

Role và MIME baseline lấy từ L2 OCR/eKYC dùng chung (`OCR_DOCUMENT`, `DOCUMENT_FRONT`, `DOCUMENT_BACK`, `LABOR_CONTRACT`; JPEG/PNG/PDF). Dossier không được mở rộng role/MIME hoặc tự dựng object path nếu chưa cập nhật contract OCR L3.

## 6.4 Contract OCR dùng chung

### 6.4.1 Ranh giới tích hợp

Dossier chỉ tích hợp capability OCR qua luồng `Market Landing Page → Market BFF → vhm-dossier-core → vhm-ocr-ekyc`; Core là caller trực tiếp sau khi thực hiện business authorization. Mọi xử lý phía sau `vhm-ocr-ekyc` nằm ngoài trust boundary và contract của Dossier.

Tích hợp OCR trực tiếp từ dossier tới provider không được phép trong kiến trúc mục tiêu. Mọi đường OCR legacy phải được loại bỏ hoặc vô hiệu hóa trước go-live sau khi E2E với `vhm-ocr-ekyc` đạt quality gate.

### 6.4.2 Tạo tài nguyên OCR

Contract mục tiêu dùng `POST /ocr` với `Idempotency-Key` bắt buộc và trả HTTP `202` cùng `Retry-After`. Context tối thiểu:

| **Trường** | **Ý nghĩa/kiểm soát** |
| --- | --- |
| `source` | `DOSSIER`; dùng cho authorization, quota và idempotency scope. |
| `referenceId` | Dossier ID dạng opaque business reference. |
| `requestBy` | Opaque actor reference, không nhúng PII. |
| `subjectRef` | Opaque applicant/customer reference. |
| `channel`, `platform` | Context kênh đã allowlist. |
| `documentType`/loại OCR | `NATIONAL_ID` hoặc contract CCCD hai mặt được đóng băng ở OpenAPI L3. |
| `s3PathFile`/media roles | Chỉ reference do service chấp nhận; không gửi presigned URL vào DB/event. |

Cùng idempotency key và cùng request trả tài nguyên hiện hữu; cùng key nhưng request khác trả `409`. Provider không phải tham số do dossier/client lựa chọn.

### 6.4.3 Vòng đời và áp dụng kết quả

Trạng thái công bố cho Domain gồm `QUEUED`, `PROCESSING`, `COMPLETED`, `FAILED`, `EXPIRED`. Core theo dõi kết quả và chiếu trạng thái cần thiết qua BFF; `nextAction` hướng dẫn `POLL`, `RETRY` hoặc `CONFIRM_AND_APPLY`.

Khi `COMPLETED`, kênh phải cho người dùng kiểm tra kết quả chuẩn trước khi áp dụng. Việc áp dụng chỉ diễn ra qua PATCH snapshot dossier bình thường; OCR service không ghi trực tiếp vào database dossier. Dossier lưu bằng chứng tối thiểu gồm `ocrId`, outcome chuẩn, thời điểm và actor xác nhận để trace/readiness; không lưu raw provider response hoặc provider job ID. Field/schema cụ thể thuộc Form/Checklist Contract L3.

### 6.4.4 Tính nhất quán và failure semantics

- `vhm-ocr-ekyc` tạo request, media refs và outbox trong một transaction rồi mới trả `202`.
- Kafka chỉ chứa OCR ID tối thiểu; worker idempotent và trạng thái terminal bất biến.
- Core không retry mù khi kết quả submit không rõ; dùng cùng idempotency key hoặc đối soát theo `ocrId`.
- OCR thất bại/timeout không tự động biến hồ sơ thành `REJECTED`; người dùng xử lý theo UX/policy đã được duyệt.
- Các giới hạn MIME, size, checksum, retention và deadline dùng đúng baseline của tài liệu L2 `vhm-ocr-ekyc`; không định nghĩa lại khác trong dossier.

## 6.5 Contract pipeline

### 6.5.1 Action contract

Các action chính gồm `UPDATE`, `SUBMIT`, `RESUBMIT`, `ASSIGN`, `REASSIGN`, `CLAIM`, `ALLOCATE_UNIT`, `APPROVE`, `REJECT`, `REQUEST_REVISION`, `RETURN_TO_SALES`, `SUBMIT_HARDCOPY`, `CONFIRM_HARDCOPY`, `REVOKE_UNIT`. Tập action khả dụng phải lấy từ response `availableActions`, không hard-code theo status ở kênh.

### 6.5.2 Role và ownership

Pipeline kiểm tra cả role (`APPLICANT_AGENT`, `PKD`, `PKD_LEAD`, `PTT`, `PTT_LEAD`) và ownership:

- `OWNER`: actor tạo hồ sơ.
- `CLAIMER`: reviewer đang claim stage.
- `NONE`: không yêu cầu ownership nhưng vẫn yêu cầu role.

### 6.5.3 Pipeline selection

Tại create, pipeline phải được xác định bằng pipeline ID/version authoritative từ contract nghiệp vụ hoặc một rule selection duy nhất. Trường hợp không tìm thấy hoặc có nhiều kết quả phải bị từ chối bằng lỗi cấu hình rõ ràng; không chọn theo thứ tự khai báo.

## 6.6 Mô hình checklist chuẩn hóa

| **Thuộc tính** | **Giá trị/ngữ nghĩa** |
| --- | --- |
| Identity | `dossierId + documentTemplateId + groupCode` |
| Upload status | `NOT_STARTED`, `COMPLETE` |
| OCR result | `NOT_OCR`, `MATCH`, `MISMATCH`, `INSUFFICIENT_DATA`, `FAILED` |
| Review status | `NOT_REVIEWED`, `VALID`, `INVALID` |
| Completed required | Upload `COMPLETE` và OCR khác `FAILED` |
| Invalid item | OCR `FAILED` hoặc review/constraint tương ứng không đạt |
| Progress | `completedRequired / requiredCount`, làm tròn 2 chữ số |

Khi file không đổi, synchronization giữ OCR/review state; khi path đổi hoặc bị xóa, trạng thái được reset; item không còn trong snapshot bị xóa. Duplicate `documentTemplateId + groupCode` trong cùng request bị từ chối.

## 6.7 Contract lỗi chuẩn

| **Code** | **Ý nghĩa** | **HTTP kỳ vọng** |
| --- | --- | --- |
| `10509` | Thiếu header bắt buộc, gồm idempotency cho MARKET | 400 |
| `11003` | Không tìm thấy dossier | 400/404 theo public mapping |
| `11005` | Dossier không ở trạng thái cho phép sửa | 400 |
| `11006` | Optimistic version conflict | 409 nên được chuẩn hóa ở public API |
| `11010` | Dossier không cho phép xóa | 400 |
| `11011` | Hồ sơ active trùng CCCD+dự án | 409 |
| `11017` | Không có checklist required để submit | 400/422 |
| `11018` | Thiếu required document | 400/422 |

# 7. Data Architecture & Data Flow

## 7.1 Data Model

### 7.1.1 Sở hữu dữ liệu logic

| **Bảng/aggregate** | **Mục đích** | **Invariant chính** |
| --- | --- | --- |
| `dossier` | Aggregate hồ sơ, JSONB form/metadata, source, PIC, pipeline projection, version | UUIDv7; source AGENT/MARKET; active duplicate/unit uniqueness. |
| `dossier_status_history` | Lịch sử trạng thái/action | Tạo trong cùng transaction với transition. |
| `dossier_checklist` | Projection readiness/progress và OCR evidence tối thiểu | PK logic template+group; FK cascade; không chứa raw provider result. |
| `dossier_stage_reviewer` | Người xử lý theo stage | Assignment/claim/review/decision metadata. |
| `dossier_reminder_sent` | Dedup reminder theo cycle | Dossier/state/rule/cycle unique. |
| `agent_project_permission` | ACL team/project/scope và rotation | Chỉ một permission active theo key. |
| `outbox_event` | Domain event chưa/đã publish | Không mất event khi business commit. |
| `notification_outbox` | Ý định notification và retry | Dedupe key/attempt/status. |
| `dossier_note` | Ghi chú general/hardcopy | Soft-delete theo use case. |
| `audit_log` | Nền tảng audit cho PIC/operation | Table có nhưng writer/use case PIC chưa hoàn chỉnh. |

### 7.1.2 Sơ đồ quan hệ dữ liệu logic

```mermaid
erDiagram
    DOSSIER ||--o{ DOSSIER_STATUS_HISTORY : has
    DOSSIER ||--o{ DOSSIER_CHECKLIST : projects
    DOSSIER ||--o{ DOSSIER_STAGE_REVIEWER : assigns
    DOSSIER ||--o{ DOSSIER_REMINDER_SENT : deduplicates
    DOSSIER ||--o{ DOSSIER_NOTE : contains
    DOSSIER ||--o{ OUTBOX_EVENT : emits
    DOSSIER ||--o{ NOTIFICATION_OUTBOX : enqueues
    AGENT_PROJECT_PERMISSION }o--|| PROJECT_REFERENCE : scopes
```

### 7.1.3 Database invariants

- Partial unique index trên normalized JSON path `applicant.idNumber + projectRegistration.projectId` cho dossier chưa terminal.
- Partial unique index trên unit được cấp cho dossier active.
- Check constraint cho `source`; MARKET yêu cầu idempotency key.
- Checklist có constraint enum và `invalid_reason` phù hợp trạng thái.
- Optimistic `version` được JPA quản lý; response create được dựng sau flush.
- Không có FK đến user/PIC/project master vì các định danh này thuộc hệ thống ngoài.

## 7.2 Data Flow Diagram

### 7.2.1 Luồng đăng ký và nộp hồ sơ

```mermaid
flowchart TB
    classDef process fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#0D3B1E
    classDef external fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,stroke-dasharray:5 5,color:#1F2937
    classDef store fill:#E8F0FE,stroke:#1A73E8,stroke-width:2px,color:#102A43

    CUSTOMER([Khách hàng / Đại lý])
    BFF[Market / Agent BFF]

    subgraph SCOPE[IN SCOPE — vhm-dossier-core]
        direction LR
        P1[1.0 Consent & Registration]
        P2[2.0 Upload & Snapshot]
        P3[3.0 Submit & Pipeline]
        P4[4.0 Outbox / Notification Relay]
        D1[(D1 · PostgreSQL dossier_db)]

        P1 <-->|DRAFT + consent evidence| D1
        P2 <-->|form snapshot + checklist| D1
        P3 <-->|state + history + outbox| D1
        D1 -->|event/notification intent đã commit| P4
    end

    FILE[Private File Service]
    STORAGE[(Private Object Storage)]
    PROJECT[Market / Project]
    KAFKA[(Kafka)]
    MESSAGE[Message Delivery]

    CUSTOMER ==>|form · consent · file metadata| BFF
    BFF ==>|signed command/query| P1
    BFF ==>|prepare-upload · update| P2
    BFF -->|submit| P3
    P1 -->|dossierId · consent state| BFF
    P2 <-->|upload reference · existence| FILE
    FILE -.->|quản lý object| STORAGE
    CUSTOMER ==>|PUT file bytes bằng presigned URL| STORAGE
    P3 <-->|project/unit reference| PROJECT
    P3 -->|status · available actions| BFF
    P4 -.->|domain event, opaque ID| KAFKA
    P4 -.->|notification metadata| MESSAGE

    class P1,P2,P3,P4 process
    class D1 store
    class CUSTOMER,BFF,FILE,STORAGE,PROJECT,KAFKA,MESSAGE external
    style SCOPE fill:#F5FBF6,stroke:#137333,stroke-width:2px
```

Ký hiệu `==>` biểu diễn luồng có dữ liệu hồ sơ hoặc file nhạy cảm; Kafka và
Message Delivery chỉ nhận metadata/opaque ID theo allowlist.

### 7.2.2 Luồng phê duyệt

```mermaid
flowchart LR
    classDef process fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#0D3B1E
    classDef external fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,stroke-dasharray:5 5,color:#1F2937
    classDef store fill:#E8F0FE,stroke:#1A73E8,stroke-width:2px,color:#102A43

    REVIEWER([PKD / PTT / BO Reviewer])
    BFF[Agent / BO BFF]

    subgraph SCOPE[IN SCOPE — vhm-dossier-core]
        P1[5.0 Authorization & Pipeline Command]
        P2[6.0 Assignment / Unit / Review]
        P3[7.0 Outbox & Reminder]
        D1[(D1 · PostgreSQL dossier_db)]

        P1 <-->|dossier state · version · history| D1
        P1 -->|validated transition| P2
        P2 <-->|reviewer · decision · unit| D1
        D1 -->|committed intent| P3
    end

    TTOL[TTOL / Roster]
    PROJECT[Market / Project]
    MESSAGE[Message Delivery]
    KAFKA[(Kafka)]

    REVIEWER ==>|action + payload| BFF
    BFF -->|signed actor context| P1
    P2 <-->|roster / holiday| TTOL
    P2 <-->|project / unit reference| PROJECT
    P1 -->|new state / available actions| BFF
    P3 -.->|notification metadata| MESSAGE
    P3 -.->|domain event, opaque ID| KAFKA

    class P1,P2,P3 process
    class D1 store
    class REVIEWER,BFF,TTOL,PROJECT,MESSAGE,KAFKA external
    style SCOPE fill:#F5FBF6,stroke:#137333,stroke-width:2px
```

### 7.2.3 Luồng OCR CCCD qua capability dùng chung

```mermaid
flowchart LR
    classDef process fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#0D3B1E
    classDef external fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,stroke-dasharray:5 5,color:#1F2937
    classDef store fill:#E8F0FE,stroke:#1A73E8,stroke-width:2px,color:#102A43

    USER([Khách hàng / Người dùng])
    BFF[Market / Agent BFF]

    subgraph SCOPE[IN SCOPE — vhm-dossier-core]
        P1[8.0 OCR Authorization & Projection]
        D1[(D1 · PostgreSQL dossier_db)]
        P1 <-->|OCR reference · confirmed fields| D1
    end

    OCR[vhm-ocr-ekyc]
    FILE[File Management / Private Storage]

    USER ==>|media metadata · confirmation| BFF
    BFF -->|request theo hồ sơ| P1
    P1 <-->|OCR command/status + opaque context| OCR
    OCR ==>|media access theo contract| FILE
    P1 -->|canonical result cần xác nhận| BFF

    class P1 process
    class D1 store
    class USER,BFF,OCR,FILE external
    style SCOPE fill:#F5FBF6,stroke:#137333,stroke-width:2px
```

Không có đường ghi trực tiếp từ OCR service vào PostgreSQL của dossier. Ranh giới xác nhận giúp OCR không tự trở thành quyết định nghiệp vụ và giữ optimistic version/duplicate/file guards tại dossier core.

## 7.3 Data Privacy & PII

### 7.3.1 Phân loại và tối thiểu hóa

- CCCD, ngày sinh, địa chỉ, phone/email là PII; ảnh CCCD là dữ liệu nhạy cảm.
- Kafka/outbox/event/log chỉ nên chứa identifier và metadata tối thiểu, không chứa raw binary, presigned URL còn hiệu lực hoặc full formData.
- `idNumber`, `phone`, `email` không được dùng làm metric label hoặc correlation ID.
- PKD/PKD_LEAD nhận dữ liệu liên hệ đã mask theo response mapping hiện hành.
- Export/download cần cùng visibility/ownership guard như detail.

### 7.3.2 Danh mục dữ liệu và yêu cầu quản lý

Retention, legal hold, purge, quyền của data subject và phạm vi audit chưa có baseline được phê duyệt; đây là điều kiện production. Xóa DRAFT là hard delete nghiệp vụ, không thay thế chính sách xóa PII cho hồ sơ đã submit/terminal và các bản sao backup/outbox/log.

### 7.3.3 Nguyên tắc quản lý consent

- Market Landing/BFF hiển thị notice và thu lựa chọn; `vhm-dossier-core` lưu, kiểm tra hiệu lực và xử lý rút consent. `vhm-ocr-ekyc` không sở hữu consent gốc.
- Core kiểm tra consent tại prepare-upload và submit của hành trình khách hàng NOXH; consent không hợp lệ hoặc đã bị rút thì hai thao tác này bị từ chối.
- Nội dung notice, UI, mô hình bằng chứng và withdrawal workflow thuộc artefact L3 do Product/Market/Privacy/Legal phê duyệt; TDD này không định nghĩa thay.

## 7.4 Data Stores & Ownership

| **Store** | **Authority** | **Failure impact** | **Phục hồi** |
| --- | --- | --- | --- |
| PostgreSQL `dossier_db` | Dossier/checklist/pipeline/history/outbox | Core không thể mutate an toàn | PITR/backup/restore TBD |
| Redis | Nonce/replay, counter, cache/coordination | Mất nonce/replay chặn toàn bộ request BFF → Core khi security required; counter/cache chỉ làm suy giảm chức năng phụ trợ | Security path cần HA multi-AZ/automatic failover; cache rebuild và assignment có manual path |
| Kafka | Distribution của event đã commit | Downstream trễ; business data không mất do outbox | Replay relay/topic retention TBD |
| Private File Store | Binary tài liệu | Upload/OCR/download không hoạt động | Do File Service quản lý |

# 8. Business Flow Diagrams

## 8.1 Sequence/State Diagram

### 8.1.1 Create DRAFT và replay

```mermaid
sequenceDiagram
    autonumber
    actor User as Khách hàng / Đại lý
    participant BFF as Market / Agent BFF
    participant API as Dossier Core API
    participant Domain as Dossier Domain
    participant DB as PostgreSQL dossier_db

    User->>BFF: Yêu cầu tạo hồ sơ
    BFF->>API: Create DRAFT + signed actor context<br/>+ Idempotency-Key theo BR-02
    API->>API: Xác thực actor, source và key
    alt Thiếu key tại kênh bắt buộc
        API-->>BFF: 10509
    else Request hợp lệ
        API->>DB: Advisory lock theo key (nếu có)
        API->>DB: Tìm replay theo key + actor
        alt Đã có kết quả cùng actor
            DB-->>API: Dossier hiện hữu
            API-->>BFF: Replay cùng dossierId/version
        else Key đã thuộc actor khác
            API-->>BFF: Từ chối xung đột key
        else Request mới
            API->>Domain: Validate cấu trúc/schema/file/duplicate
            Domain->>DB: BEGIN
            Domain->>DB: Ghi DRAFT + pipeline + history<br/>+ checklist + outbox
            Domain->>DB: Flush và COMMIT
            DB-->>Domain: dossierId + version
            Domain-->>API: DRAFT đã tạo
            API-->>BFF: dossierId + version
        end
    end
    BFF-->>User: Kết quả tạo hồ sơ
```

Unique constraint tại DB là race guard cuối; conflict được map về lỗi nghiệp vụ và
không để lại một phần aggregate.

### 8.1.2 Thu consent và upload tài liệu

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Khách hàng
    participant Market as Market Landing / BFF
    participant API as Dossier Core API
    participant Domain as Dossier Domain
    participant DB as PostgreSQL dossier_db
    participant File as File Management
    participant Store as Private Object Storage

    Market-->>Customer: Hiển thị notice và lựa chọn consent
    Customer->>Market: Xác nhận lựa chọn
    Market->>API: Ghi consent evidence<br/>(subject, scope, notice version, decision, timestamp)
    API->>Domain: Validate subject/scope/version
    Domain->>DB: Lưu consent evidence
    DB-->>Domain: Consent state hiện hành
    Domain-->>Market: Đã ghi nhận

    Customer->>Market: Chọn tài liệu để upload
    Market->>API: Prepare upload theo dossier
    API->>Domain: Kiểm tra visibility + consent hiện hành
    Domain->>DB: Đọc dossier và consent
    alt Consent thiếu, không hợp lệ hoặc đã rút
        Domain-->>Market: Từ chối prepare-upload
        Market-->>Customer: Yêu cầu xử lý consent
    else Consent hợp lệ
        Domain->>File: Yêu cầu upload reference theo dossier/actor
        File-->>Domain: Opaque path + presigned URL ngắn hạn
        Domain-->>Market: Upload reference + presigned URL
        Market-->>Customer: Cho phép upload
        Customer->>Store: PUT file trực tiếp bằng presigned URL
        Store-->>Customer: Kết quả upload
    end
```

Xác nhận submit không thay thế consent; Core luôn dùng trạng thái consent đã lưu
làm nguồn thẩm quyền cho gate upload và submit.

### 8.1.3 Update snapshot

```mermaid
sequenceDiagram
    autonumber
    actor User as Khách hàng / Đại lý
    participant BFF as Market / Agent BFF
    participant API as Dossier Core API
    participant Domain as Dossier Domain
    participant File as File Management
    participant DB as PostgreSQL dossier_db

    User->>BFF: Cập nhật hồ sơ + version
    BFF->>API: Update snapshot + If-Match
    API->>Domain: Kiểm tra visibility, state và version
    Domain->>DB: Đọc dossier hiện hành
    alt Version không khớp
        Domain-->>BFF: 11006
    else DRAFT / ADD_INFO
        Domain->>File: Xác minh file reference
        File-->>Domain: Trạng thái tồn tại/hợp lệ
        Domain->>Domain: Validate snapshot + duplicate<br/>giữ field do server sở hữu
        Domain->>DB: Ghi snapshot + checklist + history + outbox
        DB-->>Domain: Version mới
        Domain-->>BFF: Hồ sơ đã cập nhật
    else SUBMITTED / UNDER_REVIEW
        Domain->>Domain: Chỉ cho phép contact email/phone
        alt Có field ngoài allowlist
            Domain-->>BFF: Từ chối cập nhật
        else Chỉ thay đổi contact
            Domain->>DB: Ghi contact + history + outbox
            DB-->>Domain: Version mới
            Domain-->>BFF: Hồ sơ đã cập nhật
        end
    end
    BFF-->>User: Kết quả cập nhật
```

### 8.1.4 Submit và sinh mã

```mermaid
sequenceDiagram
    autonumber
    actor User as Khách hàng / Đại lý
    participant BFF as Market / Agent BFF
    participant API as Dossier Core API
    participant Domain as Dossier Domain
    participant Project as Market / Project
    participant DB as PostgreSQL dossier_db

    User->>BFF: Xác nhận nộp hồ sơ
    BFF->>API: Command SUBMIT + signed actor context
    API->>Domain: Authorize owner/action/state
    Domain->>DB: Đọc DRAFT + consent + checklist
    alt Consent không còn hợp lệ
        Domain-->>BFF: Từ chối submit
    else Checklist thiếu hoặc chưa COMPLETE
        Domain-->>BFF: 11017 / 11018
    else Guard hợp lệ
        Domain->>Project: Lấy project/SAP context authoritative
        Project-->>Domain: Project/SAP reference
        Domain->>DB: BEGIN + khóa/kiểm tra duplicate active
        Domain->>DB: Sinh mã lần đầu + transition Sales review<br/>+ history + assignment best effort + outbox
        Domain->>DB: COMMIT
        DB-->>Domain: Mã hồ sơ + state/version mới
        Domain-->>API: Submit thành công
        API-->>BFF: Mã hồ sơ + available actions
        BFF-->>User: Hồ sơ đã được nộp
    end
```

### 8.1.5 Phê duyệt, yêu cầu bổ sung và reminder

```mermaid
sequenceDiagram
    autonumber
    actor Reviewer as PKD / PTT / BO Reviewer
    participant BFF as Agent / BO BFF
    participant API as Dossier Core API
    participant Domain as Dossier Domain
    participant DB as PostgreSQL dossier_db
    participant Scheduler as Reminder Scanner
    participant Calendar as Market / TTOL
    participant Message as Message Delivery

    Reviewer->>BFF: Approve / Reject / Request revision
    BFF->>API: Pipeline command + actor context
    API->>Domain: Validate role, ownership, state, action
    Domain->>DB: BEGIN + lock current state/version
    Domain->>DB: Ghi transition + reviewer/unit/history<br/>+ notification intent + outbox
    Domain->>DB: COMMIT
    DB-->>Domain: State/version mới
    Domain-->>BFF: Kết quả + available actions
    BFF-->>Reviewer: Trạng thái mới

    loop Fixed-delay scan theo cycle
        Scheduler->>DB: Claim hồ sơ cần nhắc, chống trùng
        Scheduler->>Calendar: Tính ngày làm việc/ngày nghỉ
        alt Đến mốc T+6 hoặc T+18
            Scheduler->>DB: Ghi notification intent
            Scheduler->>Message: Relay notification metadata
            Message-->>Scheduler: Thành công / retryable error
        else Chưa đến hạn hoặc dependency lỗi
            Scheduler->>Scheduler: Bỏ qua hoặc retry theo runbook
        end
    end
```

Reminder không tự động reject hồ sơ. Rule hiện dùng các mốc 144/216 giờ và
432/504 giờ, loại trừ ngày nghỉ; manual trigger chỉ dành cho `PKD/PKD_LEAD`.

## 8.2 Ma trận xử lý lỗi

| **Tình huống** | **Hành vi** | **Retry** | **Tính nhất quán** |
| --- | --- | --- | --- |
| File không tồn tại | Reject create/full update | Sau khi upload đúng | Không ghi dossier/update |
| Duplicate precheck | Trả `11011` | Không, trừ khi hồ sơ cũ terminal | Không ghi |
| Duplicate race tại unique index | Map `11011` | Như trên | Transaction rollback |
| Version mismatch | Trả `11006` | Caller reload rồi quyết định | Không ghi |
| Checklist thiếu | `11017/11018` | Upload/sync rồi submit lại | Dossier giữ nguyên state |
| External reviewer roster lỗi | Bỏ auto-assign, cho manual path | Best effort | Transition có thể vẫn commit theo action |
| Kafka down | Outbox giữ pending | Relay retry | Business commit không mất |
| Message Delivery lỗi | Notification outbox retry/backoff | Hữu hạn rồi FAILED | Transition không rollback |
| OCR `FAILED`/`EXPIRED`/timeout | Kết thúc polling, hiển thị retry/manual review | Theo `nextAction` và cùng idempotency policy | Không tự sửa/reject dossier |
| Redis security nonce lỗi khi required | Fail closed | Theo runbook | Không xử lý request |

## 8.3 Chuẩn hóa dữ liệu

- Chuẩn hóa CCCD/project ID trước duplicate lookup; không coi chuỗi trắng là identity hợp lệ.
- XSS sanitizer áp dụng cho form/metadata và dữ liệu đưa vào tài liệu/thông báo.
- Server giữ quyền sở hữu `source`, `assignedUnitCode`, `assignedUnitId`, pipeline projection và audit fields.
- `documents[].documentTemplateId` phải là UUID hợp lệ; `groupCode` được chuẩn hóa rỗng cho legacy.
- Timestamps lưu theo kiểu thời gian thống nhất của persistence; API phải biểu diễn ISO-8601 có timezone.

# 9. Security & Compliance Architecture

## 9.1 Identity & Authentication

Public client xác thực tại BFF. Mọi request từ BFF vào `/internal/**` của core phải có hai lớp danh tính độc lập:

1. **Workload identity**: Basic Auth kết hợp HMAC request signature trên client ID, timestamp, nonce, body SHA-256 và signature.
2. **Business actor**: actor context được ký riêng, chứa `subject`, display name, pipeline roles, visibility, thời hạn và JTI.

Ở STAG/PROD, chữ ký nội bộ và actor context là bắt buộc. Timestamp giới hạn replay window; nonce được kiểm tra qua Redis. Basic username phải khớp client ID. Local bypass chỉ hợp lệ khi active profile chính xác là `local`; bypass không được xuất hiện ở STAG/PROD.

## 9.2 Authorization & Access Control

Authorization áp dụng defense in depth:

| **Lớp** | **Kiểm soát** |
| --- | --- |
| Kênh/BFF | Xác thực end-user, scope public API và object-level authorization. |
| Visibility core | `ALL`, `TEAM`, `SELF_CREATED`, `ASSIGNED`; các mode chưa hỗ trợ phải deny by default. |
| Project permission | Team/project/scope active quyết định phạm vi đại lý và nguồn auto-assignment. |
| Pipeline role | Mỗi action chỉ dành cho role đã cấu hình. |
| Ownership | `OWNER` hoặc `CLAIMER` khi action yêu cầu. |
| Dữ liệu | Mask phone/email theo persona; download/export dùng cùng phạm vi với detail. |
| Consent gate | Core kiểm tra consent theo subject/scope trước prepare-upload và submit; giá trị do client tự khai không phải nguồn thẩm quyền. |

`REGION` và `DEPARTMENT` hiện không có semantics đầy đủ trong core nên bị từ chối, không được mở rộng bằng fallback `ALL`. Lịch sử assignment có thể được dùng cho read visibility nếu feature tương ứng được bật và phê duyệt.

## 9.3 Secrets & Credential Management

- HMAC secret, Basic credential, File/OCR/Market/Message/TTOL credential và encryption key không được nằm trong source, image hoặc tài liệu này.
- Secret phải được cấp qua secret manager/runtime, có owner, rotation period và emergency revocation runbook.
- Core và `vhm-ocr-ekyc` dùng danh tính workload riêng, audience/scope tối thiểu; dossier không nhận provider credential OCR.
- Không copy cookie STG vào code/config/log; credential từng được chia sẻ ngoài luồng phải được rotate/revoke.
- Startup/readiness phải fail khi capability bắt buộc được bật nhưng thiếu secret/config hợp lệ.

## 9.4 Application Security & Data Protection

### Kiểm soát request và file

- Body bị từ chối nếu chứa các trường giả mạo actor như `actorId`, `actorDisplayName`, `roles` ở bất kỳ cấp lồng nhau.
- Structural guard giới hạn shape/kích thước; JSON Schema bảo vệ semantic khi feature được bật; XSS sanitizer chạy trước persistence/render.
- File type/size/magic/checksum phải được kiểm soát ở File/OCR contract; chỉ path opaque được lưu.
- Presigned URL có TTL ngắn, phạm vi object chính xác, không cho list và không được persist/log.
- File existence không đồng nghĩa ownership; attach authorization chỉ được coi là hoàn chỉnh khi có upload-grant/owner contract.

### Ma trận bảo vệ dữ liệu

| **Trạng thái dữ liệu** | **Kiểm soát bắt buộc** |
| --- | --- |
| In transit | TLS; HMAC/mTLS/JWT theo ranh giới; không gửi credential trong query. |
| At rest | PostgreSQL/object storage encryption theo chuẩn VHM; backup cũng phải mã hóa. |
| In use | Least privilege, không dump formData/raw OCR vào log/APM; giới hạn export/download. |
| Event | Chỉ opaque ID và metadata tối thiểu; không PII, media path hoặc presigned URL. |
| Retention/deletion | Policy theo purpose/legal hold; purge cả primary, object, outbox đã hết hạn và backup theo lịch. |
| Consent evidence | Giữ nội dung/version và quyết định có thể kiểm chứng; tách khỏi debug log và không chứa media/raw OCR. |

### Nhật ký kỹ thuật

Log tối thiểu gồm correlation ID, client ID, actor subject dạng opaque, dossier ID, action, kết quả, duration và error code. Không log request/response body chứa PII, CCCD, contact, file URL, HMAC/actor token hoặc OCR result. Audit quyết định nghiệp vụ phải tách khỏi debug log và có quyền truy cập/retention riêng.

### Mô hình mối đe dọa

| **ID** | **Mối đe dọa** | **Kiểm soát** | **Tồn dư** |
| --- | --- | --- | --- |
| TH-01 | IDOR đọc/sửa dossier ngoài phạm vi | Signed actor, visibility, project ACL, ownership | Cần E2E negative matrix. |
| TH-02 | Giả mạo request nội bộ/replay | HMAC body hash, timestamp, nonce, Redis | Rotation/HA Redis cần runbook. |
| TH-03 | Race tạo hồ sơ/cấp căn trùng | Advisory lock, optimistic lock, partial unique index | Cần load/concurrency test. |
| TH-04 | Attach file của actor khác | Access-before-upload + existence check | Chưa có owner/upload-grant; rủi ro cao. |
| TH-05 | Client tự khai required checklist | Có thể làm sai submit readiness nếu server tin snapshot client | Checklist authority và version server-side. |
| TH-06 | PII/secret lọt log/event | Allowlist log/event, scan CI/APM, runbook incident | Cần evidence production. |
| TH-07 | OCR result tự động gây quyết định sai | Người dùng/người có thẩm quyền xác nhận trước khi áp dụng; không auto reject | UX/contract cần UAT. |
| TH-08 | Bỏ qua consent hoặc dùng consent sai subject/scope | Consent gate tại Core và negative E2E | Evidence contract thuộc L3. |

# 10. Deployment & Infrastructure Topology

## 10.1 Environments

| **Môi trường** | **Mục đích** | **Đặc điểm/điều kiện** |
| --- | --- | --- |
| Local | Phát triển và E2E cục bộ | Core và dependency cục bộ; không dùng dữ liệu/credential thật. |
| STAG | Contract/UAT/integration | Security nội bộ bật như PROD; external sandbox; migration rehearsal; synthetic/masked data. |
| PROD | Nghiệp vụ thật | HA, secrets runtime, signature/actor required, file validation, outbox relay, alert/runbook và policy privacy đã duyệt. |

## 10.2 Production Deployment Diagram (CI/CD)

### 10.2.1 CI/CD và promotion artefact

```mermaid
flowchart LR
    Dev([Developer]) -->|Merge request| Git[GitLab repository]
    Git --> Runner[GitLab Runner<br/>short-lived workload identity]
    Runner --> Gate[Build · Unit/Integration/Contract Test<br/>SAST/SCA · Secret/Image/IaC Scan]
    Gate --> Registry[(Immutable Image Registry<br/>image digest)]
    Gate --> Evidence[(Artefact Store<br/>manifest · SBOM · test evidence)]
    Registry --> CD[Approved CD Pipeline]
    Evidence --> CD
    CD --> Migration[One-shot Liquibase Job]
    CD --> Deploy[Deploy manifest by image digest]
```

CI build một lần; CD quảng bá đúng image digest và manifest đã được phê duyệt,
không build lại giữa các môi trường. Credential pipeline dùng workload identity
ngắn hạn, không nằm trong source, image hoặc artefact.

### 10.2.2 Production runtime topology

```mermaid
flowchart TB
    classDef external fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,stroke-dasharray:5 5,color:#1F2937
    classDef app fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#0D3B1E
    classDef data fill:#E8F0FE,stroke:#1A73E8,stroke-width:2px,color:#102A43
    classDef platform fill:#FFF4E5,stroke:#B06000,stroke-width:1.5px,color:#4A2A00

    BFF[Market / Agent BFF]
    EXT[File · Market/Project · Message · TTOL<br/>vhm-ocr-ekyc]

    subgraph LZ[AWS Production Landing Zone — account/region TBD]
        direction TB
        subgraph VPC[Application VPC — CIDR TBD]
            direction TB
            INGRESS[Internal Load Balancer / Ingress<br/>private endpoint]
            SERVICE[EKS Service<br/>namespace vhm-dossier]

            subgraph AZA[Availability Zone A]
                direction TB
                subgraph APPA[Private application subnet A]
                    COREA[Dossier Core replica A<br/>>= 1 pod]
                end
                subgraph DATAA[Private data subnet A]
                    PGA[(PostgreSQL primary)]
                    REDISA[(Redis HA member A)]
                end
            end

            subgraph AZB[Availability Zone B]
                direction TB
                subgraph APPB[Private application subnet B]
                    COREB[Dossier Core replica B<br/>>= 1 pod]
                end
                subgraph DATAB[Private data subnet B]
                    PGB[(PostgreSQL standby)]
                    REDISB[(Redis HA member B / quorum)]
                end
            end

            DBEP[PostgreSQL HA endpoint]
            REDISEP[Redis HA endpoint<br/>automatic failover · mode TBD L3]
            KAFKA[(Kafka replicated multi-AZ)]
            EGRESS[Controlled egress / private endpoints]
            VAULT[Vault / Secret Manager]
            OBS[Metrics · Logs · Traces]

            INGRESS --> SERVICE
            SERVICE --> COREA
            SERVICE --> COREB
            COREA --> DBEP
            COREB --> DBEP
            DBEP --> PGA
            DBEP --> PGB
            PGA -.->|replication / failover| PGB
            COREA --> REDISEP
            COREB --> REDISEP
            REDISEP --> REDISA
            REDISEP --> REDISB
            COREA --> KAFKA
            COREB --> KAFKA
            COREA --> EGRESS
            COREB --> EGRESS
            VAULT -.-> COREA
            VAULT -.-> COREB
            COREA -.-> OBS
            COREB -.-> OBS
        end
    end

    BFF -->|TLS + signed workload/actor context| INGRESS
    EGRESS -->|TLS · destination allowlist| EXT

    class BFF,EXT external
    class COREA,COREB app
    class PGA,PGB,REDISA,REDISB,KAFKA data
    class INGRESS,SERVICE,DBEP,REDISEP,EGRESS,VAULT,OBS platform
    style LZ fill:#FAFAFA,stroke:#374151,stroke-width:2px
    style VPC fill:#FFFFFF,stroke:#64748B,stroke-width:1.5px
```

Baseline availability của L2 là tối thiểu hai Core replica phân bố trên ít nhất hai
AZ; PostgreSQL, Redis và Kafka phải có HA/replication multi-AZ. Account, region,
CIDR, node group, resource request/limit, số replica theo tải, Redis
Sentinel/Cluster/managed mode và failover SLO là quyết định L3 của Platform/SRE;
không được để `TBD` sau cổng OAT.

## 10.3 Deployment Strategy

- Build artifact/image bất biến; cùng artifact đi qua STAG và PROD, khác nhau bằng externalized configuration.
- Triển khai rolling hoặc canary với readiness, graceful shutdown và backward-compatible DB changes.
- Migration chạy một lần trước app version cần schema; không để mọi replica tự tranh DDL production.
- Feature flag cho JSON Schema enforcement, file validation, Kafka publish, actor replay và auto-assignment phải có owner/default production được duyệt.
- Rollback ứng dụng chỉ an toàn khi migration tương thích ngược; destructive cleanup thực hiện ở release sau khi hết compatibility window.

### Quản lý cấu hình

| **Nhóm** | **Production gate** |
| --- | --- |
| Security signature/actor | Bật và required; secret khác local; nonce store HA. |
| Form validation | Chốt bật JSON Schema sau compatibility test; structural guard luôn bật. |
| File validation | Bật; không dùng local default `false`. |
| Outbox Kafka | Chốt topic/ACL/schema rồi bật publisher; dashboard backlog trước go-live. |
| Notification relay | Template/recipient/dedupe/retry được contract test. |
| OCR | Base URL, workload IAM và timeout trỏ `vhm-ocr-ekyc`; direct provider config bị loại bỏ. |
| Auto-assignment | Roster/permission/counter có fallback manual và alert. |

## 10.4 Infrastructure & Network Security

- Chỉ BFF/workload được allow mới gọi core internal ingress.
- DB, Redis, Kafka và external credentials dùng network identity/ACL tối thiểu; không public internet nếu không bắt buộc.
- Egress allowlist tới File, Market, TTOL, Message Delivery và `vhm-ocr-ekyc`; provider OCR chỉ do `vhm-ocr-ekyc` truy cập.
- TLS termination và re-encryption tuân theo platform standard; không hạ cấp clear text qua trust boundary.
- Backup, log, trace và exported report phải ở vùng dữ liệu được duyệt.

### Ma trận luồng mạng

| **Nguồn** | **Đích** | **Luồng** | **Kiểm soát bắt buộc** |
| --- | --- | --- | --- |
| Khách hàng/Đại lý | Market/Agent BFF | HTTPS nghiệp vụ | Xác thực end-user, session/rate limit tại public boundary; BFF ngoài scope Core. |
| Market/Agent BFF | Internal ingress → Dossier Core | HTTPS JSON | Private route, workload allowlist, HMAC/actor context, timestamp và nonce; không có đường public trực tiếp tới Core. |
| Dossier Core | PostgreSQL HA endpoint | Kết nối DB mã hóa | Private data subnet, security group/ACL tối thiểu, credential riêng và rotation. |
| Dossier Core | Redis HA endpoint | Nonce/replay, counter, coordination | Private route, auth/TLS, multi-AZ automatic failover; lỗi security path phải fail closed, không bypass. |
| Dossier Core | Kafka | Domain event từ outbox | Private route, TLS, ACL theo topic/producer; payload theo allowlist. |
| Dossier Core | File, Market/Project, Message, TTOL, `vhm-ocr-ekyc` | HTTPS API | Controlled egress/private endpoint, workload identity, destination allowlist và timeout theo use case. |
| Khách hàng | Private Object Storage | HTTPS PUT bằng presigned URL | Chỉ đúng object/method, TTL ngắn, không list; quyền cấp qua File contract. |
| Core/Platform | Nền tảng quan sát | Metric/log/trace | Kết nối mã hóa, field allowlist; không gửi PII, file URL, token hoặc OCR result. |

## 10.5 Migration Strategy

Liquibase quản lý schema theo phiên bản. Baseline hiện có các nhóm migration cho dossier, permissions, pipeline projection, notification outbox, checklist, source/PIC/audit và race guards. Migration production phải:

1. Chạy thử trên snapshot có kích thước gần production và kiểm tra lock duration.
2. Dùng expand/migrate/contract cho thay đổi không tương thích.
3. Xác minh unique partial index bằng preflight duplicate report trước create index.
4. Có backup/PITR marker và rollback application plan; rollback DDL chỉ dùng khi thực sự an toàn.
5. Đối soát row count, constraint/index, version và sample business query sau deploy.

Migration OCR: mọi đường BFF/provider trực tiếp được chuyển sang topology `BFF → Core → vhm-ocr-ekyc` sau contract/E2E; không dual-write kết quả vào hai nguồn. Khi ổn định, vô hiệu hóa client/provider OCR legacy rồi xóa cấu hình ở release sau.

# 11. Cost & Capacity/Performance

## 11.1 Capacity/Performance

Không dùng mục tiêu `200 req/s` hoặc P95 làm cam kết khi chưa có workload model được phê duyệt. Capacity plan phải tách ít nhất:

| **Workload** | **Đơn vị đo bắt buộc** | **Điểm nghẽn cần kiểm thử** |
| --- | --- | --- |
| Create/update/submit | TPS, P95/P99, error rate | DB transaction, JSONB, file validation, external project call. |
| List/detail/statistics | Concurrent users, page size, P95 | JSONB query/index, visibility predicate, N+1. |
| Pipeline action | Actions/minute, contention | Optimistic lock, reviewer/unit unique, notification intent. |
| Outbox/reminder | Events/minute, oldest age, recovery time | Batch lock, Kafka/Message quota. |
| Report/download | Rows/file size/concurrency | Memory, temp storage, File/Syncfusion latency. |
| OCR CCCD | Request/minute và polling rate | Core phải backoff theo `Retry-After`; quota/worker thuộc capacity `vhm-ocr-ekyc`. |

Trước production, Product cung cấp MAU/DAU, hồ sơ/ngày, peak factor, tài liệu/hồ sơ, retention và report size. Vận hành/DBA chốt pool, timeout, batch, resource request/limit và headroom; QA lưu bằng chứng load/soak test.

## 11.2 Cost

Cost drivers gồm PostgreSQL HA/backup, Redis, Kafka retention, object storage/egress, Message Delivery, document rendering và mức sử dụng `vhm-ocr-ekyc`. Dossier không hạch toán trực tiếp provider OCR. Cost model phải có unit cost trên một hồ sơ hoàn tất, storage growth theo retention, peak compute và alert ngân sách; giá trị tiền tệ là TBD do FinOps/System Owner phê duyệt.

# 12. Scalability & Reliability

## 12.1 Scaling Strategy

- Scale ngang HTTP replicas; không lưu session/actor state trong process.
- Scale relay/scanner theo leader/DB claim để không xử lý một row đồng thời.
- Dùng pagination có giới hạn và index cho filter phổ biến; report lớn chạy với quota/batch phù hợp.
- Cache chỉ tối ưu read; DB vẫn là authority. Cache miss/stale không được mở rộng quyền.
- Core phải tôn trọng `Retry-After` của OCR, tránh polling storm; OCR worker/capacity thuộc service dùng chung.
- Auto-assignment Redis counter không được chặn manual assignment khi Redis/roster suy giảm.

## 12.2 Reliability

| **Failure mode** | **Hành vi an toàn** | **Phục hồi** |
| --- | --- | --- |
| Core replica dừng giữa request | Transaction rollback hoặc commit nguyên tử | Client retry create bằng idempotency key; reload version. |
| PostgreSQL không sẵn sàng | Fail request; không nhận mutation giả thành công | HA/failover/PITR theo runbook. |
| Kafka không sẵn sàng | Outbox backlog tăng, hồ sơ vẫn commit | Relay phát lại; alert oldest age. |
| Message Delivery lỗi | Notification retry/FAILED | Manual replay/runbook; không rollback transition. |
| Redis nonce/replay không sẵn sàng | Mọi request đã ký từ BFF vào Core bị fail closed; không bypass kiểm tra replay | Redis Sentinel/Cluster hoặc managed HA tương đương qua nhiều AZ, quorum và automatic failover; alert/failover drill; downtime tính vào availability SLO. |
| Redis counter/cache không sẵn sàng | Auto-assignment/cache suy giảm nhưng không mở rộng quyền hoặc làm sai dossier authority | Manual assignment, đọc nguồn authority và rebuild cache/counter theo runbook. |
| Market/TTOL lỗi | Guard bắt buộc fail hoặc auto-assign best effort | Cache/manual path và alert tùy use case. |
| `vhm-ocr-ekyc` lỗi | Không tạo OCR mới hoặc polling báo lỗi | Không reject dossier; retry hoặc fallback theo quyết định Product. |
| File Service lỗi | Không prepare/validate/download được | Retry hữu hạn; không attach path chưa xác minh. |

## 12.3 Sao lưu và phục hồi

- PostgreSQL cần automated backup, PITR và restore drill có bằng chứng.
- RPO phải bao phủ dossier, pipeline history, checklist, reviewer và outbox trong cùng database.
- Object file do File Service backup/retention; restore phải bảo toàn reference hoặc có reconciliation.
- Redis cache/counter có thể rebuild; nonce/replay trong cửa sổ security phải fail closed và phục hồi qua topology HA, không được reset/bypass để mở traffic không kiểm soát.
- Kafka không thay thế DB backup; outbox là nguồn replay event trong retention window.
- Kiểm thử DR phải bao gồm backlog relay, notification và consistency sau restore, không chỉ khởi động ứng dụng.

# 13. Observability & Monitoring

## 13.1 Yêu cầu nền tảng

- Correlation ID xuyên BFF → core → external calls/outbox, không dùng PII.
- Structured log có schema/version, environment, service, action, outcome và error code.
- Metrics cho HTTP, DB pool/query, external dependency, cache, outbox, notification, reminder và pipeline.
- Distributed trace với sampling phù hợp; body/PII/file URL/token luôn bị loại.
- Dashboard/cảnh báo có owner, route on-call và link runbook.

## 13.2 Chỉ số bắt buộc

| **Nhóm** | **Chỉ số** |
| --- | --- |
| API | Request rate, P50/P95/P99, 4xx/5xx, timeout theo operation. |
| Business | DRAFT created, submit success/fail theo reason, transition count, revision age, duplicate conflict. |
| Checklist | Missing/invalid distribution, submit blocked `11017/11018`, progress anomaly. |
| Data | DB pool saturation, transaction/lock time, unique conflict, slow query, storage growth. |
| Outbox | Pending count, oldest age, publish/send rate, retry, FAILED. |
| Assignment | Auto/manual rate, roster/Redis failure, unassigned stage age. |
| External | File/Market/TTOL/Message/OCR availability, latency, timeout và error class. |
| Security | Invalid signature, stale timestamp, nonce replay, actor expiry/role/visibility denial. |

Label không được chứa dossier ID, actor ID, project ID có cardinality cao hoặc PII. Business drill-down dùng log/audit có kiểm soát thay vì metric label.

## 13.3 Cảnh báo

| **Cảnh báo** | **Điều kiện nguyên tắc** | **Ưu tiên** |
| --- | --- | --- |
| Mutation error/latency burn | Vượt SLO theo nhiều cửa sổ | P1/P2 theo burn rate |
| PostgreSQL saturation/lock | Pool gần cạn, lock/transaction bất thường | P1 |
| Outbox/notification backlog | Oldest age vượt delivery SLO | P1/P2 |
| Reviewer chưa được assign | Stage age vượt ngưỡng nghiệp vụ | P2 |
| Signature/replay anomaly | Tăng đột biến hoặc client bị deny liên tục | Security incident |
| File/OCR/Market/TTOL dependency | Error/timeout vượt budget | P2; P1 nếu chặn toàn bộ submit |
| Reminder missed | Scan không chạy hoặc due row quá hạn | P2 |

## 13.4 SLI/SLO

SLI bắt buộc: availability của read/mutation, successful submit/transition, event delivery latency, notification intent delivery, reminder timeliness và data consistency. Giá trị mục tiêu, measurement window, error budget, maintenance exclusion và owner đều `TBD`; không được chuyển trạng thái `APPROVED` nếu chưa có baseline đo trên STAG/load test.

# 14. Operational Readiness

## 14.1 RTO & RPO

| **Hạng mục** | **Mục tiêu** | **Trạng thái** |
| --- | --- | --- |
| RTO dossier core | TBD | System Owner/Vận hành phê duyệt và diễn tập |
| RPO PostgreSQL | TBD | DBA phê duyệt và có bằng chứng PITR |
| Event/notification recovery | Trong delivery SLO được duyệt | Cần backlog replay drill |
| File/object recovery | Theo File Service SLA và retention | Contract ngoài |
| OCR recovery | Theo `vhm-ocr-ekyc` SLO; không mất `ocrId` đã nhận | Contract ngoài |

## 14.2 Runbook bắt buộc

- Invalid HMAC/actor signature, Redis nonce store lỗi, automatic failover và luân chuyển/revoke secret; không có thủ tục bypass replay control.
- PostgreSQL failover/PITR, Liquibase lỗi, unique-index migration và data reconciliation.
- Kafka down, outbox backlog, duplicate publish và poison event.
- Message Delivery lỗi, notification `FAILED`, dedupe và manual replay.
- Reviewer roster/Redis counter lỗi, unassigned dossier và manual assignment.
- Market/special-day cache lỗi ảnh hưởng sinh mã/SLA reminder.
- File ownership/existence/download incident và object mồ côi.
- `vhm-ocr-ekyc` unavailable, OCR stuck/terminal error và migration rollback.
- PII/credential xuất hiện trong log, export hoặc event.
- Rollback release khi migration đã chạy.

## 14.3 Danh sách kiểm tra sẵn sàng cơ sở

- Owner/on-call/escalation matrix cho core và từng dependency.
- Production config review chứng minh signature, actor, file validation, schema validation và relays đúng default.
- Dashboard/cảnh báo đã fire thử và route đúng người.
- Backup/restore, deployment rollback, secret rotation và backlog replay đã diễn tập.
- OpenAPI/event/schema/pipeline version được đóng băng và contract test.
- Privacy retention/deletion/export access được phê duyệt.
- Mọi vấn đề Critical/High tại mục 16 đã đóng hoặc có risk acceptance hữu hạn, owner và expiry.

# 15. Testing & Quality Strategy

## 15.1 Phạm vi kiểm thử bắt buộc

Bộ kiểm thử phải bao phủ invariant domain, database concurrency, API/security contract, external integration, outbox/recovery, performance và OAT/DR. Unit test không thay thế PostgreSQL integration test, contract test, E2E hoặc failure drill.

## 15.2 Cổng chất lượng

| **Lớp kiểm thử** | **Phạm vi bắt buộc** | **Cổng** |
| --- | --- | --- |
| Unit/domain | State/action/role/ownership, checklist, code/unit, masking, error mapping | Bắt buộc |
| Database integration | Migration, JSONB, partial index, advisory/row lock, optimistic lock, cascade | Bắt buộc |
| Concurrency | Concurrent idempotent create, duplicate identity/project, allocate unit, reviewer claim | Bắt buộc |
| API/contract | Agent ↔ core DTO/header/status/error/backward compatibility | Bắt buộc |
| Security | Signature, nonce replay, Redis failover không bypass control, actor expiry/role/visibility/IDOR, body actor injection | Bắt buộc |
| Privacy/consent | Consent gate, bằng chứng và withdrawal; chi tiết UI theo artefact L3 | Bắt buộc |
| External contract | File, Market, TTOL, Message Delivery và `vhm-ocr-ekyc` | Bắt buộc |
| Outbox/reliability | Rollback, broker/send lỗi, publish lặp, retry/FAILED, backlog recovery | Bắt buộc |
| E2E | Create DRAFT → upload → PATCH → submit → multi-stage decision/revision | Bắt buộc |
| Performance/soak | Workload model mục 11, report/download và polling OCR | Bắt buộc |
| OAT/DR | Deploy/rollback, multi-AZ failover gồm Redis security path, restore, secret rotation, alert/runbook | Bắt buộc |

## 15.3 Kịch bản kiểm thử trọng yếu

- Create `{}` trả DRAFT/version đúng DB; create không sinh checklist.
- Concurrent create cùng actor/key trả cùng dossier; actor khác không replay được key.
- `MARKET` thiếu key bị từ chối; caller machine identity phải có allowlist/scope, audit và negative test.
- Agent timeout sau khi Core đã commit rồi retry phải nhận lại cùng dossier. Nếu deviation Agent không bắt buộc key được phê duyệt, phải kiểm thử phát hiện–đối soát–xóa DRAFT mồ côi theo owner/SLA đã chốt.
- Hai hồ sơ active cùng CCCD+dự án bị chặn ở create, full update, submit và DB race guard.
- Full update file không tồn tại rollback cả dossier/checklist/outbox; xóa documents reset/delete projection đúng.
- Submit không checklist/thiếu required trả `11017/11018`; required complete mới chuyển trạng thái.
- If-Match cũ, concurrent command, allocate cùng căn và double claim không làm lost update.
- Mọi state/action/role/ownership branch của pipeline, gồm revision loop và revoke sau approve.
- Signature/body hash/timestamp/nonce/actor JTI/visibility negative matrix và không có body actor spoofing.
- Mất Redis node/AZ phải automatic failover trong SLO được duyệt, không chấp nhận nonce lặp hoặc mở bypass; outage của nonce path phải được tính vào availability end-to-end.
- Outbox crash trước/sau broker acknowledgement có thể phát lặp nhưng không mất event.
- Notification retry/dedupe/FAILED và reminder T+6/T+18 qua ngày nghỉ/cycle mới.
- OCR: idempotent create, `202`/`Retry-After`, polling `QUEUED/PROCESSING/terminal`, user-confirm-before-apply, timeout và service unavailable.
- Negative contract/E2E phải chứng minh prepare-upload và submit bị từ chối khi consent không hợp lệ hoặc đã bị rút.
- UI/L3 phải kiểm thử việc thu/rút consent và xác nhận submit không bị dùng để suy diễn consent.
- File cross-owner/cross-reference, path traversal, MIME/magic/checksum, expired presign và upload grant.
- Migration trên dữ liệu trùng, rollback application và restore/PITR reconciliation.

## 15.4 Dữ liệu kiểm thử và quản lý bằng chứng

Test tự động/SIT chỉ dùng CCCD/file tổng hợp hoặc đã làm sạch. Dữ liệu cá nhân thật cần phê duyệt, kho cô lập, purpose/retention đích danh và bằng chứng xóa. Fixture của dependency phải có version, không chứa credential/PII và bao phủ cả success, timeout, malformed response, duplicate và permission denial.

Bằng chứng quality gate phải được lưu theo release, tối thiểu gồm test report, migration rehearsal, contract/E2E result, security scan, load/soak report, restore drill, dashboard/alert verification và danh sách risk acceptance còn hiệu lực.

# 16. Risks & Open Issues

## 16.1 Architecture Risks

| **Mã** | **Nhóm** | **Mô tả/ảnh hưởng** | **Mức độ** | **Giảm thiểu/điều kiện đóng** |
| --- | --- | --- | --- | --- |
| AR-001 | Toàn vẹn nghiệp vụ | Tin `documents[].isRequired` từ client có thể làm sai bộ hồ sơ được phép submit | Nghiêm trọng | Tích hợp nguồn Checklist chuẩn, snapshot/version server-side, contract test. |
| AR-002 | An toàn file | File response chưa chứng minh uploader/upload-grant owner | Nghiêm trọng | File contract trả owner/grant và verify khi attach; negative E2E. |
| AR-003 | Determinism | Thiếu pipeline ID/version authoritative có thể route hồ sơ sai quy trình | Cao | Bắt buộc unique selection rule và fail khi kết quả không xác định. |
| AR-004 | Security | `source=MARKET` chưa hạn chế theo machine identity | Cao | Client allowlist/scope tại BFF và core, audit/test. |
| AR-005 | Tích hợp OCR | Duy trì direct synchronous OCR sẽ phá vỡ ranh giới capability dùng chung | Cao | Chỉ tích hợp `vhm-ocr-ekyc`; E2E/contract đạt và loại bỏ legacy trước go-live. |
| AR-006 | Audit/PIC | Có `picId`/audit table nhưng chưa có use case gán/chuyển PIC hoàn chỉnh | Trung bình | Chốt owner, API, permission và audit semantics hoặc bỏ khỏi contract. |
| AR-007 | Contract | Dùng không nhất quán `required` và `isRequired` có thể tạo quyết định duyệt khác nhau | Cao | Chỉ công bố một field chuẩn trong Form/Checklist Contract và có regression test. |
| AR-008 | Notification | Schema có nhiều kênh nhưng relay hiện chỉ dispatch email | Trung bình | Chốt scope kênh; triển khai hoặc loại khỏi contract công bố. |
| AR-009 | Validation | JSON Schema enforcement có thể mặc định tắt | Cao | Chốt production default bật, compatibility test và alert config drift. |
| AR-010 | Event delivery | Kafka outbox publish có thể mặc định tắt | Cao | Production config gate, readiness/metric và backlog verification. |
| AR-011 | Privacy | Retention/deletion/legal hold/audit access cho CCCD chưa được định nghĩa | Nghiêm trọng | DPIA/policy/runbook và bằng chứng purge/restore. |
| AR-012 | Availability | Full E2E phụ thuộc File Service và nhiều enterprise dependency | Cao | Sandbox/SLA, timeout/degradation, synthetic probe và runbook. |
| AR-013 | Privacy | Notice, consent evidence và withdrawal contract chưa được phê duyệt | Nghiêm trọng | Product/Market/Privacy/Legal phê duyệt artefact L3 và E2E trước dữ liệu thật. |
| AR-014 | Idempotency/Integrity | Agent create chưa bắt buộc key; timeout sau commit rồi retry có thể tạo nhiều DRAFT `{}` mồ côi, gây sai số và tăng dữ liệu rác | Cao | Bắt buộc `Idempotency-Key` cho Agent như MARKET trước production; hoặc có deviation/risk acceptance phê duyệt kèm phát hiện, đối soát, xóa, owner, SLA, metric và E2E retry. |
| AR-015 | Availability/Security | Redis nonce/replay là dependency đồng bộ fail-closed; Redis downtime chặn toàn bộ request đã xác thực từ BFF vào Core và làm availability Core phụ thuộc trực tiếp vào Redis | Nghiêm trọng | Redis Sentinel/Cluster hoặc managed HA tương đương qua nhiều AZ, quorum/automatic failover, capacity headroom, monitoring và failover drill; đưa dependency vào SLO/runbook, không bypass security. |

## 16.2 Vấn đề thiết kế cần quyết định

| **Vấn đề cần quyết định** | **Owner đề xuất** | **Điều kiện đóng** |
| --- | --- | --- |
| Nguồn Checklist chuẩn, contract và version/snapshot | BA/Checklist Team/Backend | API/schema/authority được duyệt; client không tự quyết `isRequired`. |
| File ownership/upload-grant khi attach | File Team/ANBM | Response/verify API và E2E cross-owner đạt. |
| Pipeline selection authoritative | BA/Kiến trúc/Backend | Một pipeline ID/version rõ ràng từ contract/config. |
| Machine identity cho MARKET | Kiến trúc IAM/Backend | Client scope allowlist, audit và negative test. |
| Contract dossier ↔ `vhm-ocr-ekyc` cho CCCD hai mặt và apply result | OCR Team/BFF/Backend | OpenAPI L3, opaque refs, IAM, status/result mapping và E2E ký duyệt. |
| Ý nghĩa/ownership của `picId` so với stage reviewer | Product/BA/Backend | Use case và permission/audit rõ hoặc bỏ field. |
| SLO, peak workload, RTO/RPO và capacity/cost | Product/Vận hành/DBA/FinOps | Baseline số được duyệt và load/DR đạt. |
| Retention, deletion, legal hold, encryption và audit access | Privacy/Pháp chế/ANBM | Policy/DPIA/runbook được phê duyệt. |
| Notification channels và recipient authority | Product/Message Team | Contract channel/dedupe/template/address và test đạt. |
| Form schema enforcement và backward compatibility | BA/Backend/QA | Bật trên STAG, clean data report và regression đạt. |
| Consent notice/UI/evidence/withdrawal | Product/Market/Privacy/Legal/Backend | Artefact L3 có owner, version, approval và E2E. |
| Agent create idempotency | Product/Agent/Backend/Kiến trúc | Enforce key thống nhất; hoặc phê duyệt deviation có lý do nghiệp vụ, reconciliation/cleanup owner/SLA/metric và E2E retry. |
| Redis HA cho nonce/replay fail-closed | Platform/SRE/ANBM/Backend | Chốt Sentinel/Cluster/managed mode, topology multi-AZ/quorum, failover SLO, dashboard/runbook và diễn tập đạt. |

Vấn đề mở không mặc nhiên được chấp nhận. Risk acceptance phải có owner, phạm vi, kiểm soát bù trừ, người phê duyệt và ngày hết hạn.

# Appendix

## A. Glossary

| **Thuật ngữ** | **Định nghĩa** |
| --- | --- |
| NOXH | Nhà ở Xã hội. |
| Dossier | Aggregate hồ sơ đăng ký một applicant cho một project. |
| BFF | Public boundary xác thực kênh và gọi core bằng signed workload/actor context. |
| PKD/PTT/SXD | Các cấp Sales/Procedure/Department-of-Construction trong pipeline. |
| Checklist | Projection tài liệu bắt buộc/trạng thái upload/OCR/review của dossier. |
| Pipeline | Cấu hình state/action/role/ownership có phiên bản, thực thi trong core. |
| Actor context | Payload danh tính nghiệp vụ được BFF ký và core xác minh. |
| Visibility | Phạm vi hồ sơ actor được phép đọc/xử lý. |
| Idempotency key | Khóa opaque để replay an toàn create/OCR. |
| Transactional outbox | Ghi business state và ý định phát/gửi trong cùng transaction DB. |
| `vhm-ocr-ekyc` | Capability OCR/eKYC dùng chung, sở hữu lifecycle và kết quả OCR chuẩn. |
| Opaque reference | Identifier tương quan không nhúng PII hoặc secret. |

## B. References

| **Tài liệu/artefact** | **Tham chiếu** |
| --- | --- |
| L2 - Dịch vụ OCR/eKYC dùng chung | [L2 - VHMKDO2O - Dịch vụ OCR/eKYC](https://vin3s.atlassian.net/wiki/spaces/VARW/pages/3014268156/L2+-+VHMKDO2O+-+D+ch+v+OCR+eKYC) |
| TDD gốc - Dịch vụ quản lý hồ sơ NOXH | [L2 - VHMKDO2O - Nhà ở xã hội](https://vin3s.atlassian.net/wiki/spaces/VARW/pages/3009027308/L2+-+VHMKDO2O+-+Nh+x+h+i) |
| Luật Bảo vệ dữ liệu cá nhân số 91/2025/QH15 | [Cổng TTĐT Chính phủ](https://vanban.chinhphu.vn/?classid=1&docid=214590&pageid=27160&typegroup=) — hiệu lực 01/01/2026 |
| Nghị định 356/2025/NĐ-CP | [Cổng TTĐT Chính phủ](https://vanban.chinhphu.vn/?classid=1&docid=216387&pageid=27160) |
| Pipeline Definition Social Housing v1 | Tài liệu L3 chính thức: TBD |
| Form Data Contract Social Housing v1 | Tài liệu L3 chính thức: TBD |
| Database model và migration plan | Tài liệu L3/DBA chính thức: TBD |

## C. Đầu vào bắt buộc trước production

| **Đầu vào** | **Chủ sở hữu** | **Cổng** |
| --- | --- | --- |
| Checklist authority/version/snapshot | BA/Checklist Team | Submit/UAT |
| File ownership/upload-grant | File Team/ANBM | Attachment security |
| Pipeline ID/version selection | BA/Kiến trúc | Cấu hình pipeline |
| OCR OpenAPI/IAM/two-side CCCD/apply-result | OCR Team/BFF | E2E OCR |
| MARKET workload scope | IAM/Backend | Security approval |
| Privacy retention/deletion/encryption | Privacy/Pháp chế/ANBM | Dữ liệu thật |
| Consent notice/UI/evidence/withdrawal | Privacy/Pháp chế/Product/Market/Backend | UAT với dữ liệu thật/production |
| Workload/SLO/capacity/cost | Product/Vận hành/FinOps | Load/OAT |
| Agent create idempotency decision | Product/Agent/Backend/Kiến trúc | API approval/production |
| Production landing zone/network/Redis HA | Platform/SRE/ANBM/Backend | OAT/production |
| RTO/RPO/backup/restore | DBA/Vận hành | DR/OAT |
| Dashboard/alert/on-call/runbook | Vận hành | Go-live |
| Contract test File/Market/TTOL/Message/Kafka | Tích hợp/QA | Release |

## D. Danh mục quyết định kiến trúc (ADR)

| **ID** | **Quyết định** | **Cơ sở/hệ quả** | **Trạng thái** |
| --- | --- | --- | --- |
| ADR-001 | PostgreSQL là source of truth | Dossier/checklist/pipeline/history/outbox nhất quán; cần HA/PITR | CHẤP NHẬN |
| ADR-002 | Pipeline versioned thực thi trong process | Không cần Camunda/Zeebe; transition nguyên tử với dossier | CHẤP NHẬN |
| ADR-003 | Create luôn DRAFT, submit là command riêng | Hỗ trợ upload và hoàn thiện snapshot trước nộp | CHẤP NHẬN |
| ADR-004 | JSONB snapshot + schema version | Linh hoạt form; đổi lại cần schema/guard/index JSON rõ ràng | CHẤP NHẬN |
| ADR-005 | Advisory lock + actor-scoped replay + DB unique | Chống concurrent forwarding race và key reuse sai actor | CHẤP NHẬN |
| ADR-006 | Partial unique index là race guard cuối cho CCCD+dự án | Service precheck cho UX, DB bảo đảm invariant | CHẤP NHẬN |
| ADR-007 | Transactional outbox cho event/notification | Không mất intent sau commit; chấp nhận at-least-once | CHẤP NHẬN |
| ADR-008 | Signed actor context và deny-by-default visibility | Không tin identity/role từ client body | CHẤP NHẬN |
| ADR-009 | File path opaque, không kiểm tra dossier-prefix | Upload namespace độc lập; ownership phải dựa contract File Service | CHẤP NHẬN có điều kiện |
| ADR-010 | OCR qua capability dùng chung `vhm-ocr-ekyc` | Dossier không sở hữu provider/worker/raw result; cần migration legacy | ĐỀ XUẤT — chờ phê duyệt |
| ADR-011 | OCR chỉ áp dụng sau xác nhận và PATCH dossier | OCR không tự quyết định nghiệp vụ; giữ optimistic/business guards | ĐỀ XUẤT — chờ phê duyệt |
| ADR-012 | Domain sở hữu consent cho hành trình NOXH | Market/BFF hiển thị và thu; Core lưu/kiểm tra tại upload/submit; `vhm-ocr-ekyc` không sở hữu consent gốc; chi tiết thuộc L3 | ĐỀ XUẤT — chờ Privacy/Legal và Kiến trúc duyệt |
