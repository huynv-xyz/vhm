> **TÀI LIỆU NỘI BỘ** — Tài liệu mô tả kiến trúc mục tiêu L2 của năng lực quản lý hồ sơ Nhà ở Xã hội. Không chia sẻ ra ngoài phạm vi dự án khi chưa được phê duyệt.

# L2 - VHMKDO2O - Dịch vụ quản lý hồ sơ Nhà ở Xã hội

| **Trường** | **Nội dung** |
| --- | --- |
| **Trạng thái** | **ĐANG THẨM ĐỊNH (UNDER REVIEW)** |
| **Phiên bản & lịch sử thay đổi** | `v0.9.0` — 15/08/2026 — Thiết lập kiến trúc mục tiêu cho dịch vụ quản lý hồ sơ NOXH |
| **Chủ sở hữu tài liệu** | TBD |
| **Chủ sở hữu hệ thống** | TBD |
| **Hệ thống** | `vhm-dossier-core` — modular monolith quản lý hồ sơ và pipeline NOXH |
| **Hệ thống liên quan** | Kênh Agent/Back Office, Agent API (BFF), kênh Market, Market API (BFF), `vhm-ocr-ekyc`, PostgreSQL, Redis, Kafka, File Management, Message Delivery, TTOL |
| **Đội ngũ/PIC** | Backend: TBD · Kiến trúc: TBD · Tích hợp: TBD · ANBM: TBD · Quyền riêng tư dữ liệu: TBD · Vận hành: TBD |
| **Người rà soát/phê duyệt** | Sản phẩm/BA: TBD · Kiến trúc: TBD · ANBM: TBD · DBA/Vận hành: TBD · QA: TBD |
| **Mốc thiết kế** | Kiến trúc mục tiêu phục vụ thẩm định giải pháp và làm đầu vào cho thiết kế L3 |
| **Phạm vi hệ thống** | Dossier Core và các ranh giới tích hợp thể hiện trong sơ đồ ngữ cảnh |
| **Tài liệu nguồn** | SRS/BRD NOXH: TBD · L2 OCR dùng chung của `vhm-ocr-ekyc` |
| **Lần rà soát gần nhất** | 15/08/2026 |

## Approval & Review Gates

| **Vai trò rà soát/phê duyệt** | **Phạm vi rà soát** | **Quyết định** | **Ngày xác nhận** |
| --- | --- | --- | --- |
| Chủ sở hữu Sản phẩm/Nghiệp vụ | Luồng đăng ký, checklist, duyệt PKD/PTT/SXD, SLA nhắc bổ sung | Chờ rà soát | — |
| Kiến trúc Ứng dụng/Giải pháp | Ranh giới modular monolith, API, pipeline và tính nhất quán | Chờ rà soát | — |
| Kiến trúc Tích hợp | Agent API, Market API, File Management, `vhm-ocr-ekyc`, Message Delivery, TTOL, Kafka | Chờ rà soát | — |
| ANBM | IAM nội bộ, actor context, chống phát lại, PII và file | Chờ rà soát | — |
| Quyền riêng tư/Pháp chế | CCCD, thông tin liên hệ, lưu giữ, truy cập và xóa dữ liệu | Chờ rà soát | — |
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
| OpenAPI Agent API/Market API ↔ Dossier Core | DRAFT | Backend/Tích hợp/ANBM | Trước duyệt API | Cùng business API nhưng signed claims khác theo channel; liên kết chính thức: TBD |
| Form Data Contract Social Housing v1 | DRAFT | Backend/BA | Trước UAT | Liên kết tài liệu chính thức: TBD |
| Pipeline Definition Schema, Social Housing v1 và activation/migration policy | DRAFT | Backend/BA/Kiến trúc/Vận hành | Trước UAT | Liên kết tài liệu chính thức: TBD |
| Contract Checklist chuẩn | **CHƯA CÓ** | BA/Tích hợp/Backend | Trước production | Chưa chốt nguồn authority và version |
| Contract File Management cho tài liệu không OCR | **CHƯA ĐỦ** | File Team/ANBM | Trước production | Chưa có owner/upload-grant contract |
| OpenAPI Dossier Core ↔ `vhm-ocr-ekyc` | DRAFT | OCR Team/Backend/ANBM | Trước production | Chỉ dùng OCR CCCD hai mặt; cần chốt IAM, `subjectRef` đầu vào, prepare-upload, `POST /ocr`, `/ocr/result`, timeout và retention/delete runbook |
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

`vhm-dossier-core` số hóa việc tiếp nhận, cập nhật, kiểm tra và phê duyệt hồ sơ đăng ký Nhà ở Xã hội. Kênh Agent hoặc Market tạo bản nháp, tải tài liệu, hoàn thiện snapshot hồ sơ rồi chủ động nộp. Hồ sơ được xử lý qua các nhóm PKD, PTT và đầu mối SXD. Kênh Market kiểm soát theo authenticated data subject tại Market API; kênh Agent/Back Office kiểm soát theo role do BUS định nghĩa, scope và pipeline policy.

#### Current Business Problem

- Hồ sơ giấy/Excel/email khó kiểm soát tính đầy đủ, phiên bản và lịch sử xử lý.
- Nhập tay CCCD và thông tin khách hàng dễ sai; tài liệu thiếu hoặc không tồn tại chỉ được phát hiện muộn.
- Nhiều người có thể tạo hồ sơ cho cùng khách hàng và dự án, dẫn đến trùng nghiệp vụ và tranh chấp căn.
- Market API phải xác thực customer và ràng buộc đúng chủ thể dữ liệu; Agent API phải truyền role BUS và scope đã xác thực. Core áp dụng signed context tương ứng cùng state/invariant nghiệp vụ.
- Luồng trả bổ sung cần SLA, nhắc hẹn, lịch sử và khả năng tiếp tục đúng cấp duyệt.
- Các tích hợp File, OCR, TTOL và Message Delivery có độ sẵn sàng khác nhau; không được làm mất trạng thái đã commit.

#### Business Objectives

- Tạo hồ sơ `DRAFT` nhanh với form rỗng hoặc một phần; create không tự động submit.
- Tiếp nhận hồ sơ từ kênh Agent/Back Office qua Agent API (`source=AGENT`) và từ kênh Market qua Market API (`source=MARKET`).
- Duy trì một snapshot `formData`/`metadata` có version, lịch sử trạng thái và projection pipeline.
- Ngăn hồ sơ active trùng theo `(subjectRef, projectId)`; `subjectRef` là định danh opaque, ổn định do nguồn định danh domain có thẩm quyền cấp/bind server-side và được truyền vào `vhm-ocr-ekyc`, Core không lưu CCCD để đối chiếu.
- Bảo đảm checklist bắt buộc đã được upload trước `SUBMIT`.
- Thực thi pipeline PKD → PTT → SXD, bao gồm trả bổ sung, phân công, nhận xử lý, cấp/thu hồi căn và hồ sơ giấy.
- Cung cấp list/detail/statistics/export và progress checklist cho hai BFF nghiệp vụ.
- Tách việc gửi sự kiện và notification khỏi transaction nghiệp vụ bằng outbox.
- Không persist PII/OCR của applicant/spouse tại Dossier Core; dữ liệu chỉ được proxy theo actor context, visibility, ownership và xác thực nội bộ có chữ ký.

## 1.1 In Scope

| **Capability** | **Phạm vi** | **Yêu cầu thiết kế** |
| --- | --- | --- |
| Hồ sơ | Create/read/list/update/delete DRAFT, statistics trên aggregate nghiệp vụ; Dossier Core không cung cấp lookup applicant theo PII | `BẮT BUỘC` |
| Nguồn tạo hồ sơ | Kênh Agent/Back Office qua Agent API; kênh Market qua Market API | `BẮT BUỘC` |
| Luồng đăng ký công khai | Create DRAFT → prepare upload → PATCH snapshot → submit | `BẮT BUỘC` |
| Checklist | Snapshot từ nguồn chuẩn, progress, missing/invalid, readiness submit | `BẮT BUỘC` |
| Pipeline | State/action; Market dùng data-subject ownership, Agent/Back Office dùng BUS role/scope/ownership | `BẮT BUỘC` |
| Phân công reviewer | Manual, round-robin PKD và danh sách nhân sự TTOL cho PTT/SXD | `BẮT BUỘC` |
| Quyền dự án | Grant/revoke/list/group theo team/project/scope | `BẮT BUỘC` |
| OCR CCCD | Tích hợp capability dùng chung `vhm-ocr-ekyc` theo mô hình bất đồng bộ | `BẮT BUỘC` |
| Tài liệu/media | Chuẩn bị upload, xác minh reference/quyền attach, download và lưu artefact xuất | `BẮT BUỘC` |
| Notification/reminder | Transactional outbox và nhắc bổ sung T+6/T+18 | `BẮT BUỘC` |
| Báo cáo/tải file | Export danh sách NOXH; tải hợp đồng/tệp đính kèm | `BẮT BUỘC` |
| Notes/hardcopy | Ghi chú và theo dõi hồ sơ giấy | `BẮT BUỘC` |

## 1.2 Out of Scope

- Camunda/Zeebe hoặc BPMN engine bên ngoài; pipeline hiện chạy trong cùng ứng dụng và transaction.
- Sở hữu master data dự án, người dùng, đội nhóm, ngày nghỉ hoặc file binary.
- Thuật toán, provider credential, queue/worker và dữ liệu raw OCR; toàn bộ thuộc `vhm-ocr-ekyc`.
- Mọi capability ngoài OCR CCCD hai mặt của service dùng chung; Dossier không gọi các API ngoài phạm vi này.
- Quyết định pháp lý về đủ điều kiện NOXH dựa riêng vào OCR.
- Quản trị checklist chuẩn ở trình duyệt; `isRequired` do client gửi chưa thể là authority production.
- Thanh toán, ký điện tử và tích hợp tự động trực tiếp với cơ quan SXD.
- Chứng minh người upload sở hữu file nếu File Contract hoặc OCR media contract chưa trả owner/upload-grant đã xác minh.

### Assumptions, Constraints & Dependencies

| **ID** | **Giả định/Ràng buộc** | **Trạng thái** | **Ảnh hưởng** |
| --- | --- | --- | --- |
| A-01 | Agent API và Market API là hai inbound BFF ngang hàng của Dossier Core | Quyết định kiến trúc | Hai kênh không gọi trực tiếp Core và không gọi chéo BFF. |
| A-02 | PostgreSQL là nguồn sự thật của hồ sơ, checklist, projection pipeline và outbox | Quyết định hiện hành | Mọi mutation trọng yếu dùng cùng transaction DB. |
| A-03 | Create chỉ tạo `DRAFT`; submit là lệnh riêng | Contract bắt buộc | Agent API và Market API phải điều phối đủ bốn bước đăng ký. |
| A-04 | Kafka có thể giao lặp; relay/consumer phải idempotent | Giả định nền tảng | Outbox chấp nhận publish lặp, không làm lặp quyết định nghiệp vụ. |
| A-05 | File path là opaque; namespace upload độc lập với dossier ID | Đã xác minh STG | Không áp dụng kiểm tra prefix `registrations/{dossierId}/`. |
| A-06 | Mỗi dossier phải nhận một pipeline ID/version xác định | `BẮT BUỘC` | Không lựa chọn pipeline theo thứ tự cấu hình. |
| A-07 | Structural guard và Form Data Contract phải được thực thi trước persistence | `BẮT BUỘC` | Production không được bỏ qua schema validation. |
| A-08 | OCR tài liệu đi qua `vhm-ocr-ekyc`; dossier không gọi provider trực tiếp | Quyết định kiến trúc mục tiêu | Cần OpenAPI L3, workload IAM và migration khỏi client OCR legacy. |
| A-09 | Tài liệu không OCR đi trực tiếp File Management; media OCR đi qua `vhm-ocr-ekyc` | Quyết định kiến trúc mục tiêu | Phân tuyến theo mục đích sử dụng, không theo extension hoặc lựa chọn từ client. |
| A-10 | External services có contract/SLA riêng | `BÊN NGOÀI` | Cần timeout, retry hữu hạn, monitoring và contract test. |
| A-11 | Market Customer không dùng business role | Quyết định phân quyền | Market API xác thực và kiểm tra data-subject ownership; Core không yêu cầu role Agent cho request Market. |
| A-12 | Agent/Back Office dùng role catalogue do BUS định nghĩa | Quyết định phân quyền | Role/scope phải đến từ nguồn tin cậy, được Agent API ký và được pipeline map tới action. |

### Stakeholders & Personas

| **Nhóm** | **Trách nhiệm/quyền** |
| --- | --- |
| Market Customer | Không có business role; Market API xác thực và chỉ cho thao tác trên hồ sơ của đúng data subject theo trạng thái nghiệp vụ. |
| Agent / `APPLICANT_AGENT` | Role thuộc catalogue BUS; tạo/cập nhật/upload/submit/resubmit và xem hồ sơ trong project/scope được cấp. |
| PKD / `PKD`, `PKD_LEAD` | Phân công/nhận hồ sơ, cấp căn, duyệt, trả bổ sung, từ chối, hồ sơ giấy. |
| PTT / `PTT`, `PTT_LEAD` | Kiểm tra thủ tục, duyệt/trả bổ sung/từ chối, chuyển SXD. |
| Đầu mối SXD | Được mô hình hóa bằng stage `SXD` nhưng xử lý qua roster/role PTT hiện hành. |
| BO/Admin | Quản lý quyền dự án, tra cứu/báo cáo và vận hành. |
| BUS | Chủ sở hữu catalogue role Agent/Back Office và mapping semantics nghiệp vụ; cơ chế phân phối/runtime source cần được đóng băng ở contract L3. |
| Agent API | BFF Agent/Back Office; xác thực actor, lấy/kiểm tra role BUS và scope, map DTO, gán `source=AGENT`, ký request/actor context. |
| Market API | BFF Market; xác thực customer, kiểm tra data-subject/object ownership, map DTO, gán `source=MARKET`, ký request/data-subject context; không phát sinh role Agent giả. |
| File/`vhm-ocr-ekyc`/Message Delivery/TTOL | Cung cấp năng lực tích hợp không thuộc sở hữu core. |

### Personal Data Processing Summary

| **Dữ liệu** | **Mục đích** | **Vị trí** | **Kiểm soát hiện hành/yêu cầu** |
| --- | --- | --- | --- |
| CCCD, họ tên, ngày sinh, giới tính, ngày/nơi cấp và địa chỉ trích xuất từ CCCD applicant/spouse | OCR CCCD và hiển thị kết quả tạm thời để người dùng kiểm tra | Kết quả mã hóa tại `vhm-ocr-ekyc`; byte media tại kho riêng tư do File Management quản lý | Dossier Core chỉ proxy projection từ `/ocr/result` trong vòng đời request, không persist/cache/log. |
| Số điện thoại và email applicant/spouse | Liên hệ nghiệp vụ/notification | Nguồn hồ sơ khách hàng có thẩm quyền: `TBD`; **không thuộc OCR contract** | Dossier Core không persist; không forward sang `vhm-ocr-ekyc`; nguồn hiển thị/gửi phải được chốt ở Contract L3. |
| Định danh applicant trong Dossier | Duplicate guard và liên kết nghiệp vụ | `dossier.subject_ref` dạng opaque | Nguồn định danh domain có thẩm quyền cấp/bind server-side trước `POST /ocr`; không nhận từ form, không chứa hoặc đảo ngược được về CCCD. |
| Reviewer/recipient | Phân công và notification | Actor/recipient ID dạng opaque | Không lưu tên/email trong Dossier; resolve tại nguồn IAM/TTOL/Message khi hiển thị hoặc gửi. |
| Đường dẫn file | Gắn tài liệu | Tài liệu không OCR: JSONB/checklist + File Management; media OCR: reference tại `vhm-ocr-ekyc`, byte tại File Management | Core chỉ lưu `s3PathFile` của tài liệu không OCR; không persist media path/presigned URL OCR. |
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
| ARCH-12 | Core gọi File Management cho tài liệu không OCR; media OCR chỉ đi qua `vhm-ocr-ekyc`, không gọi provider hoặc File Management trực tiếp trong nhánh OCR. |
| ARCH-13 | Cấp duyệt là cấu hình có phiên bản; không gắn cứng số lượng hoặc thứ tự cấp duyệt vào API, database hay kênh. |
| ARCH-14 | Pipeline đã công bố là bất biến theo `(pipelineCode, pipelineVersion)`; thay đổi luồng phải tạo phiên bản mới. |
| ARCH-15 | `vhm-ocr-ekyc` là source of truth cho lifecycle, media reference và kết quả OCR; File Management/kho riêng tư giữ byte media. Dossier Core chỉ lưu `subjectRef` opaque và không tạo bản sao PII/kết quả. |

## 2.2 Sơ đồ kiến trúc ứng dụng

### 2.2.1 Sơ đồ ngữ cảnh hệ thống

```mermaid
flowchart LR
    subgraph Channels[Inbound Channels]
        direction TB
        AgentChannel[Kênh Agent / Back Office]
        MarketChannel[Kênh Market]
    end
    AgentChannel --> AgentAPI[Agent API / BFF]
    MarketChannel --> MarketAPI[Market API / BFF]
    AgentAPI -->|Basic + HMAC + signed actor context| Core[vhm-dossier-core]
    MarketAPI -->|Basic + HMAC + signed actor context| Core
    Core --> PG[(PostgreSQL)]
    Core --> Redis[(Redis)]
    Core --> Kafka[(Kafka)]
    Core --> File[File Management]
    Core --> OCR[vhm-ocr-ekyc]
    OCR --> File
    Core --> Msg[Message Delivery]
    Core --> TTOL[TTOL]
```

### 2.2.2 Sơ đồ thành phần

```mermaid
flowchart LR
    subgraph Channels[Inbound Channels]
        direction TB
        AgentChannel[Kênh Agent / Back Office]
        MarketChannel[Kênh Market]
    end
    AgentChannel --> AgentAPI[Agent API / BFF]
    MarketChannel --> MarketAPI[Market API / BFF]
    AgentAPI --> API[Dossier Core API]
    MarketAPI --> API
    API --> Domain[Hồ sơ + Checklist + Pipeline]
    Domain -->|Ghi nghiệp vụ và outbox| DB[(PostgreSQL)]
    Relay[Outbox Relay & Scheduler] -->|Đọc bản ghi chờ xử lý| DB
    Relay --> Kafka[(Kafka)]
    Relay --> Message[Message Delivery]
    API --> File[File Management]
    API --> OCR[vhm-ocr-ekyc]
    OCR --> File
    API --> Enterprise[TTOL]
```

### 2.2.3 Sơ đồ kiến trúc pipeline nội bộ

Pipeline Social Housing là cấu hình có phiên bản được nạp cùng ứng dụng. Khi tạo hồ sơ, core ghi phiên bản pipeline và trạng thái khởi tạo. Command Market được kiểm tra channel/data-subject ownership và state; command Agent/Back Office được kiểm tra BUS role, scope, pipeline ownership và guard nghiệp vụ. Projection, history, reviewer và outbox được cập nhật trong một transaction. Không có process instance Camunda và không có ranh giới eventual consistency với workflow engine ngoài.

### 2.2.4 Phân định trách nhiệm module

| **Khối kiến trúc** | **Trách nhiệm** | **Dữ liệu quản lý** | **Không chịu trách nhiệm** |
| --- | --- | --- | --- |
| Agent API | BFF Agent/Back Office; xác thực actor, BUS role/project scope, public authorization và ký actor context | Không sở hữu dossier aggregate hoặc tự định nghĩa role | State/invariant, pipeline mapping và persistence cuối. |
| Market API | BFF Market; xác thực customer, data-subject/object authorization và ký subject context | Không sở hữu dossier aggregate hoặc role catalogue | State/invariant, signed subject binding và persistence cuối. |
| Dossier domain | Vòng đời hồ sơ, validation, duplicate, checklist, pipeline, phân công và điều phối OCR | Aggregate dossier và projection liên quan | Identity kênh, file binary, OCR raw. |
| Outbox/Scheduler | Phát sự kiện, gửi notification và nhắc SLA sau commit | Trạng thái delivery/dedup | Thay đổi quyết định nghiệp vụ đã commit. |
| `vhm-ocr-ekyc` | Điều phối prepare-upload với File Management, quản lý tài nguyên OCR bất đồng bộ và chuẩn hóa kết quả | OCR lifecycle, media reference, request/kết quả OCR đã mã hóa | Sở hữu dossier, giữ byte media, xử lý tài liệu không OCR hoặc tự áp kết quả vào form. |
| File Management | Upload, kiểm tra, lưu trữ và download file; nhánh không OCR do Core gọi trực tiếp, nhánh OCR do `vhm-ocr-ekyc` điều phối | File binary/object và metadata thuộc contract File | Sở hữu dossier, OCR lifecycle hoặc kết quả OCR. |
| Enterprise services | TTOL, Message Delivery | Dữ liệu thuộc từng miền | Sở hữu aggregate dossier. |

### 2.2.5 Ranh giới tin cậy

| **Ranh giới** | **Mức tin cậy** | **Kiểm soát** | **Khoảng trống** |
| --- | --- | --- | --- |
| Customer → Market API | Không tin cậy | Authentication, data-subject/object ownership, validation và rate limit; không dùng business role | Market public boundary. |
| Agent/BO → Agent API | Không tin cậy | Authentication, BUS role, project/team scope, validation và rate limit | Agent public boundary. |
| Agent API/Market API → Core | Zero Trust nội bộ | Client identity riêng, Basic Auth, HMAC, timestamp/nonce/body hash, actor signature | Cần allowlist và secret rotation độc lập cho từng BFF. |
| Market subject context → business | Chỉ tin sau verify | `channel=MARKET`, actor subject, data-subject binding, expiry, JTI; không chứa Agent roles | Core kiểm tra client/channel và subject–dossier binding. |
| Agent actor context → business | Chỉ tin sau verify | `channel=AGENT`, actor subject, BUS roles, project/team scope, visibility, expiry, JTI | Role/scope không nhận từ request body. |
| Core → `vhm-ocr-ekyc` | Zero Trust nội bộ | Workload identity, audience/scope, object context, idempotency | OpenAPI/IAM L3 và E2E chưa hoàn tất. |
| Core → File Management | Dependency ngoài process | Workload identity, object scope, checksum và owner/upload grant | Chỉ dành cho tài liệu không OCR. |
| Core → TTOL/Message | Dependency ngoài process | Credential server-side, timeout/retry/config | Contract/SLA cần chốt. |
| `vhm-ocr-ekyc` → File Management | Ranh giới được ủy quyền | Workload identity và object scope của media OCR | Core không tham gia contract File của nhánh OCR. |
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

Sơ đồ trên là snapshot của Social Housing pipeline v1, không phải giới hạn cố định ba cấp duyệt. Việc thêm hoặc bỏ một cấp duyệt được thực hiện bằng pipeline version mới; API vẫn trả stage timeline và `availableActions` theo definition đã gắn với từng hồ sơ.

## 2.4 Tính nhất quán và Idempotency

### 2.4.1 Tạo tài nguyên và idempotency

- Agent API gán `source=AGENT`; Market API gán `source=MARKET`. MARKET bắt buộc header `Idempotency-Key` (`10509`), còn AGENT không bắt buộc.
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
| OCR bất đồng bộ | Tạo/poll không ghi PII vào dossier; chỉ domain apply metadata không PII sau khi kiểm tra kết quả | `vhm-ocr-ekyc` quản lý media reference, lifecycle và kết quả chuẩn; File Management giữ byte media |

### 2.4.4 Concurrency guards

- `@Version` và `If-Match` ngăn lost update khi caller cung cấp version; bỏ header đồng nghĩa chấp nhận last-write-wins ở API hiện hành.
- Unique partial index ngăn race hồ sơ active trùng `subjectRef+dự án`; conflict ánh xạ `11011`.
- Unique partial index căn được cấp ngăn hai hồ sơ active giữ cùng căn.
- Reviewer assignment PKD dùng row lock/rotation trong DB; PTT/SXD dùng atomic counter Redis và sticky assignment khi quay lại.
- Command pipeline kiểm tra current state/action trong transaction; mọi side effect chỉ commit khi toàn bộ guard thành công.

# 3. Functional Requirements

## 3.1 Ma trận năng lực chức năng

| **ID** | **Năng lực/yêu cầu** | **Thiết kế** | **Mức bắt buộc** |
| --- | --- | --- | --- |
| FR-01 | Tạo hồ sơ nháp | Agent API hoặc Market API gọi Core create `SOCIAL_HOUSING`; luôn trả `DRAFT` | `BẮT BUỘC` |
| FR-02 | Upload tài liệu | Core kiểm tra quyền; tài liệu không OCR gọi File Management, media OCR gọi `vhm-ocr-ekyc`; kênh PUT theo presigned contract | `BẮT BUỘC` |
| FR-03 | Cập nhật snapshot | Chỉ business snapshot allowlist không chứa OCR/PII; applicant/contact không thuộc persistence API và không được forward sang `vhm-ocr-ekyc` | `BẮT BUỘC` |
| FR-04 | Nộp hồ sơ | Command `SUBMIT`, duplicate guard, checklist readiness, sinh mã | `BẮT BUỘC` |
| FR-05 | Chống trùng | Application precheck + database race guard | `BẮT BUỘC` |
| FR-06 | Checklist/progress | Snapshot theo template+group; counts/missing/invalid | `BẮT BUỘC` |
| FR-07 | Phê duyệt nhiều cấp | Pipeline versioned thực thi trong core | `BẮT BUỘC` |
| FR-08 | Phân công reviewer | Manual/reassign/claim + auto round-robin | `BẮT BUỘC` |
| FR-09 | Cấp/thu hồi căn | Atomic allocation/approve; unique active unit | `BẮT BUỘC` |
| FR-10 | Hồ sơ giấy | Submit/confirm hardcopy và timeline | `BẮT BUỘC` |
| FR-11 | OCR CCCD | Dossier bind `subjectRef` authoritative, gọi `vhm-ocr-ekyc` tạo `202`, poll `/ocr/result` và xử lý `CONFIRM_AND_APPLY` tại domain; không gọi OCR-confirm riêng và không persist media/kết quả OCR | `BẮT BUỘC` |
| FR-12 | Notification/reminder | Outbox, kênh được phê duyệt, T+6/T+18 và manual trigger | `BẮT BUỘC` |
| FR-13 | Tra cứu/báo cáo | List/detail/statistics/export tại Core chỉ dùng business metadata allowlist; tra cứu theo contact và hydrate applicant PII không thuộc baseline Dossier–OCR hiện tại | `BẮT BUỘC` |
| FR-14 | Xóa bản nháp | Chỉ DRAFT; xóa checklist và dữ liệu phụ thuộc | `BẮT BUỘC` |
| FR-15 | PIC hồ sơ | Use case, permission và audit semantics | `TBD` |
| FR-16 | Thêm/bớt cấp duyệt | Công bố pipeline version mới; không đổi schema DB/public API và không làm đổi luồng hồ sơ đang chạy | `BẮT BUỘC` |
| FR-17 | Phân quyền theo kênh | Market dùng authenticated data-subject ownership tại Market API; Agent/Back Office dùng BUS role/scope | `BẮT BUỘC` |

## 3.2 Quy tắc nghiệp vụ

| **ID** | **Quy tắc** |
| --- | --- |
| BR-01 | Public create không được submit; trạng thái sau create luôn là `DRAFT`. |
| BR-02 | Agent API gán `source=AGENT`, Market API gán `source=MARKET` từ channel context tin cậy; MARKET create phải có `Idempotency-Key`, AGENT không bắt buộc key. |
| BR-03 | Một cặp `subjectRef` authoritative + dự án chỉ có một hồ sơ chưa terminal; DRAFT `{}` được phép chưa có `subjectRef`, nhưng bind reference và submit phải kiểm tra duplicate. Core không dùng CCCD rõ làm khóa. |
| BR-04 | Create `{}` không tạo checklist; create có `documents[]` và mọi full update DRAFT/ADD_INFO phải synchronize checklist. |
| BR-05 | Identity checklist là `(dossierId, documentTemplateId, groupCode)`. |
| BR-06 | Submit cần tồn tại ít nhất một required item và mọi required item ở `COMPLETE`. |
| BR-07 | Full update business snapshot chỉ ở DRAFT/ADD_INFO; OCR/PII/contact không thuộc `formData` và luôn đi qua contract authoritative. |
| BR-08 | Update không được ghi đè `source`, `assignedUnitCode` hoặc `assignedUnitId` do server quản lý. |
| BR-09 | Media identity applicant/spouse được xác minh qua `vhm-ocr-ekyc`; `documents[].s3PathFile` không OCR được xác minh qua File Management khi file-validation bật. |
| BR-10 | Không kiểm tra file path prefix theo dossier ID. |
| BR-11 | Mọi command phải hợp lệ theo state và channel policy: Market cần signed data-subject ownership; Agent/Back Office cần BUS role, scope và ownership tương ứng. |
| BR-12 | Lần submit đầu sinh mã `<sapId>-<agencyId>-<ddMMyy>-<sequence4>`; PKD approve chuẩn hóa thành `<sapId>-<sequence5>`. |
| BR-13 | Reject/revoke unit phải giải phóng căn đang cấp. |
| BR-14 | DRAFT delete xóa checklist tường minh; FK checklist cũng có `ON DELETE CASCADE`. |
| BR-15 | Comment hiện là optional theo pipeline/config; không mô tả là bắt buộc nếu chưa bật guard. |
| BR-16 | Mỗi dossier được gắn bất biến với đúng `(pipelineCode, pipelineVersion)` trong suốt vòng đời, trừ migration có kiểm soát được phê duyệt. |
| BR-17 | Không sửa hoặc xóa definition đã có hồ sơ tham chiếu; ngừng dùng một cấp duyệt chỉ áp dụng cho pipeline version mới. |
| BR-18 | Hồ sơ `source=MARKET` phải bind `ownerSubject` opaque từ signed Market context tại create; field này server-owned, không nhận từ body và mọi Market access phải khớp binding. |

# 4. Non-Functional Requirements

Các mục tiêu `200 req/s`, P95 cụ thể và availability `99.9%` chưa có workload/capacity evidence được phê duyệt nên chưa được coi là baseline. Trước production phải chốt NFR theo workload thực tế.

| **ID** | **Nhóm** | **Yêu cầu** | **Cổng** |
| --- | --- | --- | --- |
| NFR-01 | Availability | Core stateless ở tầng HTTP; DB/Redis/Kafka/external có health và degradation policy | `BẮT BUỘC` |
| NFR-02 | Consistency | Không mất mutation đã commit; outbox cho event/notification | `BẮT BUỘC` |
| NFR-03 | Concurrency | Idempotency, optimistic lock, unique index, row/Redis lock | `BẮT BUỘC` |
| NFR-04 | Performance | P95/P99 theo endpoint và peak TPS phải được đo | TBD |
| NFR-05 | Scalability | Scale ngang API/relay; không dùng session cục bộ làm authority | `BẮT BUỘC` |
| NFR-06 | Security | Signed request/actor, least privilege, không persist/cache/log dữ liệu OCR/PII được proxy | `BẮT BUỘC` |
| NFR-07 | Privacy | Data minimization tại Core; purpose/retention/deletion của OCR/PII theo TDD `vhm-ocr-ekyc` | TBD/BẮT BUỘC |
| NFR-08 | Recoverability | PostgreSQL PITR, outbox replay, backup/restore drill | TBD/BẮT BUỘC |
| NFR-09 | Observability | Metrics/log/trace/correlation và alert có owner | `BẮT BUỘC` |
| NFR-10 | Maintainability | Versioned migration/schema/pipeline; backward-compatible API | `BẮT BUỘC` |
| NFR-11 | Workflow configurability | Thêm/bớt một cấp duyệt bằng definition mới, không cần đổi DB/public API; hồ sơ version cũ tiếp tục xử lý được | `BẮT BUỘC` |

## 4.1 NFR Target Values dự kiến

Các giá trị dưới đây là **DỰ KIẾN để sizing và thiết kế kiểm thử**, chưa phải SLO
production được phê duyệt. Latency HTTP không bao gồm thời gian client upload/download
binary hoặc thời gian provider thực hiện OCR; các khoảng đó được đo thành SLI riêng.
Target chỉ chuyển thành baseline khi Product cung cấp workload forecast và
load/soak/DR test đạt trên cấu hình production-like với data volume/index gần thực
tế.

| **ID/phạm vi** | **SLI/cách đo** | **Target dự kiến** | **Điều kiện/loại trừ** | **Cổng xác nhận baseline** |
| --- | --- | --- | --- | --- |
| NFR-T01 — Availability Core | Tỷ lệ valid HTTP request không trả `5xx` tại internal ingress theo tháng | **≥99,9%/tháng** | Không tính `4xx` hợp lệ và maintenance window được duyệt; availability end-to-end với File/OCR/TTOL/Message phải có SLI riêng | 30 ngày STAG/PROD telemetry, dependency SLA và error-budget policy được System Owner/Vận hành duyệt |
| NFR-T02 — Local read API | P95/P99 của list/detail/progress/timeline chỉ dùng Core DB/cache | **P95 ≤800 ms; P99 ≤1.500 ms** | Page mặc định 20, tối đa dự kiến 100; không gồm đọc kết quả OCR transient, download hoặc report generation | Load test theo filter/visibility/role, data volume dự báo và query plan không N+1/seq-scan ngoài budget |
| NFR-T03 — Mutation API | P95/P99 create/update/submit/pipeline action, gồm transaction và validation bắt buộc | **P95 ≤2.000 ms; P99 ≤4.000 ms** | Không gồm upload bytes/OCR processing; dependency call bắt buộc vẫn tính trong latency endpoint | Mixed-workload test với contention, file/OCR/project stub có latency/error distribution theo contract |
| NFR-T04 — External orchestration API | P95/P99 prepare-upload, OCR create và poll `/ocr/result` | **P95 ≤3.000 ms; P99 ≤8.000 ms** | Không gồm PUT/download binary và OCR background SLA; timeout/retry không được làm vượt end-to-end deadline | Contract/load test với real sandbox, connect/read timeout và `Retry-After`/degradation policy được chốt |
| NFR-T05 — Throughput | Mixed non-binary request rate tại Core, error/CPU/pool dưới ngưỡng | **≥200 req/s trong 30 phút; burst ≥400 req/s trong 5 phút** | Workload mix dự kiến: 70% read, 25% mutation, 5% integration/report initiation; `5xx <1%`, CPU trung bình <70%, DB pool <80% | Product xác nhận peak factor; load/soak test production-like và capacity/headroom review |
| NFR-T06 — Concurrency/integrity | Kết quả concurrent create/update/action và số invariant bị vi phạm | **100%** cùng actor/key/payload trả cùng dossier khi chạy **≥100 request đồng thời**; **0** duplicate active subject/project hoặc active unit; **0** lost update | Khác actor/cùng key hoặc cùng key/khác payload phải bị từ chối; optimistic conflict là business outcome, không tính platform error | Concurrency test trên PostgreSQL thật, unique/advisory/optimistic guard và reconciliation query |
| NFR-T07 — Event outbox | Thời gian từ business commit đến Kafka publish khi broker healthy | **P95 ≤10 giây; P99 ≤60 giây; 0 event mất** | At-least-once nên duplicate được phép nhưng consumer phải idempotent; maintenance/outage đo recovery riêng | Load/failure/replay test với relay interval 2 giây, batch 500, Kafka ACL/quota production-like |
| NFR-T08 — Notification/reminder | Commit intent đến delivery acceptance; độ lệch reminder so với due time | Notification **P95 ≤60 giây, P99 ≤5 phút**; reminder drift **≤5 phút** khi dependency healthy | Không cam kết delivery tới inbox/thiết bị cuối; business transition không rollback vì notification lỗi | Message contract/quota test, backlog replay và scheduler failover/duplicate drill |
| NFR-T09 — Report/export | Thời gian tạo và lưu artefact cho dataset giới hạn | Tối đa dự kiến **10.000 rows/file**, hoàn tất **≤120 giây**, tối đa **5 job đồng thời/instance** | Export lớn hơn phải chunk/async/quota; không giữ file tạm quá TTL và không tính download time | Capacity test Syncfusion/POI với template thật, memory/temp-disk limit và File Management latency |
| NFR-T10 — Recoverability DB | RPO/RTO từ PITR/restore/failover drill | **RPO ≤5 phút; RTO Core ≤60 phút** | Dossier, pipeline/history/checklist/reviewer/outbox phải phục hồi cùng consistency point; File/OCR theo SLA ngoài | DBA/System Owner/Vận hành phê duyệt và restore/reconciliation drill đạt |
| NFR-T11 — Security | Tỷ lệ request PROD có workload + signed actor/subject hợp lệ; release vulnerability | **100%** request bắt buộc được xác thực/chống replay; **0 Critical** và **0 High exploitable** chưa có risk acceptance | Local bypass không áp dụng STAG/PROD; `4xx` denial đúng policy không phải availability error | Security/IDOR/replay test, SAST/SCA/container/IaC scan và ANBM sign-off |
| NFR-T12 — Privacy/data leakage | Phát hiện raw OCR/PII trong Core persistence/cache/event/log/APM/report staging | **0 phát hiện**; **100%** response PII dùng field projection/masking/no-store theo contract | Opaque references/business metadata allowlist vẫn được phép; scan phải bao phủ backup mẫu và DLQ | Automated data scan + SIT/E2E negative test + ANBM/Privacy approval trước dữ liệu thật |
| NFR-T13 — Observability | Correlation/metric/trace coverage và thời gian phát hiện lỗi nghiêm trọng | **100%** request có correlation ID; dashboard/metric cho mọi dependency; alert P1/P2 phát trong **≤5 phút** | Trace sampling được phép nhưng security/error trace phải đủ; body/PII/token/file URL luôn loại bỏ | Dashboard/alert synthetic test, on-call routing và incident drill |
| NFR-T14 — Workflow evolution | Regression khi thêm/bớt một stage bằng definition mới | **100%** hồ sơ cũ tiếp tục chạy version đã pin; **0** DB/public API change chỉ vì thêm/bớt stage | Side effects assignment/reminder/notification/report/audit phải dùng stage policy, không hard-code tên stage | Pipeline validation, backward-compatibility và E2E hai version chạy song song |

Target page size, report concurrency, workload mix và timeout ở bảng trên là giả
định ban đầu, không phải lý do hard-code. Giá trị production phải externalize khi
phù hợp, có upper bound server-side và được cập nhật đồng thời ở mục 11, 13, 14
sau khi baseline được phê duyệt.

# 5. Technology Stack & Justification

| **Công nghệ** | **Vai trò** | **Cơ sở lựa chọn/hệ quả** |
| --- | --- | --- |
| Java 25, Spring Boot 4.1 | Runtime/service framework | Stack nền tảng của dịch vụ; cần image/JVM production được chứng nhận. |
| Spring Data JPA/Hibernate | Aggregate persistence, optimistic lock | Phù hợp transaction domain; cần tránh N+1 và giữ `open-in-view=false`. |
| PostgreSQL, Liquibase | Source of truth, JSONB, constraint, advisory lock | Hỗ trợ transaction/partial index; migration phải forward-safe. |
| Redis | Replay/nonce, counter phân công, cache/lock hỗ trợ | Không được là source of truth của dossier. |
| Kafka | Phát domain event từ outbox | At-least-once; consumer cần idempotent. |
| Caffeine | Cache cục bộ cho dữ liệu tham chiếu | Giảm latency; cần invalidation/TTL rõ ràng. |
| JSON Schema 2020-12 | Validate form Social Housing v1 | Cho phép evolution schema; enforcement hiện phụ thuộc config. |
| Thrift/HTTP clients | File Management, `vhm-ocr-ekyc` và các tích hợp nội bộ | Tách client/config theo ranh giới; contract bên ngoài cần timeout/retry/test. |
| Syncfusion/Apache POI | Export/tạo tài liệu | Cần quản lý license, template và kiểm thử output. |

## 5.1 ADR Log

ADR chi tiết nằm tại Phụ lục D. Các quyết định nền tảng: modular monolith, pipeline nội bộ bằng YAML, PostgreSQL source of truth, snapshot JSONB có schema version, transactional outbox, signed actor context, database constraint là race guard cuối.

## 5.2 Trade-off Analysis

`Phương án được chọn` là baseline thiết kế của TDD này; trạng thái phê duyệt chính
thức và điều kiện còn mở được quản lý tại Phụ lục D. Khi workload, contract hoặc
yêu cầu nền tảng thay đổi, quyết định liên quan phải được đánh giá lại thay vì đổi
stack ngầm trong implementation.

| **ID/ADR** | **Vấn đề cần quyết định** | **Phương án A (Ưu–Nhược)** | **Phương án B (Ưu–Nhược)** | **Phương án được chọn** | **Lý do/đánh đổi còn lại** |
| --- | --- | --- | --- | --- | --- |
| BASELINE-01 | Runtime và application framework | **Java 25 + Spring Boot 4.1** — Ưu: phù hợp baseline VHM, hệ sinh thái transaction/security/observability và năng lực đội ngũ. Nhược: footprint JVM, startup/memory và image/JDK mới cần được chứng nhận. | **Go/Node.js hoặc runtime khác** — Ưu: có thể nhẹ hơn cho API stateless. Nhược: lệch baseline, phải xây lại domain/persistence/security integration và tăng stack vận hành. | Phương án A | Đây là baseline tổ chức, không phải tối ưu bằng benchmark riêng của Dossier; production image, GC/memory và compatibility phải được kiểm thử tải/chứng nhận. |
| TECH-01 | Cách truy cập PostgreSQL | **Spring Data JPA/Hibernate cho aggregate, native/JDBC cho truy vấn lock/outbox đặc thù** — Ưu: mapping domain, transaction và optimistic lock thuận tiện nhưng vẫn giữ quyền kiểm soát SQL nóng. Nhược: hai style persistence, nguy cơ N+1/flush ngoài ý muốn và cần review query plan. | **SQL-first hoàn toàn bằng jOOQ/JDBC** — Ưu: SQL minh bạch, batch/query phức tạp dễ tối ưu. Nhược: tăng mapping/boilerplate và trách nhiệm quản lý aggregate/version thủ công. | Phương án A có điều kiện | Use case mutation phù hợp aggregate JPA; outbox, advisory/row lock và report được phép dùng SQL chuyên biệt. Bắt buộc `open-in-view=false`, query-count test và explain plan cho endpoint nóng. |
| ADR-001 | Source of truth cho dossier/pipeline/history/outbox | **PostgreSQL + Liquibase** — Ưu: ACID, constraint/partial index, JSONB, advisory lock và outbox trong cùng transaction. Nhược: cần HA/PITR, migration forward-safe, capacity/index/vacuum được vận hành chặt. | **Document/NoSQL store** — Ưu: scale ngang và document model tự nhiên hơn cho form linh hoạt. Nhược: business invariant, join/report, optimistic concurrency và atomic outbox phức tạp hơn; thêm một stack dữ liệu. | Phương án A | Invariant hồ sơ/cấp căn/checklist và ranh giới commit quan trọng hơn lợi ích scale document store; JSONB đáp ứng phần snapshot linh hoạt. |
| ADR-002/ADR-013 | Thực thi workflow và khả năng thêm/bớt cấp duyệt | **Pipeline YAML versioned chạy trong modular monolith** — Ưu: transition và dossier commit nguyên tử, ít hạ tầng, hồ sơ pin đúng version. Nhược: Core tự chịu validation/tooling/migration/observability workflow và không có UI BPMN sẵn có. | **Camunda/Zeebe hoặc processor service riêng** — Ưu: engine/tooling workflow, timer và visualization chuyên dụng; cô lập scale/runtime. Nhược: distributed consistency, thêm deployable/contract và chi phí vận hành; tăng rủi ro lệch trạng thái Core–engine. | Phương án A | UC-01–UC-06 ở modular monolith; chưa có bằng chứng cần engine/service riêng. Definition mới phải immutable, có validation gate và giữ được hồ sơ version cũ. |
| ADR-004 | Mô hình lưu form Social Housing | **JSONB snapshot + `schemaVersion` + JSON Schema** — Ưu: evolve form ít migration cột, giữ snapshot theo thời điểm và hỗ trợ nhiều product version. Nhược: query/index/constraint field-level khó hơn, payload dễ nhận field ngoài contract nếu guard lỏng. | **Bảng/cột strongly typed cho từng trường form** — Ưu: type/constraint/query/report rõ ở DB. Nhược: migration dày, coupling release với thay đổi biểu mẫu và số bảng/cột tăng theo product/version. | Phương án A | Form thay đổi theo policy/product nhưng invariant lõi vẫn được đưa thành cột/index riêng. Structural denylist, versioned schema, size limit và index allowlist là điều kiện bắt buộc. |
| TECH-02 | Replay/nonce, counter, lock hỗ trợ và cache phân tán | **Redis cho state ngắn hạn/atomic TTL; PostgreSQL vẫn là authority** — Ưu: nonce/replay TTL, counter và coordination nhanh; giảm tải DB. Nhược: thêm HA/degradation/runbook và có thể fail security nếu dùng sai fallback. | **Chỉ dùng PostgreSQL** — Ưu: giảm dependency và một nguồn vận hành. Nhược: tăng contention/cleanup cho nonce-counter, latency cao hơn và trộn state ngắn hạn với business data. | Phương án A có điều kiện | Redis phù hợp dữ liệu có thể rebuild hoặc security state có fail-closed policy; không được quyết định dossier ownership/status hay thay race guard DB. |
| ADR-007 | Nhất quán giữa mutation và phát event/notification | **Transactional outbox + Kafka/relay** — Ưu: business row và intent commit cùng transaction, replay được và scale consumer. Nhược: eventual consistency, at-least-once, backlog/cleanup/duplicate phải được vận hành. | **Publish Kafka trực tiếp hoặc gọi downstream đồng bộ sau DB write** — Ưu: ít bảng/relay và latency thấp hơn khi thành công. Nhược: dual-write gap, request bị phụ thuộc downstream và khó phục hồi khi DB commit nhưng publish/call lỗi. | Phương án A | Không chấp nhận mất intent sau commit. Consumer/relay phải idempotent, có retry/DLQ, lag alert, retention và reconciliation runbook. |
| TECH-03 | Cache dữ liệu tham chiếu trong process | **Caffeine TTL-bounded cho dữ liệu read-mostly** — Ưu: latency thấp, không thêm network hop. Nhược: cache khác nhau giữa pod, invalidation chậm và mất khi restart. | **Redis-only hoặc không cache** — Ưu: shared view giữa pod hoặc luôn đọc authoritative. Nhược: network/dependency/latency cao hơn; Redis cache vẫn có bài toán invalidation. | Phương án A có điều kiện | Chỉ cache reference không phải authority, không chứa OCR/PII và chấp nhận stale có giới hạn. Role/scope/ownership, pipeline version đã pin và security decision không được dựa vào local cache thiếu freshness contract. |
| ADR-009/ADR-010/ADR-012 | Sở hữu file và OCR CCCD | **Dùng File Management và `vhm-ocr-ekyc`, Core chỉ giữ reference/metadata tối thiểu** — Ưu: tập trung media/PII/provider/credential, giảm binary và dữ liệu nhạy cảm tại Core. Nhược: thêm network hop, phụ thuộc SLA/contract/authorization/delete orchestration và khó chứng minh ownership nếu dependency thiếu claim. | **Core lưu file/OCR result hoặc tích hợp provider trực tiếp** — Ưu: ít hop và Core tự chủ luồng. Nhược: nhân bản PII/media, tăng storage/key/provider scope, coupling và chi phí compliance/vận hành. | Phương án A | Capability dùng chung là ranh giới authoritative cho OCR; File Management giữ byte media. Trước production cần OpenAPI L3, owner/upload-grant, timeout/degradation, retention/delete và E2E cross-owner; không fallback sang bản sao local. |
| TECH-04 | Sinh báo cáo/tài liệu | **Syncfusion/Apache POI trong Core** — Ưu: đáp ứng template Excel/DOCX hiện hữu và giữ business projection gần domain. Nhược: license, memory/CPU, xử lý template/file không tin cậy và coupling release template với service. | **Document/report service chuyên biệt** — Ưu: cô lập tài nguyên, template và security sandbox; tái sử dụng đa domain. Nhược: thêm service/contract, network hop và consistency của dữ liệu export. | Phương án A cho phạm vi hiện tại | Chưa có capability tài liệu authoritative được chốt. Phải pin version/license, giới hạn row/file/time/memory, sanitize template value và chuyển sang async/service riêng nếu capacity test vượt budget Core. |

# 6. Integration Architecture

## 6.1 Danh mục giao diện tích hợp

| **ID** | **Tích hợp** | **Hướng** | **Kiểu** | **Mục đích** | **Failure policy** |
| --- | --- | --- | --- | --- | --- |
| INT-01 | Agent API → Dossier Core | Inbound | HTTP sync | Agent/Back Office registration/list/detail/action với signed BUS role/scope, `source=AGENT` | Signature/role/scope fail closed |
| INT-02 | Market API → Dossier Core | Inbound | HTTP sync | Market registration/list/detail/action với signed data-subject context, `source=MARKET`; không business role | Signature/subject binding fail closed |
| INT-03 | Dossier Core → File Management | Outbound | Client sync | Tài liệu không OCR: prepare upload, existence/ownership, download và lưu artefact xuất | Fail hard cho validation bắt buộc |
| INT-04 | Dossier Core → `vhm-ocr-ekyc` | Outbound | HTTP async resource | Prepare media, tạo OCR với `subjectRef` đầu vào và poll `/ocr/result`; không có OCR-confirm riêng, không persist bản sao | Idempotent create, polling hữu hạn, không retry mù |
| INT-05 | TTOL | Outbound | HTTP sync | Lấy danh sách nhân sự PTT/SXD theo dự án và vai trò để auto-assign reviewer | Best effort; manual assignment fallback |
| INT-06 | Message Delivery | Outbound | Outbox relay | Email notification | Retry/backoff → FAILED |
| INT-07 | Kafka | Outbound | Async | Domain event đã commit | Outbox retry; publish có feature flag |

## 6.2 Contract API hồ sơ VHM

### 6.2.1 Registration flow qua Agent API/Market API

Kênh Agent/Back Office đi qua Agent API; kênh Market đi qua Market API. Hai BFF ngang hàng sử dụng cùng internal business API nhưng mang security context khác nhau. Agent API gửi signed actor context chứa BUS role/scope; Market API gửi signed data-subject context và không gửi Agent roles. Mỗi BFF gán `source` tương ứng từ channel context đã xác thực rồi gọi Core bằng workload identity riêng; client không được tự chọn `source`, role, scope, owner hoặc data subject trong request body.

| **Bước** | **API Agent/Market** | **Kết quả bắt buộc** |
| --- | --- | --- |
| 1 | `POST /v1/social-housing/registrations` | Tạo duy nhất một `DRAFT`, nhận `dossierId`. |
| 2 | `POST /v1/social-housing/registrations/{id}/prepare-upload` rồi PUT file | Chỉ cấp URL sau khi caller đọc được dossier; giữ `s3PathFile`. |
| 3 | `PATCH /v1/social-housing/registrations/{id}` | Ghi snapshot form/documents đầy đủ. |
| 4 | `POST /v1/social-housing/registrations/{id}/submit` | Chuyển vào pipeline nếu mọi guard đạt. |

### 6.2.2 Nhóm năng lực API nội bộ

Core công bố API nội bộ có version cho các nhóm năng lực: quản lý hồ sơ, command pipeline, quyết định tài liệu, quyền dự án, notes/hardcopy, download/export, statistics và reminder. Public contract không phản chiếu nguyên xi endpoint nội bộ; Agent API và Market API chịu trách nhiệm map DTO, HTTP semantics và ẩn cấu trúc nội bộ. Danh sách path/field đầy đủ thuộc OpenAPI L3, không lặp lại trong tài liệu L2 này.

### 6.2.3 Envelope và phân trang

Core trả `ServiceResponse { code, message, data }`, trong đó `code=0` là thành công. Danh sách dùng `PageDto { items, pagination }`, page number là 1-based. HTTP status và application error code phải cùng biểu đạt một kết quả; client không được chỉ dựa vào message tiếng Việt.

## 6.3 Contract tài liệu và phân tuyến file/OCR

### 6.3.1 Phân loại tài liệu

| **Nhóm tài liệu** | **Ví dụ trong hồ sơ NOXH** | **Có xử lý OCR** | **Năng lực sử dụng** |
| --- | --- | --- | --- |
| Media định danh | Mặt trước/sau CCCD của người đăng ký và vợ/chồng | Có | `Dossier Core → vhm-ocr-ekyc → File Management`; tạo tài nguyên OCR. |
| Giấy tờ checklist | File gắn với `documents[].s3PathFile` | Không mặc định | `Dossier Core → File Management`; upload, xác minh ownership/tồn tại, duyệt và tải xuống. |
| Gói hợp đồng | Mẫu hợp đồng/phiếu được render và đóng gói ZIP | Không | `Dossier Core → File Management`; lưu artefact và cấp quyền tải xuống. |
| Gói tài liệu đính kèm | ZIP tổng hợp giấy tờ của hồ sơ | Không | `Dossier Core → File Management`; đọc source hợp lệ, lưu snapshot ZIP và tải xuống. |
| Báo cáo NOXH | File XLSX kết quả tra cứu/xuất báo cáo | Không | `Dossier Core → File Management`; lưu báo cáo và cấp quyền tải xuống có kiểm soát. |

Quy tắc phân tuyến do use case phía server quyết định. Client không được chọn `useOcr` để đổi đường tích hợp. Media định danh hoặc tài liệu được nghiệp vụ chỉ định mới đi qua `vhm-ocr-ekyc`; checklist, ZIP và XLSX không OCR gọi File Management trực tiếp.

### 6.3.2 Luồng tài liệu không OCR

- Request chuẩn bị upload phải chứa dossier ID hợp lệ và metadata file được allowlist; Core kiểm tra quyền hồ sơ trước khi gọi File Management.
- Upload contract gồm presigned URL ngắn hạn, headers bắt buộc và `s3PathFile`; kênh PUT bytes trực tiếp tới object URL được cấp.
- Core chỉ persist opaque reference như `s3PathFile`; không giữ presigned URL và không dùng path prefix làm bằng chứng ownership.
- Khi create/update/attach, Core gọi File Management để xác minh batch existence, owner hoặc upload grant. Existence đơn thuần không đủ để cho phép attach.
- ZIP/XLSX do Core sinh ra được upload và cấp download URL qua File Management; cùng authorization/visibility của dossier áp dụng cho export/download.
- MIME, size, magic bytes, checksum, TTL, retention và quyền download phải được chốt trong File Contract L3.

### 6.3.3 Media dùng cho OCR

Media OCR đi theo luồng `Kênh → Agent API/Market API → Dossier Core → vhm-ocr-ekyc → File Management`. Dossier Core xác thực quyền hồ sơ và gửi context nghiệp vụ tới `vhm-ocr-ekyc`; service này sở hữu bước chuẩn bị/lấy media với File Management và vòng đời OCR. Core không gọi File Management trực tiếp cho media thuộc request OCR.

Phạm vi Dossier chỉ dùng hai role CCCD `DOCUMENT_FRONT`, `DOCUMENT_BACK` và MIME được OCR Contract L3 allowlist trong tập JPEG/PNG/PDF. Các role khác của capability dùng chung không thuộc luồng Dossier. Dossier không được mở rộng role/MIME hoặc tự dựng object path nếu chưa cập nhật contract OCR L3.

## 6.4 Contract OCR dùng chung

### 6.4.1 Ranh giới tích hợp

Dossier Core là consumer duy nhất của `vhm-ocr-ekyc` trong luồng hồ sơ NOXH và gọi service này bằng workload identity. Capability OCR điều phối media OCR với File Management, lưu media reference, quản lý lifecycle, processor, provider credential, polling provider, dữ liệu OCR thô và chính sách bảo vệ kết quả OCR; byte media nằm tại kho riêng tư do File Management quản lý. Các yêu cầu đó thuộc [L2 - VHMKDO2O - Capability OCR dùng chung](https://vin3s.atlassian.net/wiki/spaces/VARW/pages/3014268156/L2+-+VHMKDO2O+-+D+ch+v+OCR+eKYC), không được định nghĩa lại trong TDD Dossier. Agent API và Market API không gọi trực tiếp OCR service; Dossier Core chỉ biết contract OCR VHM và các opaque reference, không biết contract provider. Phạm vi tích hợp chỉ gồm OCR CCCD hai mặt.

Tích hợp OCR trực tiếp từ dossier tới provider không được phép trong kiến trúc mục tiêu. Mọi đường OCR legacy phải được loại bỏ hoặc vô hiệu hóa trước go-live sau khi E2E với `vhm-ocr-ekyc` đạt quality gate.

### 6.4.2 Tạo tài nguyên OCR

Contract mục tiêu dùng `POST /ocr` với `Idempotency-Key` bắt buộc và trả HTTP `202` cùng `Retry-After`. Context tối thiểu:

| **Trường** | **Ý nghĩa/kiểm soát** |
| --- | --- |
| `source` | `DOSSIER`; dùng cho authorization, quota và idempotency scope. |
| `referenceId` | Dossier ID dạng opaque business reference. |
| `requestBy` | Opaque actor reference, không nhúng PII. |
| `subjectRef` | Opaque applicant/customer reference do nguồn định danh domain có thẩm quyền cấp/bind server-side trước lời gọi; Dossier truyền vào OCR và OCR lưu để tương quan/phân quyền. Client không được khai trong form. |
| `channel`, `platform` | Context kênh đã allowlist. |
| `documentType`/loại OCR | `NATIONAL_ID` hoặc contract CCCD hai mặt được đóng băng ở OpenAPI L3. |
| `s3PathFile`/media roles | Chỉ reference do service chấp nhận; không gửi presigned URL vào DB/event. |

Cùng idempotency key và cùng request trả tài nguyên hiện hữu; cùng key nhưng request khác trả `409`. Provider không phải tham số do dossier/client lựa chọn.

### 6.4.3 Vòng đời và áp dụng kết quả

Trạng thái OCR gồm `QUEUED`, `PROCESSING`, `COMPLETED`, `FAILED`, `EXPIRED`. Agent API hoặc Market API thăm dò qua API của Dossier Core; Dossier Core gọi `/ocr/result` của `vhm-ocr-ekyc` và trả trạng thái chuẩn về BFF tương ứng. `nextAction` hướng dẫn `POLL`, `RETRY` hoặc `CONFIRM_AND_APPLY`.

Khi `COMPLETED`, Dossier Core có thể proxy projection kết quả chuẩn từ `/ocr/result` để kênh cho người dùng kiểm tra. `CONFIRM_AND_APPLY` là hành động của domain: BFF gửi `ocrId` về Dossier Core, Core đọc lại `/ocr/result`; OCR capability authorize tài nguyên theo `source`, `referenceId`, `requestBy` và `subjectRef` đã lưu, còn Core kiểm tra `COMPLETED` và dossier/subject binding local trước khi chỉ áp dụng readiness/metadata không PII. Baseline OCR hiện tại **không có endpoint confirm riêng** và Dossier không giả định OCR lưu lịch sử xác nhận nghiệp vụ.

Media reference, dữ liệu trích xuất, confidence, outcome, `ocrId` và provider metadata được lưu tại OCR capability; byte media tại File Management. Dossier không PATCH/copy các trường này vào snapshot. `subjectRef` đã được domain bind trước khi tạo OCR, được gửi trong `POST /ocr` và không phải giá trị OCR phát hành sau khi hoàn tất.

### 6.4.4 Tính nhất quán và failure semantics

- `vhm-ocr-ekyc` tạo request, media refs và outbox trong một transaction rồi mới trả `202`.
- Kafka chỉ chứa OCR ID tối thiểu; worker idempotent và trạng thái terminal bất biến.
- Dossier Core không retry mù khi kết quả submit không rõ; caller gửi lại cùng idempotency key/`ocrId` và Core đối soát trực tiếp với nguồn authoritative, không lưu bản sao trạng thái.
- OCR thất bại/timeout không tự động biến hồ sơ thành `REJECTED`; người dùng có thể retry theo policy hoặc nhập/đối chiếu thủ công.
- Các giới hạn MIME, size, checksum, retention và deadline dùng đúng baseline của tài liệu L2 `vhm-ocr-ekyc`; không định nghĩa lại khác trong dossier.
- Read/search theo họ tên/CCCD, batch hydrate, export PII, correction và delete API không thuộc contract OCR baseline hiện tại. Mọi nhu cầu tương lai phải mở rộng OpenAPI có version và được ANBM/Privacy phê duyệt; retention/delete hiện phối hợp bằng policy/runbook.
- OpenAPI L3 phải chốt cách truyền hoặc lưu liên kết `ocrId` để submit/readiness đọc đúng tài nguyên. Nếu Core cần persist, chỉ được lưu `ocrId` opaque tối thiểu; không lưu status/result/media reference và phải có retention/unlink policy.

## 6.5 Contract pipeline

### 6.5.1 Action contract

Các action chính gồm `UPDATE`, `SUBMIT`, `RESUBMIT`, `ASSIGN`, `REASSIGN`, `CLAIM`, `ALLOCATE_UNIT`, `APPROVE`, `REJECT`, `REQUEST_REVISION`, `RETURN_TO_SALES`, `SUBMIT_HARDCOPY`, `CONFIRM_HARDCOPY`, `REVOKE_UNIT`. Tập action khả dụng phải lấy từ response `availableActions`, không hard-code theo status ở kênh.

### 6.5.2 Channel policy, role và ownership

Pipeline không áp dụng một ma trận role chung cho cả hai kênh:

| **Principal** | **Điều kiện authorization** | **Không được làm** |
| --- | --- | --- |
| Market Customer | Market API đã xác thực customer và object ownership; Core verify signed `channel=MARKET`, data subject khớp dossier và action hợp lệ theo state | Không yêu cầu hoặc tự gán `APPLICANT_AGENT`; không dùng `ALL/TEAM/ASSIGNED`; không thực hiện reviewer/lead action. |
| Agent/Back Office | Agent API xác thực actor; Core kiểm tra signed BUS role, project/team scope, pipeline action và ownership | Không nhận role/scope từ body; role không tự mở quyền ngoài project/scope hoặc ownership. |

Catalogue minh họa hiện gồm `APPLICANT_AGENT`, `PKD`, `PKD_LEAD`, `PTT`, `PTT_LEAD`; BUS là authority chốt danh sách và semantics chính thức. Pipeline Definition map role BUS tới stage/action, không để Agent API tự diễn giải state machine.

Ownership policy gồm:

- `DATA_SUBJECT`: chỉ dùng cho Market; data subject trong signed context khớp owner subject đã bind server-side với dossier.
- `OWNER`: actor Agent tạo/được giao quản lý hồ sơ trong scope hợp lệ.
- `CLAIMER`: reviewer Agent/Back Office đang claim stage.
- `NONE`: không yêu cầu object ownership nhưng với Agent vẫn cần BUS role/scope; không dùng để mở quyền Market.

`availableActions` được Core tính theo state/invariant và policy tương ứng của channel. Market API có thể tiếp tục lọc UX nhưng không được mở rộng action ngoài response Core.

### 6.5.3 Pipeline selection

Tại create, pipeline phải được xác định bằng pipeline ID/version authoritative từ contract nghiệp vụ hoặc một rule selection duy nhất. Trường hợp không tìm thấy hoặc có nhiều kết quả phải bị từ chối bằng lỗi cấu hình rõ ràng; không chọn theo thứ tự khai báo.

### 6.5.4 Mô hình cấp duyệt có thể cấu hình

Một cấp duyệt không chỉ là một state `APPROVE`. Pipeline Definition L3 phải mô tả đầy đủ các policy sau để runtime và các consumer không phụ thuộc tên stage cụ thể:

| **Nhóm cấu hình** | **Nội dung bắt buộc** |
| --- | --- |
| Định danh và hiển thị | Stage code bất biến trong một version, thứ tự, nhãn hiển thị và business status ánh xạ. |
| Quyền xử lý | Market subject action/ownership policy; Agent reviewer/lead BUS role, scope, ownership rule và tập action được phép. |
| Phân công | `MANUAL`, `ROUND_ROBIN` hoặc `EXTERNAL_ROSTER`; nguồn candidate và chính sách giữ reviewer khi quay lại stage. |
| Quyết định | Đích của `APPROVE`, `REJECT`, `REQUEST_REVISION`, `RETURN` và các guard nghiệp vụ áp dụng. |
| Bổ sung hồ sơ | Stage nhận yêu cầu bổ sung, actor được cập nhật, stage tiếp nhận lại và điểm tiếp tục sau resubmit. |
| SLA và thông báo | Deadline/reminder, recipient policy và event/template intent theo transition. |
| Audit và hiển thị | Reviewer decision, lịch sử transition, timeline stage, report/export semantics. |

Kênh không hard-code chuỗi `SALES → PROCEDURE → SXD`; kênh render timeline theo danh sách stage và chỉ hiển thị action do Core trả. Database lưu stage code dạng tham chiếu và không tạo cột riêng cho từng cấp duyệt. Assignment, notification, reminder, audit và report phải dùng metadata/policy của pipeline thay vì rẽ nhánh theo tên stage.

Nếu một cấp không hỗ trợ yêu cầu bổ sung, definition có thể không khai báo `REQUEST_REVISION`. Nếu có hỗ trợ, definition bắt buộc chỉ rõ toàn bộ đường đi yêu cầu bổ sung và quay lại review; không suy luận theo cấp đứng trước hoặc đứng sau.

### 6.5.5 Phiên bản và thay đổi số cấp duyệt

| **Tình huống** | **Cách thực hiện bắt buộc** | **Ảnh hưởng hồ sơ đang chạy** |
| --- | --- | --- |
| Thêm một cấp duyệt | Tạo version mới, khai báo stage/state/role/assignment/transition/revision/SLA/notification, validate rồi mới activate | Không ảnh hưởng; hồ sơ cũ tiếp tục dùng version cũ. |
| Bỏ một cấp duyệt | Tạo version mới và nối lại rõ các transition/guard giữa hai phía của cấp bị bỏ | Không tự động bỏ qua stage của hồ sơ cũ. |
| Đổi role hoặc chính sách phân công | Tạo version mới nếu làm thay đổi quyền hoặc routing nghiệp vụ | Reviewer/history của version cũ được giữ nguyên. |
| Sửa nhãn không đổi semantics | Cho phép bản vá metadata có audit nếu policy quản trị chấp thuận; không đổi code/transition | Không thay đổi state hoặc quyền. |
| Chuyển hồ sơ đang chạy sang version mới | Chỉ dùng migration plan riêng có state mapping, reviewer mapping, precondition, dry-run, audit và rollback | Không hỗ trợ migration ngầm hoặc hàng loạt mặc định. |

Pipeline registry phải tra cứu chính xác bằng `(pipelineCode, pipelineVersion)`, cho phép nhiều version cùng tồn tại và chỉ có một version active cho create theo rule đã duyệt. Version cũ chỉ được ngừng phục vụ sau khi không còn hồ sơ active tham chiếu và retention/audit cho phép.

### 6.5.6 Cổng kiểm tra Pipeline Definition

Definition phải bị từ chối trước khi activate nếu vi phạm một trong các điều kiện: trùng stage/state/action trong cùng scope; initial state không tồn tại; transition trỏ tới state không tồn tại; stage/role reference không hợp lệ; có state không thể tới từ initial state; non-terminal state bị cụt; không có đường hợp lệ tới kết quả cuối; revision loop thiếu đường quay lại; business status mapping không hợp lệ; assignment/recipient policy thiếu dữ liệu bắt buộc; hoặc thay đổi một published version.

## 6.6 Mô hình checklist chuẩn hóa

| **Thuộc tính** | **Giá trị/ngữ nghĩa** |
| --- | --- |
| Identity | `dossierId + documentTemplateId + groupCode` |
| Upload status | `NOT_STARTED`, `COMPLETE` |
| Review status | `NOT_REVIEWED`, `VALID`, `INVALID` |
| Completed required | Upload `COMPLETE` và các constraint tài liệu không OCR đạt |
| Invalid item | Review/constraint tài liệu không OCR tương ứng không đạt |
| Progress | `completedRequired / requiredCount`, làm tròn 2 chữ số |

Khi file không OCR không đổi, synchronization giữ review state; khi path đổi hoặc bị xóa, trạng thái được reset; item không còn trong snapshot bị xóa. Duplicate `documentTemplateId + groupCode` trong cùng request bị từ chối. Readiness OCR được kiểm tra trực tiếp tại `vhm-ocr-ekyc` khi use case yêu cầu, không projection vào checklist.

## 6.7 Contract lỗi chuẩn

| **Code** | **Ý nghĩa** | **HTTP kỳ vọng** |
| --- | --- | --- |
| `10509` | Thiếu header bắt buộc, gồm idempotency cho MARKET | 400 |
| `11003` | Không tìm thấy dossier | 400/404 theo public mapping |
| `11005` | Dossier không ở trạng thái cho phép sửa | 400 |
| `11006` | Optimistic version conflict | 409 nên được chuẩn hóa ở public API |
| `11010` | Dossier không cho phép xóa | 400 |
| `11011` | Hồ sơ active trùng `subjectRef`+dự án | 409 |
| `11017` | Không có checklist required để submit | 400/422 |
| `11018` | Thiếu required document | 400/422 |

# 7. Data Architecture & Data Flow

## 7.1 Data Model

### 7.1.1 Sở hữu dữ liệu logic

| **Bảng/aggregate** | **Mục đích** | **Invariant chính** |
| --- | --- | --- |
| `dossier` | Aggregate hồ sơ, JSONB business form/metadata, opaque `subjectRef`, Market `ownerSubject`, source, PIC, pipeline projection, version | Không chứa media/kết quả/PII OCR; `subjectRef` có thể rỗng ở DRAFT `{}` nhưng bắt buộc trước OCR/submit; `ownerSubject` server-owned cho MARKET; active subject/project và unit uniqueness. |
| `dossier_status_history` | Lịch sử trạng thái/action | Tạo trong cùng transaction với transition. |
| `dossier_checklist` | Projection readiness/progress tài liệu nghiệp vụ | PK logic template+group; FK cascade; không chứa OCR status/result/evidence. |
| `dossier_stage_reviewer` | Người xử lý theo stage | Chỉ lưu reviewer ID/role và decision metadata cần thiết; tên/email resolve từ nguồn authoritative. |
| `dossier_reminder_sent` | Dedup reminder theo cycle | Dossier/state/rule/cycle unique. |
| `agent_project_permission` | ACL team/project/scope và rotation | Chỉ một permission active theo key. |
| `outbox_event` | Domain event chưa/đã publish | Không mất event khi business commit. |
| `notification_outbox` | Ý định notification và retry | Chỉ lưu recipient ID, template và business reference; Message Delivery resolve địa chỉ, không copy PII/OCR. |
| `dossier_note` | Ghi chú general/hardcopy | Soft-delete; validation ngăn nhập PII/OCR vào free text. |
| `audit_log` | Nền tảng audit cho PIC/operation | Không lưu snapshot PII rõ; table có nhưng writer/use case PIC chưa hoàn chỉnh. |

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

- Partial unique index trên `subject_ref + project_id` cho dossier chưa terminal và `subject_ref IS NOT NULL`; reference do nguồn định danh domain có thẩm quyền cấp/bind server-side, được gửi sang `vhm-ocr-ekyc` và không chứa/khôi phục được PII.
- Partial unique index trên unit được cấp cho dossier active.
- Check constraint cho `source`; MARKET yêu cầu idempotency key.
- Dossier `source=MARKET` yêu cầu `owner_subject` không rỗng; giá trị chỉ được bind từ signed Market context và không cho update.
- Checklist có constraint enum và `invalid_reason` phù hợp trạng thái.
- Optimistic `version` được JPA quản lý; response create được dựng sau flush.
- `pipeline_code + pipeline_version` xác định duy nhất definition của dossier; `current_stage_code/group` phải tồn tại trong đúng version đó.
- Stage reviewer dùng khóa `(dossier_id, stage_code)`, vì vậy thêm cấp duyệt không yêu cầu thêm cột hoặc bảng nghiệp vụ mới.
- Không có FK đến user/PIC/project master vì các định danh này thuộc hệ thống ngoài.

## 7.2 Data Flow Diagram

### 7.2.1 Luồng đăng ký và nộp hồ sơ

```mermaid
sequenceDiagram
    actor U as Người dùng Agent / Market
    participant A as Agent API / Market API
    participant C as dossier-core
    participant O as vhm-ocr-ekyc
    participant F as File Management
    participant D as PostgreSQL

    U->>A: POST registrations {} / partial
    A->>C: Create dossier + signed actor
    C->>C: MARKET bind ownerSubject / AGENT validate BUS context
    C->>D: Insert DRAFT + pipeline + history + outbox
    D-->>C: Commit/version
    C-->>A: dossierId, DRAFT, version
    A-->>U: 201 Created
    U->>A: POST {id}/prepare-upload
    A->>C: GET dossier (authorize)
    A->>C: POST /prepare-upload
    alt Media dùng cho OCR
        C->>O: Prepare OCR media + workload identity
        O->>F: Request presigned upload
        F-->>O: URL ngắn hạn + opaque path
        O-->>C: Upload contract
    else Tài liệu không OCR
        C->>F: Request presigned upload
        F-->>C: URL ngắn hạn + opaque path
    end
    C-->>A: Upload contract
    U->>F: PUT file bytes
    U->>A: PATCH business form/non-OCR documents
    A->>C: PUT dossier
    C->>F: Verify non-OCR references + owner/grant
    F-->>C: Valid / invalid non-OCR references
    C->>D: Update snapshot + synchronize checklist + outbox
    U->>A: POST {id}/submit
    A->>C: command SUBMIT
    C->>O: /ocr/result cho ocrId đã bind với dossier
    O-->>C: COMPLETED / not ready
    C->>D: Guard subject/checklist + transition + code + outbox
    A-->>U: Submitted/Under review
```

### 7.2.2 Luồng phê duyệt

```mermaid
sequenceDiagram
    actor R as Reviewer
    participant A as Agent API
    participant C as Dossier Core Pipeline
    participant D as PostgreSQL
    participant N as Notification Relay

    R->>A: command(action, version, payload)
    A->>C: Signed actor context
    C->>D: Lock/load dossier + current reviewer
    C->>C: Validate state, BUS role, scope, ownership, business guards
    C->>D: Update state/reviewer/history/unit/outbox/noti intent
    D-->>C: Commit
    C-->>R: New state/version/actions
    N->>D: Claim notification outbox
    N->>N: Dispatch + retry independently
```

### 7.2.3 Luồng OCR CCCD qua capability dùng chung

```mermaid
sequenceDiagram
    actor U as Người dùng
    participant B as Agent API / Market API
    participant C as Dossier Core
    participant O as vhm-ocr-ekyc
    participant F as File Management
    participant D as PostgreSQL

    U->>B: Yêu cầu chuẩn bị media OCR
    B->>C: Prepare upload + signed actor context
    C->>C: Kiểm tra quyền hồ sơ và media context
    C->>O: Prepare upload + workload identity
    O->>F: Xin presigned PUT
    F-->>O: URL ngắn hạn + s3PathFile
    O-->>C: Upload contract
    C-->>B: Upload contract
    U->>F: PUT media trực tiếp
    B->>C: Tạo OCR + Idempotency-Key
    C->>C: Lấy subjectRef từ signed domain identity context
    C->>D: Bind subjectRef và kiểm tra duplicate nếu chưa có
    C->>O: POST /ocr + source + referenceId + requestBy + subjectRef + media refs
    O-->>C: 202 + ocrId + Retry-After
    C-->>B: 202 + ocrId + Retry-After
    loop Cho tới terminal/deadline
        B->>C: Lấy trạng thái OCR
        C->>O: /ocr/result
        O-->>C: QUEUED / PROCESSING / terminal
        C-->>B: Trạng thái chuẩn
    end
    B-->>U: Hiển thị kết quả chuẩn để xác nhận
    U->>B: Xác nhận kết quả OCR
    B->>C: CONFIRM_AND_APPLY + ocrId
    C->>O: Đọc lại /ocr/result
    O-->>C: COMPLETED + result chuẩn
    C->>C: Kiểm tra reference và chỉ áp dụng metadata không PII
    C-->>B: Xác nhận domain thành công
```

Không có đường ghi dữ liệu OCR/PII vào PostgreSQL của dossier. Nguồn định danh domain có thẩm quyền cấp/bind `subjectRef` server-side; Core persist reference opaque này và truyền nó trong `POST /ocr`. `vhm-ocr-ekyc` liên kết OCR resource bằng `referenceId=dossierId`, lưu media reference, lifecycle và kết quả OCR đã mã hóa; File Management giữ byte media. Baseline không có OCR-confirm endpoint hoặc lịch sử xác nhận nghiệp vụ tại OCR. Các business field không thuộc OCR tiếp tục đi qua PATCH dossier theo allowlist bình thường.

## 7.3 Data Privacy & PII

### 7.3.1 Checklist dữ liệu cá nhân

Nghiệp vụ hồ sơ NOXH có liên quan dữ liệu cá nhân, nhưng **Dossier Core không tự
thực hiện OCR, không là source of truth và không lưu nội dung PII của
applicant/spouse**. Tương tự checklist trong TDD mẫu của `vhm-ocr-ekyc`, cột
**Phát sinh trong use case** cho biết loại dữ liệu có xuất hiện trong toàn bộ luồng
nghiệp vụ hay không; cột **Lưu tại Dossier Core** xác định ranh giới persistence
của service. Dữ liệu ngoài contract không được chủ đích thu thập; nếu phát sinh
trong file/free text thì phải bị từ chối hoặc xử lý theo policy được Privacy phê
duyệt.

| **Loại dữ liệu cá nhân** | **Phân nhóm** | **Phát sinh trong use case** | **Lưu tại Dossier Core** | **Nơi xử lý/source of truth** |
| --- | --- | :---: | :---: | --- |
| Họ, chữ đệm, tên applicant/spouse/người liên quan | Cơ bản | **Có** | **Không** | `vhm-ocr-ekyc`; File Management với tài liệu không OCR |
| Ngày, tháng, năm sinh | Cơ bản | **Có** | **Không** | `vhm-ocr-ekyc` |
| Giới tính | Cơ bản | **Có** | **Không** | `vhm-ocr-ekyc` |
| Nơi sinh, quê quán, nơi cư trú, địa chỉ liên hệ | Cơ bản | **Có** | **Không** | `vhm-ocr-ekyc`; File Management với tài liệu không OCR |
| Quốc tịch | Cơ bản | **Có** | **Không** | `vhm-ocr-ekyc` |
| Số và thông tin giấy tờ định danh; ngày/nơi cấp; ngày hết hạn | Cơ bản | **Có** | **Không** | `vhm-ocr-ekyc` |
| Số điện thoại, thư điện tử của applicant/spouse | Cơ bản | **Có** | **Không** | Nguồn Applicant/Customer Profile có thẩm quyền: `TBD`; **OCR contract không thu thập contact** |
| Tình trạng hôn nhân, quan hệ vợ/chồng/đồng đăng ký và thông tin hộ gia đình/người phụ thuộc | Cơ bản | **Có** | **Không** | Nguồn authoritative cho applicant/spouse; File Management với tài liệu chứng minh |
| Tình trạng nhà ở, sở hữu bất động sản và nhóm đối tượng được hưởng chính sách NOXH | Cơ bản | **Có** | **Không** | Nguồn authoritative được Product/Privacy chốt; File Management giữ tài liệu chứng minh |
| Nghề nghiệp, đơn vị công tác, quan hệ lao động, mã số thuế/BHXH trong hồ sơ chứng minh điều kiện | Cơ bản | **Có** | **Không** | File Management/nguồn dữ liệu nghiệp vụ authoritative |
| Thu nhập, bảng lương, tài khoản hoặc nội dung tài chính trong tài liệu chứng minh | Nhạy cảm | **Có** | **Không** | File Management/nguồn dữ liệu nghiệp vụ authoritative |
| Thông tin trẻ em/người chưa thành niên trong giấy tờ hộ gia đình hoặc tài liệu chứng minh | Cơ bản | **Có** | **Không** | File Management/nguồn dữ liệu nghiệp vụ authoritative |
| `subjectRef`, `ownerSubject`, actor/reviewer/recipient ID và định danh kỹ thuật có thể liên kết tới cá nhân | Cơ bản/giả danh | **Có** | **Có** | Core chỉ lưu opaque reference tối thiểu; nguồn phát hành reference nằm ngoài Core |
| Họ tên, email/số điện thoại công việc của agent/reviewer/recipient phục vụ phân công và thông báo | Cơ bản | **Có** | **Không** | IAM/TTOL/Message Delivery; Core chỉ lưu opaque recipient/actor ID |
| Ảnh giấy tờ định danh và chữ ký có trong tài liệu hồ sơ | Nhạy cảm | **Có** | **Không** | Byte media tại File Management/kho riêng tư; `vhm-ocr-ekyc` giữ media reference và kết quả OCR; File Management cũng giữ tài liệu không OCR |
| Dữ liệu vị trí thời gian thực hoặc lịch sử di chuyển | Cơ bản | **Không** | **Không** | Ngoài phạm vi Dossier |
| Nguồn gốc chủng tộc, dân tộc | Nhạy cảm | **Không** | **Không** | Ngoài phạm vi Dossier |
| Quan điểm chính trị, tôn giáo, tín ngưỡng | Nhạy cảm | **Không** | **Không** | Ngoài phạm vi Dossier |
| Đời sống riêng tư, bí mật cá nhân/bí mật gia đình ngoài thông tin quan hệ và điều kiện NOXH đã nêu | Nhạy cảm | **Không** | **Không** | Ngoài phạm vi Dossier |
| Tình trạng sức khỏe, hồ sơ y tế hoặc dữ liệu di truyền | Nhạy cảm | **Không** | **Không** | Ngoài phạm vi Dossier |
| Xu hướng tính dục | Nhạy cảm | **Không** | **Không** | Ngoài phạm vi Dossier |
| Dữ liệu về hành vi phạm tội, tiền án, tiền sự | Nhạy cảm | **Không** | **Không** | Ngoài phạm vi Dossier |

`Lưu tại Dossier Core = Không` không loại bỏ trách nhiệm Privacy nếu một API của
Core vẫn proxy/stream raw PII: truyền dữ liệu transient vẫn phải được xem xét như
một hoạt động xử lý. Target contract phải ưu tiên chỉ trả readiness/status và
opaque reference cho Core; nếu bắt buộc proxy PII thì ANBM/Privacy phải duyệt riêng
field projection, authorization, masking và no-cache/no-log evidence.

Checklist này ở trạng thái **DRAFT — chờ ANBM và Privacy/Pháp chế xác nhận và ký
duyệt**. Product/System Owner phải chốt purpose/lawful basis cho từng dòng **Có**
ở cột use case; Backend, OCR, File và Message owner phải cung cấp contract/evidence
chứng minh các dòng **Không** không được persist, log, phát event hoặc thu thập
ngoài mục đích.

### 7.3.2 Phân loại và tối thiểu hóa

- `vhm-ocr-ekyc` sở hữu lifecycle, media reference, dữ liệu trích xuất, confidence và kết quả OCR đã mã hóa; File Management/kho riêng tư sở hữu byte media CCCD.
- Dossier Core chỉ persist `subjectRef` opaque và dữ liệu nghiệp vụ không thuộc OCR; request form chứa CCCD, họ tên, ngày sinh, phone/email, media path hoặc OCR result phải bị từ chối tại persistence boundary.
- Dữ liệu OCR được proxy chỉ tồn tại trong memory trong thời gian request, không cache, không đưa vào Kafka/outbox/event/log/trace/audit snapshot và không ghi vào report tạm.
- Phone/email không nằm trong input/result OCR theo TDD `vhm-ocr-ekyc`; Dossier không được forward contact sang OCR. Nguồn contact có thẩm quyền phải được chốt riêng trước khi có chức năng hiển thị hoặc notification theo địa chỉ.
- Reviewer/recipient dùng opaque ID; tên/email được resolve từ IAM/TTOL/Message tại thời điểm sử dụng và không persist trong Dossier Core.
- Baseline Dossier–OCR chỉ dùng prepare-upload, `POST /ocr` và `/ocr/result`; không giả định có read/search/export/hydrate/delete API theo PII. List/detail/export của Core chỉ trả business metadata allowlist và không tạo local projection/cache chứa PII.

### 7.3.3 Ranh giới bảo vệ dữ liệu với `vhm-ocr-ekyc`

Kết quả OCR CCCD, provider payload, retention và payload encryption được xử lý tại `vhm-ocr-ekyc`; byte media, at-rest encryption của object và vòng đời object thuộc File Management/kho riêng tư. Xem [L2 - VHMKDO2O - Capability OCR dùng chung](https://vin3s.atlassian.net/wiki/spaces/VARW/pages/3014268156/L2+-+VHMKDO2O+-+D+ch+v+OCR+eKYC). TDD Dossier không định nghĩa lại thuật toán mã hóa hoặc vòng đời khóa của các capability này.

Trách nhiệm của Dossier Core giới hạn ở:

1. Xác thực/ủy quyền request trước khi gọi OCR; lấy `subjectRef` từ nguồn định danh domain có thẩm quyền, bind server-side rồi truyền `source`, `referenceId=dossierId`, `requestBy`, `subjectRef` và purpose dạng signed/opaque context.
2. Chỉ stream/proxy đúng field mà persona được phép xem; response PII không đi qua cache, persistence, log, event hoặc APM body capture.
3. Persist `subjectRef` opaque đã được domain bind và truyền giá trị đó vào `POST /ocr`; OCR lưu reference để tương quan/phân quyền. Client không được gửi hoặc sửa reference này trong form body.
4. Duplicate guard dùng `subjectRef + projectId`; issuer, tính ổn định, uniqueness và non-reversibility của `subjectRef` phải được đóng băng trong Domain Identity/OCR Contract L3.
5. Chỉ dùng prepare-upload, `POST /ocr` và `/ocr/result` trong baseline. `CONFIRM_AND_APPLY` xử lý tại domain bằng cách đọc lại kết quả `COMPLETED`; không gọi endpoint confirm chưa tồn tại. Search/export/hydrate/delete theo PII là ngoài baseline và không được fallback sang bản sao local.
6. Dossier Core không sở hữu encryption key/salt cho kết quả OCR hoặc media. Rotation/revocation của payload OCR thuộc `vhm-ocr-ekyc`; khóa object và cryptographic erasure của byte media thuộc File Management/KMS owner.

### 7.3.4 Danh mục dữ liệu và yêu cầu quản lý

Retention/legal hold/purge/quyền data subject đối với kết quả OCR tuân TDD/policy của `vhm-ocr-ekyc`, còn byte media tuân File Management policy. Thời hạn 30 ngày của FPT Sale không phải retention mặc định cho VHM. Dossier Core chỉ quản retention của aggregate nghiệp vụ và opaque reference; purge dossier không đồng nghĩa đã purge OCR result/media. Baseline chưa công bố delete API, vì vậy trước production cần policy/runbook đối soát idempotent giữa Dossier, OCR, File, FPT và backup hoặc một OpenAPI mở rộng đã được phê duyệt.

## 7.4 Data Privacy

Việc ủy quyền OCR/media/PII cho `vhm-ocr-ekyc` và File Management không loại bỏ
trách nhiệm Privacy của Dossier Core. Core vẫn sở hữu aggregate có thể liên kết
với cá nhân, opaque reference, metadata checklist/quyết định và ranh giới
authorization/orchestration tới các nguồn authoritative.

| **Chủ thể DL** | **Hệ thống lưu trữ** | **Số lượng bản ghi** | **Tổng dung lượng** | **Truyền sang bên ngoài** | **Khu vực DL đi qua** | **Kiểu DL thu thập** | **Mục đích** | **Mã hóa lưu trữ** | **Vị trí khóa** | **Xoay khóa** | **Mã hóa đường truyền** | **Masking** | **Vòng đời DL** | **Tự động xóa** | **Xóa theo yêu cầu KH** | **Ẩn danh** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Applicant/spouse gắn với hồ sơ NOXH | PostgreSQL `dossier_db` chỉ lưu aggregate nghiệp vụ, `subjectRef`, `ownerSubject`, project/unit, trạng thái, checklist/quyết định và audit metadata; không lưu raw PII/media/kết quả OCR | `TBD` — theo forecast Product; tối đa một hồ sơ active cho một `subjectRef + projectId` | `TBD` — theo số hồ sơ, checklist/history/outbox và retention; không tính binary/OCR payload | **Không trực tiếp từ Core** — chỉ trao đổi nội bộ với OCR/File/Message; mọi onward transfer của dependency phải được đánh giá trong DPA/DPIA của service owner | Agent/Market API → Dossier Core → PostgreSQL/Redis/Kafka; tuyến tới OCR/File/Message và region triển khai: `TBD` theo topology được duyệt | Opaque/pseudonymous subject/owner reference; dossier/project/unit/status; checklist, reviewer/decision và business metadata allowlist | Quản lý vòng đời hồ sơ, phân quyền object, chống trùng, readiness, phê duyệt và audit nghiệp vụ | **Có — yêu cầu thiết kế**; PostgreSQL, volume và backup phải mã hóa theo chuẩn nền tảng, cấu hình/evidence `TBD` | KMS/key service do nền tảng quản lý; account/key/owner cụ thể `TBD`, không lưu key trong source/config | `TBD` — theo policy KMS/ANBM; phải có rotation/revocation runbook | TLS 1.2 trở lên; ưu tiên TLS 1.3; signed actor/workload context trên tuyến nội bộ | Không hiển thị opaque ID nếu không cần; field projection theo persona; không ghi body/reference nhạy cảm vào log/APM/event | Tạo DRAFT → xử lý/phê duyệt → terminal → giữ theo retention/legal hold → purge/unlink; thời hạn từng trạng thái `TBD` | **Có — yêu cầu thiết kế**; lịch/SLA purge idempotent cho primary, outbox hết hạn và backup `TBD` | **Có — yêu cầu thiết kế**; orchestration với OCR/File/Message, trừ legal hold; SLA và bằng chứng đối soát `TBD` | Không ẩn danh hồ sơ active; sau retention phải purge hoặc unlink không thể tái liên kết theo policy được duyệt |
| Applicant/spouse có PII trong kết quả OCR CCCD | Không persist/cache tại Dossier Core; projection `/ocr/result` chỉ tồn tại transient trong memory/stream. `vhm-ocr-ekyc` lưu request/kết quả đã mã hóa; File Management/kho riêng tư lưu byte media; FPT xử lý theo contract | **0 bản ghi PII persistent tại Core**; số OCR record tại capability theo forecast `TBD` | **0 dung lượng PII at rest tại Core**; payload transient và giới hạn response theo OCR OpenAPI L3 | **Có trong toàn luồng** — `vhm-ocr-ekyc` truyền media/dữ liệu cần thiết tới FPT; Core không gọi FPT trực tiếp. DPA/DPIA, sub-processor và data residency thuộc OCR owner | Agent/Market API → Dossier Core → `vhm-ocr-ekyc` → File Management/FPT; region VHM và vùng xử lý/lưu của FPT: `TBD` | Họ tên, CCCD, ngày sinh, giới tính, địa chỉ, quê quán, quốc tịch, ngày/nơi cấp/hết hạn và confidence/cảnh báo theo allowlist; **không gồm phone/email hoặc dữ liệu ngoài OCR CCCD** | OCR CCCD hai mặt và cho người dùng kiểm tra projection kết quả; Core không OCR, không xác nhận semantic và không tạo local projection | **Không áp dụng at rest tại Core**; kết quả tại OCR dùng payload encryption theo TDD OCR, media tại File dùng server-side encryption/KMS; cấm spill/heap dump/body capture/cache ở Core | Core không sở hữu key/salt OCR/media; CMK payload thuộc OCR owner/KMS, CMK object thuộc File/platform owner | `TBD` theo policy KMS/ANBM của OCR và File; Core chỉ yêu cầu evidence, không xoay khóa của dependency | TLS 1.2 trở lên, ưu tiên TLS 1.3; workload identity và signed actor/purpose context; TLS bắt buộc tới FPT | Domain Backend authorize/project đúng field; Agent/Market API/UI áp dụng `FULL/MASKED/NONE`; `Cache-Control: no-store`; cấm body/media/path trong log/APM/event | Tại Core chỉ trong vòng đời request; kết quả OCR, media, provider metadata và backup theo retention tách biệt đã được duyệt | Không áp dụng tại Core vì không persist PII; OCR/File/FPT phải có purge idempotent và bằng chứng theo policy, lịch/SLA `TBD` | Baseline chưa có delete/correction API; xử lý bằng policy/runbook phối hợp OCR–File–FPT–backup hoặc OpenAPI mở rộng được phê duyệt, trừ legal hold | Không ẩn danh dữ liệu OCR đang phục vụ định danh; chỉ cho phép dữ liệu tổng hợp đã được Privacy phê duyệt |
| Applicant/spouse/người phụ thuộc trong tài liệu không OCR | Binary lưu tại kho riêng tư do File Management quản lý; Core chỉ lưu `s3PathFile`/folder/reference opaque và checklist metadata | `TBD` — theo forecast số hồ sơ × catalogue tài liệu/version | `TBD` — Core chỉ tính reference/metadata; dung lượng binary do File owner capacity, quota file theo File Contract | **Không trực tiếp từ Core**; upload/download qua File Management. File owner phải công bố mọi antivirus/DLP/sub-processor hoặc transfer khác | Client → presigned File endpoint; Dossier Core ↔ File Management để prepare/verify; region/object residency `TBD` | Giấy tờ hôn nhân/hộ gia đình, cư trú, nhà ở, việc làm, thu nhập/tài chính và PII phát sinh trong binary; Core không đọc/index nội dung | Chứng minh điều kiện NOXH, kiểm tra checklist và review tài liệu có thẩm quyền | **Có — yêu cầu thiết kế** cho object, metadata Core và backup; cấu hình/evidence File `TBD` | Key object storage/KMS thuộc File owner; key DB Core thuộc platform owner; không trả key/presigned credential cho Core lâu dài | `TBD` theo File/KMS policy; presigned URL TTL ngắn và không persist | TLS 1.2 trở lên; presigned URL scope đúng object, TTL ngắn, không cho list | Không log path/presigned URL/tên file chứa PII; download chỉ sau object authorization; UI masking theo loại tài liệu/persona | Prepare/upload → attach/verify → review → terminal → giữ/xóa theo purpose/legal hold; orphan upload TTL `TBD` | **Có — yêu cầu thiết kế**; dọn orphan và purge object/reference idempotent, schedule/SLA `TBD` | **Có — yêu cầu thiết kế**; Core–File delete orchestration và đối soát object/reference, trừ legal hold | Không ẩn danh binary đang phục vụ hồ sơ; hết retention phải purge object và unlink reference |
| Agent/reviewer/recipient nội bộ | Core chỉ lưu actor/reviewer/recipient ID opaque, role/stage/decision và audit metadata; tên/email/số điện thoại công việc thuộc IAM/TTOL/Message Delivery | `TBD` — theo số dossier, stage, lần phân công/quyết định/notification | `TBD` — metadata nhỏ theo dossier/history/outbox retention | **Không** — chỉ trao đổi với dịch vụ nội bộ được duyệt; Message Delivery chịu trách nhiệm kênh/provider nếu có | Agent API/Dossier Core ↔ BUS/IAM/TTOL/Message Delivery; region và egress `TBD` | Opaque staff ID, role/scope, stage, assignment, decision, recipient ID và notification intent; không lưu contact address tại Core | Phân quyền, phân công, phê duyệt, audit và gửi notification | **Có — yêu cầu thiết kế** cho DB/backup; contact authoritative được bảo vệ tại IAM/TTOL/Message | KMS/key service của platform và từng source authoritative; chi tiết `TBD` | `TBD` theo policy ANBM của từng owner | TLS 1.2 trở lên; signed actor context; service identity tới IAM/TTOL/Message | UI chỉ hiển thị thông tin cần thiết sau resolve; log dùng opaque ID; không đưa email/phone vào event/outbox Core | Theo vòng đời assignment/audit và retention hồ sơ; thay đổi hồ sơ nhân sự được cập nhật tại source authoritative | **Có — yêu cầu thiết kế** cho outbox/assignment hết hạn; audit giữ theo policy/legal hold | Rectify contact tại source; delete/unlink ID tại Core theo policy và nghĩa vụ audit, không xóa trái legal hold | Pseudonymize bằng opaque ID; dữ liệu audit active không ẩn danh nếu còn mục đích hợp lệ |

Các giá trị `TBD` là đầu vào bắt buộc trước khi dùng dữ liệu thật. **Product/System
Owner** chốt forecast, purpose và lawful basis; **Backend/DBA/Vận hành** chốt
capacity, encryption, backup và purge; **OCR/File/Message owner** cung cấp
retention/delete/transfer evidence; **ANBM và Privacy/Pháp chế** phê duyệt data
residency, masking, key lifecycle, legal hold, quyền data subject và residual risk.

## 7.5 Data Stores & Ownership

| **Store** | **Authority** | **Failure impact** | **Phục hồi** |
| --- | --- | --- | --- |
| PostgreSQL `dossier_db` | Dossier/checklist/pipeline/history/outbox | Core không thể mutate an toàn | PITR/backup/restore TBD |
| Redis | Nonce/replay, counter, cache/coordination | Auto-assign/cache suy giảm; security replay có thể fail closed | Rebuild cache; HA/runbook TBD |
| Kafka | Distribution của event đã commit | Downstream trễ; business data không mất do outbox | Replay relay/topic retention TBD |
| Private File Store | Binary tài liệu | Upload/OCR/download không hoạt động | File Management quản lý; Core truy cập trực tiếp cho tài liệu không OCR, còn media OCR qua `vhm-ocr-ekyc`. |

# 8. Business Flow Diagrams

## 8.1 Sequence/State Diagram

### 8.1.1 Create DRAFT và replay

1. Xác thực actor và chuẩn hóa `source`/key.
2. Với key, lấy advisory lock và kiểm tra replay theo actor.
3. Chỉ khi không replay mới chạy structural guard, optional JSON Schema, XSS sanitizer và file validation.
4. Nếu signed context đã có `subjectRef`, kiểm tra trùng friendly trước insert; create `{}` chưa có reference được phép bỏ qua và phải kiểm tra lại khi bind.
5. Persist DRAFT, pipeline projection, history, checklist nếu documents không rỗng và outbox.
6. Flush để lấy đúng version; unique conflict cuối được map thành lỗi nghiệp vụ.

### 8.1.2 Update snapshot

1. Kiểm tra dossier visibility và editable state.
2. Nếu có `If-Match`, so với version hiện hành.
3. DRAFT/ADD_INFO: validate business snapshot không chứa OCR/PII, file không OCR và duplicate theo `subjectRef`; giữ server-owned unit; synchronize checklist kể cả danh sách rỗng.
4. Trường CCCD applicant/spouse chỉ đi qua luồng OCR và không PATCH vào `formData`; phone/email không thuộc OCR và phải đi qua nguồn Applicant/Customer Profile được phê duyệt nếu có use case tương lai.
5. Ghi status/form update history/outbox trong cùng transaction.

### 8.1.3 Submit và sinh mã

1. Pipeline xác nhận actor là owner và action `SUBMIT` hợp lệ ở `DRAFT`.
2. Social Housing guard yêu cầu `subjectRef` authoritative và kiểm tra duplicate active lại trong transaction.
3. Checklist phải có required item và toàn bộ required đã upload.
4. Core kiểm tra và snapshot các trường server-owned cần thiết theo contract nghiệp vụ đã được phê duyệt.
5. Sinh mã submit đầu, transition sang Sales review, auto-assign PKD best effort và ghi outbox.

### 8.1.4 Revision và reminder

Khi reviewer request revision, hồ sơ đi về stage intake tương ứng hoặc `agentUpdateAtSales`. Rule YAML hiện dùng mốc nhắc sau 144 giờ/deadline 216 giờ và sau 432 giờ/deadline 504 giờ, loại trừ ngày nghỉ theo nguồn lịch được phê duyệt. Scanner chạy theo fixed delay, dedup theo cycle; manual trigger chỉ dành cho PKD/PKD_LEAD. Nếu lịch/notification dependency lỗi, hồ sơ không được tự động reject.

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

- Chuẩn hóa `subjectRef`/project ID trước duplicate lookup; không chấp nhận reference rỗng hoặc do client tự khai. Core phải xác minh reference với signed domain identity context/issuer có thẩm quyền và truyền cùng giá trị vào OCR capability.
- XSS sanitizer áp dụng cho form/metadata và dữ liệu đưa vào tài liệu/thông báo.
- Server giữ quyền sở hữu `source`, `assignedUnitCode`, `assignedUnitId`, pipeline projection và audit fields.
- `documents[].documentTemplateId` phải là UUID hợp lệ; `groupCode` được chuẩn hóa rỗng cho legacy.
- Timestamps lưu theo kiểu thời gian thống nhất của persistence; API phải biểu diễn ISO-8601 có timezone.

# 9. Security & Compliance Architecture

## 9.1 Identity & Authentication

Kênh Agent/Back Office xác thực tại Agent API; kênh Market Customer xác thực tại Market API. Hai BFF xác định channel/source, thực thi public authorization rồi gọi Dossier Core bằng workload identity độc lập. Request vào Core có Basic Auth/HMAC và security context được ký, nhưng claim theo channel khác nhau:

| **Context** | **Claim bắt buộc** | **Claim không hợp lệ** |
| --- | --- | --- |
| Market | `channel=MARKET`, actor subject, data-subject ID/owner binding, issued/expiry, JTI | Agent/PKD/PTT role, `ALL/TEAM/ASSIGNED` visibility hoặc owner do request body cung cấp. |
| Agent/Back Office | `channel=AGENT`, actor subject, BUS roles, project/team scope, visibility, issued/expiry, JTI | Role/scope/visibility tự khai trong request body. |

Khi Market tạo hồ sơ, Core bind owner subject từ signed Market context vào dossier; không lấy từ form. Market API phải kiểm tra data subject trước mỗi list/detail/mutation, còn Core xác minh lại client/channel, chữ ký và subject–dossier binding. Khi Agent/Back Office xử lý hồ sơ, Core dùng BUS roles và scope đã ký; `source` của dossier không tự cấp hoặc tước quyền reviewer.

Ở STAG/PROD, chữ ký nội bộ và actor context là bắt buộc. Timestamp giới hạn replay window; nonce được kiểm tra qua Redis. Basic username phải khớp client ID đã đăng ký riêng cho Agent API hoặc Market API. `source=AGENT|MARKET` do BFF tương ứng gán từ channel context đã xác thực, không lấy trực tiếp từ body của client. Local bypass chỉ hợp lệ khi active profile chính xác là `local`; bypass không được xuất hiện ở STAG/PROD.

## 9.2 Authorization & Access Control

Authorization áp dụng defense in depth nhưng ma trận khác nhau theo channel:

| **Kiểm soát** | **Market Customer** | **Agent/Back Office** |
| --- | --- | --- |
| Public boundary | Market API verify authentication và customer là đúng data subject/object owner. | Agent API verify authentication, BUS roles và project/team scope. |
| Context vào Core | Signed subject context, không có business role. | Signed actor context có BUS roles, scope và visibility. |
| List/detail | Luôn ép filter owner subject; client filter không được mở rộng phạm vi. | `ALL`, `TEAM`, `SELF_CREATED`, `ASSIGNED` theo role/scope được ký. |
| Mutation hồ sơ | Chỉ dossier của data subject và action dành cho Market ở state hiện tại. | Role + project/team scope + pipeline action + `OWNER/CLAIMER/NONE`. |
| Review/lead/admin | Không được phép. | Chỉ role BUS tương ứng và scope hợp lệ. |
| Dữ liệu OCR/PII | Market API/Core proxy đúng subject và purpose; không persist/cache. | Chỉ persona nghiệp vụ được BUS/pipeline cho phép; không persist/cache tại Core. |
| Core defense | Verify workload, HMAC, channel, signed subject và subject–dossier binding; thực thi state/invariant. | Verify workload, HMAC, BUS role/scope/visibility, ownership và state/invariant. |

Các visibility `REGION` và `DEPARTMENT` hiện chưa có semantics đầy đủ cho Agent nên bị từ chối, không fallback `ALL`. Chúng không áp dụng cho Market. Lịch sử assignment chỉ mở read visibility Agent khi feature tương ứng được bật và phê duyệt. `source=MARKET|AGENT` là nguồn tạo hồ sơ, không thay thế data-subject ownership hoặc BUS authorization.

### 9.2.1 Role × Function matrix

Ký hiệu: **✓** là role/principal có capability nhưng vẫn phải qua channel,
project/team scope, visibility, state/invariant và ownership tương ứng; **△** là
capability có điều kiện chưa đủ contract hoặc chỉ áp dụng cho một phần chức năng;
**—** là không được phép. Role là điều kiện cần, không tự tạo quyền nếu action
không có trong `availableActions` của dossier.

| **Function** | **Market Customer** | **`APPLICANT_AGENT`** | **`PKD`** | **`PKD_LEAD`** | **`PTT`** | **`PTT_LEAD`** | **BO/Admin** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| List/search hồ sơ | **✓** `DATA_SUBJECT`, luôn ép owner filter | **✓** project/scope + visibility được ký | **✓** project/team/assigned theo scope | **✓** project/team scope của PKD | **✓** project/team/assigned theo scope | **✓** project/team scope của PTT/SXD | **△** chỉ khi BUS cấp role/scope tra cứu chính thức |
| Xem detail/progress/timeline và download được phép | **✓** đúng owner subject | **✓** `OWNER` + scope | **✓** stage/scope/visibility hợp lệ | **✓** scope PKD hợp lệ | **✓** stage/scope/visibility hợp lệ | **✓** scope PTT/SXD hợp lệ | **△** purpose, field projection và audit access phải được chốt |
| Tạo DRAFT | **✓** Market API bind `ownerSubject`; không business role | **✓** project/scope hợp lệ | **—** | **—** | **—** | **—** | **—** trừ khi đồng thời mang role tạo hồ sơ được BUS cấp |
| Update business snapshot, upload/attach và xóa DRAFT | **✓** `DATA_SUBJECT` + state/action Market | **✓** `OWNER` + DRAFT/ADD_INFO; delete chỉ DRAFT | **—** | **—** | **—** | **—** | **—** |
| Tạo/poll/xác nhận áp dụng OCR CCCD | **✓** đúng subject/dossier và purpose | **✓** `OWNER` + scope | **—** không xác nhận thay applicant | **—** | **—** | **—** | **—**; support access cần contract riêng |
| `SUBMIT`/`RESUBMIT` hồ sơ | **✓** `DATA_SUBJECT` + action Market được công bố | **✓** `OWNER` + readiness/state guard | **—** | **—** | **—** | **—** | **—** |
| `SUBMIT_HARDCOPY` | **△** chỉ khi Market channel policy công bố action | **✓** `OWNER` tại stage hỗ trợ | **—** | **—** | **—** | **—** | **—** |
| `ASSIGN`/`REASSIGN`/`CLAIM` reviewer | **—** | **—** | **—**; nhận auto-assignment không cấp quyền reassign | **✓** tại stage PKD; theo slot và assignee guard | **△** `ASSIGN` tại stage PTT/SXD khi definition cho phép; không `REASSIGN/CLAIM` | **✓** tại stage PTT/SXD; theo slot và assignee guard | **—** trừ khi đồng thời có Lead role phù hợp |
| Duyệt checklist/tài liệu | **—** | **—**; chỉ upload/view được phép | **✓** stage PKD + `CLAIMER` | **✓** stage PKD + `CLAIMER` | **✓** stage PTT/SXD + `CLAIMER` | **✓** stage PTT/SXD + `CLAIMER` | **—** trừ khi đồng thời có reviewer role phù hợp |
| `APPROVE`/`REJECT`/`REQUEST_REVISION` | **—** | **—** | **✓** stage PKD + `CLAIMER` | **✓** stage PKD + `CLAIMER` | **✓** stage PTT/SXD + `CLAIMER` | **✓** stage PTT/SXD + `CLAIMER` | **—** trừ khi đồng thời có reviewer role phù hợp |
| `RETURN_TO_SALES` | **—** | **—** | **—** | **—** | **✓** stage thủ tục + `CLAIMER` | **✓** stage thủ tục + `CLAIMER` | **—** |
| `ALLOCATE_UNIT`/`REVOKE_UNIT` | **—** | **—** | **✓** allocate tại PKD khi `CLAIMER`; revoke ở state được định nghĩa | **✓** như PKD | **△** chỉ `REVOKE_UNIT` ở state PTT/SXD được định nghĩa | **△** chỉ `REVOKE_UNIT` ở state PTT/SXD được định nghĩa | **—** trừ khi đồng thời có functional role phù hợp |
| `CONFIRM_HARDCOPY_RECEIVED` | **—** | **—** | **✓** tại stage được định nghĩa | **✓** tại stage được định nghĩa | **✓** tại stage được định nghĩa | **✓** tại stage được định nghĩa | **—** trừ khi đồng thời có reviewer role phù hợp |
| Statistics/report/export business metadata | **△** chỉ dữ liệu của chính subject và column allowlist | **△** owner/project scope và column allowlist | **△** scope PKD, không hydrate PII | **△** scope PKD, không hydrate PII | **△** scope PTT/SXD, không hydrate PII | **△** scope PTT/SXD, không hydrate PII | **△** cần BUS admin role, purpose, export limit và audit contract |
| Quản lý `agent_project_permission` | **—** | **—** | **—** | **—** | **—** | **—** | **△** BO/Admin role code và scope quản trị còn phải đóng băng trong BUS Contract L3 |
| Trigger reminder thủ công | **—** | **—** | **✓** theo rule/endpoint được phép | **✓** theo rule/endpoint được phép | **—** | **—** | **△** chỉ với operations role được phê duyệt, không mặc định theo nhãn Admin |

Đầu mối SXD là một **stage**, không phải role độc lập trong catalogue hiện tại;
quyền tại `sxdPending` dùng `PTT`/`PTT_LEAD` theo Pipeline Definition đã pin.
`PKD_LEAD` không có quyền ở stage PTT/SXD và `PTT_LEAD` không có quyền ở stage
PKD chỉ vì là Lead. BO/Admin không bypass pipeline: nếu thực hiện action nghiệp vụ
thì signed context phải đồng thời có functional BUS role, scope và ownership phù
hợp. Ma trận chính thức phải được xuất từ BUS Role Catalogue + Pipeline Definition
L3 và được contract-test để tránh drift với bảng này.

## 9.3 Secrets & Credential Management

- HMAC secret, Basic credential và File/OCR/Message/TTOL credential không được nằm trong source, image, application config rõ hoặc tài liệu này. Agent API và Market API dùng credential nội bộ riêng khi gọi Core. Credential File của Core chỉ có scope cho tài liệu không OCR; OCR workload dùng credential riêng để gọi File/provider, còn CMK object do File/platform owner quản lý.
- Secret phải được cấp qua secret manager/runtime, có owner, rotation period và emergency revocation runbook.
- Core và `vhm-ocr-ekyc` dùng danh tính workload riêng, audience/scope tối thiểu; dossier không nhận provider credential OCR.
- Không copy cookie STG vào code/config/log; credential từng được chia sẻ ngoài luồng phải được rotate/revoke.
- Startup/readiness phải fail khi capability bắt buộc được bật nhưng thiếu secret/config hợp lệ.

## 9.4 Application Security & Data Protection

### Kiểm soát request và file

- Body bị từ chối nếu chứa các trường giả mạo security context như `actorId`, `actorDisplayName`, `roles`, `scope`, `visibility`, `dataSubjectId` hoặc `ownerSubject` ở bất kỳ cấp lồng nhau.
- Dossier persistence API từ chối CCCD, họ tên, ngày sinh, phone/email, media OCR, OCR result/confidence/provider metadata và `subjectRef` do client tự khai. Trường CCCD chỉ đi qua OCR contract chuyên biệt; contact không thuộc OCR và phải đi qua nguồn Applicant/Customer Profile được phê duyệt nếu use case tương lai cần.
- Structural guard giới hạn shape/kích thước; JSON Schema bảo vệ semantic khi feature được bật; XSS sanitizer chạy trước persistence/render.
- File type/size/magic/checksum phải được kiểm soát tại File Contract cho tài liệu không OCR và OCR Contract cho media OCR; Core chỉ lưu path opaque của tài liệu không OCR.
- Presigned URL có TTL ngắn, phạm vi object chính xác, không cho list và không được persist/log.
- File existence không đồng nghĩa ownership; attach authorization chỉ được coi là hoàn chỉnh khi có upload-grant/owner contract.

### Ma trận bảo vệ dữ liệu

| **Trạng thái dữ liệu** | **Kiểm soát bắt buộc** |
| --- | --- |
| In transit | TLS; HMAC/mTLS/JWT theo ranh giới; không gửi credential trong query. |
| At rest | PostgreSQL Dossier không chứa OCR/PII applicant; dữ liệu nghiệp vụ/backup vẫn mã hóa theo chuẩn VHM. Bảo vệ at-rest của OCR theo TDD `vhm-ocr-ekyc`. |
| In use | Dữ liệu OCR/PII chỉ tồn tại transient khi proxy, không dump/cache/log/APM; giới hạn export/download. |
| Event | Chỉ opaque ID và metadata tối thiểu; không PII, media path hoặc presigned URL. |
| Retention/deletion | Policy theo purpose/legal hold; purge cả primary, object, outbox đã hết hạn và backup theo lịch. |

### Data Masking

Ba mức hiển thị được áp dụng sau authorization tại mục 9.2: `FULL` chỉ cho
persona có purpose và quyền trên đúng dossier; `MASKED` cho list/report hoặc
persona chỉ cần đối chiếu; `NONE` là không trả field. Dossier Core kiểm tra quyền
business object và chỉ proxy field allowlist từ `/ocr/result`; Agent API/Market API
ánh xạ persona sang `FULL/MASKED/NONE` và masking server-side trước khi trả UI,
đúng ranh giới Domain Backend/BFF của TDD OCR. Không gửi giá trị đầy đủ rồi trông
chờ UI che. Export PII không thuộc baseline hiện tại; nếu bổ sung phải có scope,
giới hạn, audit, approval và contract nguồn authoritative riêng.

| **Trường/nhóm dữ liệu** | **Authority/persistence** | **Hiển thị theo role/purpose** | **Log/event/APM** | **Format masking/yêu cầu** |
| --- | --- | --- | --- | --- |
| Họ tên applicant/spouse | `vhm-ocr-ekyc`/nguồn applicant authoritative; không persist tại Core | Market owner và `APPLICANT_AGENT` owner: `FULL`; PKD/PTT: `FULL` chỉ ở detail đúng stage/purpose, `MASKED` ở list/report; BO/Admin: `NONE` nếu chưa có purpose contract | Không ghi | Giữ ký tự đầu mỗi từ, phần còn lại `*`; ví dụ `N***** V** A*` |
| CCCD/CMND/hộ chiếu | `vhm-ocr-ekyc`; không persist tại Core | Owner hoặc reviewer có purpose định danh: `FULL` ở màn hình chi tiết được duyệt; các view khác `MASKED`/`NONE` | Không ghi, không dùng làm correlation/search log | Chỉ giữ 4 ký tự cuối: `********1234` |
| Ngày sinh | `vhm-ocr-ekyc`; không persist tại Core | `FULL` cho owner/reviewer đúng purpose; list/report mặc định `MASKED`; role không liên quan `NONE` | Không ghi | `**/**/YYYY`; ví dụ `**/**/1990` |
| Địa chỉ/nơi cư trú/quê quán | Nguồn applicant authoritative hoặc binary tại File Management; không persist raw value tại Core | `FULL` chỉ khi kiểm tra điều kiện cư trú; PKD/list/report dùng `MASKED`; role khác `NONE` | Không ghi | Chỉ giữ tỉnh/thành phố: `***, <Tỉnh/Thành phố>` |
| Số điện thoại applicant/spouse | Nguồn applicant authoritative; không persist tại Core | Market/Applicant owner và PTT đúng purpose: `FULL`; `PKD`/`PKD_LEAD`: `MASKED`; BO/Admin: `NONE` nếu không có support purpose | Không ghi | Giữ 2 ký tự đầu + 2 cuối: `09******89` |
| Email applicant/spouse | Nguồn applicant authoritative; không persist tại Core | Market/Applicant owner và PTT đúng purpose: `FULL`; `PKD`/`PKD_LEAD`: `MASKED`; BO/Admin: `NONE` nếu không có support purpose | Không ghi | Giữ ký tự đầu local-part và domain: `a***@example.com` |
| Thu nhập/tài chính/việc làm/hộ gia đình trong tài liệu | Binary tại File Management/nguồn authoritative; Core chỉ giữ reference/checklist metadata | Không trả thành field list; chỉ download/view tài liệu cho role đúng stage/purpose. Preview/report nếu có mặc định `MASKED` | Không ghi nội dung, filename gốc hoặc extracted text | `MASKED` che toàn bộ giá trị: `******`; media phải dùng viewer/download grant có thời hạn |
| Media CCCD và tài liệu không OCR | Byte media/file tại File Management/kho riêng tư; `vhm-ocr-ekyc` chỉ lưu media reference và kết quả OCR | Core không render hoặc trả raw media. BFF/UI chỉ nhận download/view grant sau object authorization; thumbnail cũng áp dụng cùng quyền | Không ghi bytes, checksum nhạy cảm, path, filename PII hoặc presigned URL | Không masking binary tại Core; dùng access control, watermark/redaction nếu File/OCR contract yêu cầu |
| `subjectRef`, `ownerSubject`, actor/reviewer/recipient ID | Core lưu opaque reference tối thiểu | `NONE` trên UI nghiệp vụ trừ support view được duyệt; UI dùng display projection từ nguồn authoritative | Chỉ dossier/correlation ID trong allowlist; không log subject/owner/recipient reference nếu không cần | Không partial-mask; phải opaque, không nhúng PII và không đảo ngược được. Loại khỏi response thay vì che hình thức |
| Tên reviewer/agent và contact công việc | IAM/TTOL/Message Delivery; Core chỉ lưu opaque ID theo target design | Tên có thể `FULL` trong assignment/timeline cho người xem dossier; email/phone `NONE` trừ notification/support purpose | Log opaque actor ID; không log tên/email/phone | Email nếu buộc hiển thị dùng `a***@domain`; phone dùng `09******89` |
| Dossier code, project/unit, status, progress và decision metadata | Core authoritative | `FULL` sau scope/object authorization; report chỉ trả column allowlist | Có thể ghi dossier ID/action/status/error code theo allowlist; không ghi full response | Không masking mặc định nhưng vẫn là metadata có thể liên kết cá nhân; cấm dùng làm metric label cardinality cao |
| File path, filename và presigned URL | Path/reference tại Core/File; URL chỉ cấp tạm thời | Path/URL `NONE`; chỉ hiển thị filename đã sanitize khi persona có quyền download | Không ghi | Không trả storage path; presigned URL đúng object/method, TTL ngắn và response `no-store` |
| Secret, HMAC/actor token, idempotency key và credential tích hợp | Secret manager/runtime hoặc state kỹ thuật tối thiểu | `NONE` cho mọi role/UI/API response | Không ghi; lỗi phải redact | Redact toàn bộ: `[REDACTED]`; idempotency key chỉ dùng lookup/hash theo thiết kế được duyệt |

Format trên là baseline cần ANBM/Privacy và Product phê duyệt trong field-level
authorization contract. Khi nhiều role cùng tồn tại trong actor context, áp dụng
mức hiển thị hạn chế nhất trừ khi contract có rule ưu tiên rõ ràng; tuyệt đối không
dùng việc ghép role để nâng từ `MASKED/NONE` lên `FULL` ngoài purpose đã ký.

### Encryption

| **Phạm vi** | **Dữ liệu/vị trí** | **Cơ chế mã hóa** | **Thuật toán/baseline** | **Quản lý khóa và xoay khóa** | **Cổng bằng chứng trước production** |
| --- | --- | --- | --- | --- | --- |
| PostgreSQL Dossier | Aggregate, `subjectRef`, `ownerSubject`, checklist/history/outbox/audit metadata | Mã hóa storage/volume/database và backup theo chuẩn nền tảng; access bằng workload identity/DB role tối thiểu | AES-256 hoặc baseline at-rest được VHM/ANBM phê duyệt; cấu hình cụ thể `TBD` | CMK/key service do platform owner quản lý; account/key owner, rotation/revocation period `TBD` | Cấu hình encryption, DB privilege review, backup/restore và rotation evidence |
| Opaque subject/owner reference | Giá trị cần index/unique trong PostgreSQL | Không mã hóa field-level mặc định để giữ lookup/constraint; dựa vào non-reversibility, DB encryption và access control. Nếu reference có thể suy ra PII thì contract không đạt | Opaque random/pseudonymous reference với entropy/format do source authority chốt; không hash CCCD bằng salt dùng chung | Core không giữ salt/key sinh `subjectRef`; issuer quản lý lifecycle. Thay đổi thuật toán/reference cần migration/version contract | Non-reversibility/threat review, cross-subject negative test và DB/log data scan |
| Redis | Nonce/replay, counter, cache/coordination; không OCR/PII | TLS in transit; encryption at rest nếu persistence/snapshot bật; ACL và network isolation | Theo baseline Redis managed service được ANBM duyệt; cipher/config `TBD` | Platform owner quản lý certificate/key và rotation; security nonce lỗi phải fail closed | Redis ACL/TLS/persistence config, failover và replay test |
| Kafka/outbox | Event metadata opaque, không PII/media/path/presigned URL | TLS cho producer/broker; broker/storage encryption theo nền tảng; topic ACL | TLS 1.2 trở lên, ưu tiên TLS 1.3; at-rest baseline `TBD` | Platform Kafka owner quản lý certificate/CMK và rotation; Core không nhúng key vào config rõ | Topic/schema/ACL review, payload scan, certificate rotation và DLQ evidence |
| Tài liệu không OCR | Binary tại File Management/object storage; Core chỉ giữ reference | Server-side encryption bằng KMS, bucket/object private; presigned URL TTL ngắn và scope chính xác | SSE-KMS/AES-256 hoặc baseline File được ANBM duyệt | CMK thuộc File owner; Core không nhận key. Rotation, revocation và cryptographic erasure theo File Contract | File encryption attestation, owner/upload-grant E2E, access log và delete/restore drill |
| OCR media/kết quả/PII | Kết quả/media reference lưu tại `vhm-ocr-ekyc`, byte media tại File Management/kho riêng tư; transient trong Core chỉ khi proxy `/ocr/result` | Không persist/cache/spill tại Core; TLS khi truyền. Payload encryption thuộc OCR, server-side object encryption thuộc File | Theo TDD `vhm-ocr-ekyc` và File/KMS baseline; Core không định nghĩa lại thuật toán | OCR owner quản lý CMK payload, File/platform owner quản lý CMK object; Core chỉ yêu cầu evidence và không có decrypt/key permission | OCR OpenAPI/IAM, File encryption attestation, no-cache/no-body-log/heap-dump evidence và ANBM/Privacy approval |
| Kết nối service-to-service | Agent/Market API ↔ Core ↔ PostgreSQL/Redis/Kafka/OCR/File/Message/TTOL | HTTPS/TLS; mTLS hoặc signed workload identity tại trust boundary theo chuẩn VHM; không clear text sau TLS termination | TLS 1.2 trở lên, ưu tiên TLS 1.3; cấm protocol/cipher yếu | Certificate/private key từ certificate/secret manager, tự động renew/revoke; không nằm trong source/image | TLS scan, trust-store/certificate rotation test và network-flow approval |
| Secret và credential | HMAC, Basic credential, DB/File/Message/TTOL credential | Chỉ cấp runtime qua secret manager/KMS; không lưu trong PostgreSQL, Kafka, log, image hoặc source | Cơ chế encryption của secret manager/KMS được VHM phê duyệt | Owner, version, rotation period và emergency revocation bắt buộc; credential tách theo client/environment | Secret scan/SBOM, runtime injection evidence và rotation/revocation drill |
| Backup, snapshot và artefact export | PostgreSQL backup/PITR, Redis snapshot nếu bật, report/ZIP/XLSX tạm | Backup mã hóa bằng key tách quyền; artefact export private, TTL/retention và download authorization | Baseline backup/object encryption `TBD`; không dùng key hard-code | DBA/Vận hành/File owner quản lý key và restore permission; rotation không làm mất khả năng restore trong retention window | Restore drill, expired artefact purge, key recovery và restore-and-repurge evidence |

TDD này không khẳng định encryption đã hoàn tất chỉ vì platform hỗ trợ. Mọi giá
trị `TBD`, thuật toán tương đương, vị trí CMK, quyền decrypt, rotation period và
backup-key recovery phải có evidence và được ANBM/Vận hành phê duyệt trước dữ liệu
thật/production.

### Nhật ký kỹ thuật

Log tối thiểu gồm correlation ID, client ID, actor subject dạng opaque, dossier ID, action, kết quả, duration và error code. Không log request/response body chứa PII, CCCD, contact, file URL, HMAC/actor token hoặc OCR result. Audit quyết định nghiệp vụ phải tách khỏi debug log và có quyền truy cập/retention riêng.

### Mô hình mối đe dọa

| **ID** | **Mối đe dọa** | **Kiểm soát** | **Tồn dư** |
| --- | --- | --- | --- |
| TH-01 | IDOR đọc/sửa dossier ngoài phạm vi | Market: signed data subject + owner binding; Agent: BUS role/scope/visibility/ownership | Cần E2E negative matrix riêng theo channel. |
| TH-02 | Giả mạo request nội bộ/replay | HMAC body hash, timestamp, nonce, Redis | Rotation/HA Redis cần runbook. |
| TH-03 | Race tạo hồ sơ/cấp căn trùng | Advisory lock, optimistic lock, partial unique index | Cần load/concurrency test. |
| TH-04 | Attach file của actor khác | Access-before-upload + existence check | Chưa có owner/upload-grant; rủi ro cao. |
| TH-05 | Client tự khai required checklist | Có thể làm sai submit readiness nếu server tin snapshot client | Checklist authority và version server-side. |
| TH-06 | PII/secret lọt log/event | Allowlist log/event, scan CI/APM, runbook incident | Cần evidence production. |
| TH-07 | OCR result tự động gây quyết định sai | Người dùng xác nhận tại OCR capability; Dossier không copy/apply result vào snapshot và không auto reject | Manual review UX/contract cần UAT. |
| TH-08 | Client giả mạo `source=MARKET` | Agent API/Market API gán source theo client identity; Core không tin source do client tự khai | Cần negative contract test chéo hai BFF. |
| TH-09 | OCR/PII bị nhân bản vào Dossier DB/outbox/log/report | Persistence denylist, opaque reference, no-cache/no-body-log và data scan quality gate | Cần migration xóa dữ liệu legacy và E2E chứng minh không còn bản sao. |
| TH-10 | Market Customer truy cập dossier của data subject khác | Market API object authorization, signed subject context, Core subject–dossier binding và forced owner filter | Cần test list/detail/mutation/export chéo customer. |
| TH-11 | Agent tự khai/nâng role hoặc vượt project scope | BUS authority, Agent API ký role/scope, Core allowlist/mapping deny-by-default | Cần contract/version và negative test role/scope. |

# 10. Deployment & Infrastructure Topology

## 10.1 Environments

| **Môi trường** | **Mục đích/SLO** | **Topology và dependency** | **Dữ liệu** | **Security và truy cập** | **HA/DR** | **Cấu hình/tính năng** | **Cổng vào/ra môi trường** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Local (`local` profile) | Phát triển và E2E trên máy; không dùng để đo SLO | Dossier Core `:8888`, Agent API `:8090`, UI `:5173`; PostgreSQL/Redis/Kafka và dependency local/sandbox qua `../docker-compose.yml` khi cần | Chỉ synthetic/fixture đã làm sạch; không cookie/credential/dump STG/PROD | Local auth bypass chỉ hợp lệ khi active profile chính xác là `local`; bind loopback/private dev network; secret local không tái sử dụng môi trường khác | Không HA; rebuild/reseed được; không dùng làm bằng chứng RTO/RPO | Có thể tắt Kafka publisher/external integration; không được thêm mock fallback cho API list/create; structural/security guard vẫn được test | Unit/integration/E2E local đạt; không promote data/config/secret local |
| CI/Test ephemeral | Compile, unit/integration/contract/security scan cho từng change | Runner/container tạm thời; PostgreSQL/Redis/Kafka test container hoặc fixture có version; external client dùng stub/contract sandbox có kiểm soát | Synthetic only; repository/image/log/artefact phải qua secret/PII scan | Credential ngắn hạn, least privilege; runner cô lập; không inbound từ internet ngoài CI control plane | Không HA; tạo mới và hủy sau job; test artefact giữ theo CI retention | Cùng image/source với release; Liquibase validate, pipeline/schema validation, feature combinations bắt buộc | 100% quality gate bắt buộc pass; SBOM/SAST/SCA/license/container/IaC và contract evidence được lưu |
| STAG/UAT (`stag` profile) | SIT/contract/UAT, migration rehearsal, load baseline và security verification; chưa áp production SLO | Topology gần PROD ở quy mô nhỏ; Agent/Market API STAG; real sandbox của File/OCR/Message/TTOL/Kafka khi contract-test | Synthetic/masked; dữ liệu thật chỉ theo ngoại lệ được Privacy duyệt, kho cô lập và có bằng chứng xóa | Không local bypass; workload signature, actor/subject context, replay protection, network policy và secret manager như PROD; chỉ VPN/enterprise ingress được duyệt | Có thể giảm replica/không HA đầy đủ nhưng phải diễn tập failover/restore theo lịch; không dùng cấu hình tối giản để chứng minh capacity PROD | Security/file/schema validation bật như PROD. Kafka publisher hiện có thể tắt theo config nhưng phải bật với topic/ACL thật trước khi dùng làm release evidence | Promote cùng image digest đã test; contract/UAT/security/migration/load gate đạt; dữ liệu test được purge sau campaign |
| PROD (`prod` profile) | Nghiệp vụ thật; target dự kiến mục 4.1 và SLO cuối được phê duyệt | Core multi-replica/private ingress; PostgreSQL HA/PITR, Redis HA cho replay, Kafka/outbox, File/OCR/Message/TTOL production | Dữ liệu thật; Core chỉ giữ aggregate/opaque reference; OCR/File giữ dữ liệu authoritative theo mục 7.3/7.4 | Signature/actor required, local bypass absent, deny-by-default network/IAM, runtime secrets/KMS, encryption/masking/audit/DLP đầy đủ | Multi-AZ/HA theo capacity; RTO/RPO mục 14.1; backup/restore, backlog replay và runbook on-call | File validation, replay guard và security invariants bắt buộc; Kafka/notification/auto-assignment chỉ bật khi production gate/degradation path đạt | Canary/rolling theo mục 10.3; System Owner/Vận hành/DBA/ANBM/Privacy phê duyệt tương ứng; monitoring/alert/runbook sẵn sàng |
| DR rehearsal/recovery (logical, không phải profile ứng dụng) | Chứng minh restore/failover, RTO/RPO và reconciliation; chỉ nhận traffic khi kích hoạt được phê duyệt | Restore isolated từ PostgreSQL PITR/backup cùng pipeline/history/checklist/outbox; dependency endpoint/credential DR theo runbook | Bản restore production được cô lập, mã hóa, audit; purge lại sau drill; không copy sang local/STAG | Break-glass có thời hạn và audit; network deny-by-default; key/restore permission tách quyền | Target dự kiến RTO ≤60 phút, RPO ≤5 phút; File/OCR theo SLA ngoài | Không phát Kafka/notification thật trong rehearsal nếu chưa fencing; sau activation phải đối soát lease/outbox/idempotency | DBA/Vận hành/System Owner ký biên bản drill, row/constraint/business reconciliation đạt và restore environment được thu hồi an toàn |

`STAG/UAT` chỉ được coi là production-like cho một gate khi chính control/dependency
liên quan đã bật với semantics tương đương PROD. Việc dùng cùng tên profile không
tự chứng minh parity; image digest, migration version, feature/security config và
contract version phải được lưu trong release evidence.

## 10.2 Production Deployment Diagram (CI/CD)

```mermaid
flowchart LR
    Git[Source + Reviewed Change] --> CI[Build / Test / Scan / SBOM]
    CI --> Image[Immutable Image]
    Image --> Deploy[Deployment Pipeline]
    Migration[Liquibase Job] --> DB[(PostgreSQL)]
    Deploy --> Core[Dossier Core Replicas]
    Core --> DB
    Core --> Redis[(Redis)]
    Core --> Kafka[(Kafka)]
    Core --> Ext[Enterprise Services]
    subgraph Channels[Inbound Channels]
        direction TB
        Agent[Kênh Agent / Back Office]
        Market[Kênh Market]
    end
    Agent --> AgentAPI[Agent API Replicas]
    Market --> MarketAPI[Market API Replicas]
    AgentAPI --> Core
    MarketAPI --> Core
    Core --> File[File Management]
    Core --> OCR[vhm-ocr-ekyc]
    OCR --> File
```

Sơ đồ là topology logic. Số replica, AZ, CPU/RAM, connection pool, Kafka partition, Redis mode và ingress/network policy phải được capacity model và platform baseline phê duyệt; không suy diễn từ môi trường local.

## 10.3 Deployment Strategy

| **Thành phần/thay đổi** | **Chiến lược triển khai** | **Downtime kỳ vọng** | **Chiến lược rollback** | **Cửa sổ triển khai** | **Phê duyệt production** |
| --- | --- | --- | --- | --- | --- |
| Dossier Core API replicas | Canary trên image digest mới, đạt health/error/latency/security gate rồi rolling; readiness chỉ bật sau dependency/config check, graceful shutdown drain request | 0 downtime có kế hoạch; không nhận traffic khi pod chưa ready | Dừng rollout, chuyển traffic khỏi canary và triển khai lại image digest liền trước; chỉ hợp lệ khi DB/event/API contract còn backward-compatible | Release window đã duyệt, ngoài peak create/submit/review/report | **Y** — System Owner + Vận hành; ANBM nếu thay đổi security/data path |
| Outbox/notification relay và reminder/assignment scheduler trong modular monolith | Cùng artifact với Core; rolling với DB claim/lease, scheduler lock và graceful stop để không có hai owner xử lý cùng row | 0 downtime có kế hoạch; backlog có thể tăng tạm thời nhưng không mất intent | Tắt feature/worker role mới, rollback image, tiếp tục claim/replay từ PostgreSQL; không xóa/đánh dấu thành công thủ công khi delivery chưa đối soát | Không trùng peak notification/reminder và maintenance Kafka/Message/TTOL | **Y** — Backend + Vận hành; Message/Kafka owner khi đổi contract/quota |
| Liquibase/PostgreSQL schema, constraint và index | Job migration chạy đúng một lần trước khi pod mới nhận traffic; expand/migrate/contract, preflight lock/duplicate/size trên snapshot gần production | 0 downtime với migration tương thích ngược; migration lock vượt budget phải dừng trước PROD | Rollback application về version tương thích và forward-fix migration; không dùng down migration phá hủy trong rollback window; PITR marker trước thay đổi rủi ro | Đầu release window, trước canary; index/DDL nặng có DBA window riêng | **Y** — DBA + System Owner + Vận hành; Privacy/ANBM nếu chạm dữ liệu nhạy cảm |
| Pipeline YAML và JSON Schema versioned | Deploy version mới ở trạng thái inactive, validate/contract/E2E trên STAG rồi activate chỉ cho hồ sơ tạo mới; definition đã được dossier pin là immutable | 0 downtime; hồ sơ cũ tiếp tục version cũ | Chuyển active version về bản trước cho hồ sơ mới; không đổi version hồ sơ đang chạy. Migration hồ sơ là quy trình ngoại lệ có dry-run/rollback riêng | Cùng release hoặc configuration window có audit; tránh activate giữa đợt submit lớn | **Y** — Product/BA + Backend + QA; Architecture khi đổi stage/semantics |
| Feature flag và externalized configuration | Staged rollout theo môi trường/canary; flag có owner, default PROD, expiry và dependency readiness; secret chỉ cấp runtime | 0 downtime nếu config reload được chứng nhận; nếu cần restart thì rolling | Trả flag/config về giá trị trước, disable integration publisher/auto-assignment theo runbook; không dùng flag để bỏ security invariant bắt buộc | Change window theo mức rủi ro; security flag không được đổi ad-hoc ngoài quy trình khẩn cấp có audit | **Y** — owner chức năng + Vận hành; ANBM cho signature/replay/file/security control |

`Y` áp dụng cho production. Build artifact/image phải bất biến và được quảng bá từ
STAG sang PROD, không build lại. CI bắt buộc compile/unit/integration/contract,
Liquibase validation, SAST/SCA/license, secret/container/IaC scan, SBOM và các gate
PII/log liên quan. Rollback ứng dụng chỉ an toàn khi schema, event và integration
contract còn tương thích ngược; destructive cleanup chỉ chạy sau compatibility
window và retention/legal-hold gate.

### Quản lý cấu hình

| **Nhóm** | **Production gate** |
| --- | --- |
| Security signature/actor | Bật và required; secret khác local; nonce store HA. |
| Form validation | Chốt bật JSON Schema sau compatibility test; structural guard luôn bật. |
| File validation | Bật; non-OCR qua File Management, media OCR qua `vhm-ocr-ekyc`. |
| Outbox Kafka | Chốt topic/ACL/schema rồi bật publisher; dashboard backlog trước go-live. |
| Notification relay | Template/recipient/dedupe/retry được contract test. |
| OCR | Base URL, workload IAM và timeout trỏ `vhm-ocr-ekyc`; direct provider config bị loại bỏ. |
| Auto-assignment | Roster/permission/counter có fallback manual và alert. |

## 10.4 Infrastructure & Network Security

### Hạng mục bảo mật hạ tầng

| **Hạng mục** | **Giải pháp** | **Thông số/yêu cầu cấu hình** | **Phạm vi/owner** | **Bằng chứng production** |
| --- | --- | --- | --- | --- |
| Public edge, WAF, DDoS và anti-bot | Đặt tại public ingress của Agent API/Market API; Dossier Core không mở public endpoint | WAF block mode với managed rules/method/path allowlist; DDoS L3/L4 và L7 rate limit; anti-bot/challenge theo channel, không áp dụng cho workload route | Agent/Market API + Platform Security; Core chỉ nhận traffic nội bộ | Edge topology, WAF/rate-limit policy, penetration test và alert drill |
| Network segmentation | Private subnet/namespace, security group/network policy deny-by-default; DB/Redis/Kafka không public | Chỉ mở source, destination, port và protocol trong ma trận luồng mạng; không cấp public IP/route cho Core/data stores | Platform/DevOps cho Core, PostgreSQL, Redis, Kafka | Network policy/security group export, reachability và unauthorized-path test |
| Ingress allowlist | Chỉ workload identity riêng của Agent API và Market API gọi Core; channel người dùng không gọi Core trực tiếp | Tách client ID/credential/audience theo BFF; TLS + HMAC/signed actor; Basic username khớp registered client; body/size/rate limit | Backend + ANBM | Positive/negative mTLS/HMAC/client/channel tests và replay/IDOR evidence |
| Egress control/SSRF | Firewall/service mesh policy mặc định từ chối, chỉ allowlist endpoint enterprise đã chốt | Core chỉ tới PostgreSQL, Redis, Kafka, File Management, `vhm-ocr-ekyc`, TTOL, Message Delivery, market/project source, secret/KMS và observability; không gọi URL do client cung cấp; provider OCR chỉ do OCR service truy cập | Platform/ANBM + integration owner | Egress policy, DNS/endpoint inventory, SSRF test và denied-egress alert |
| Workload Identity & IAM | Danh tính workload ngắn hạn, least privilege, tách runtime và deployment identity | Quyền DB schema, Redis keyspace/command, Kafka topic/group, File path/API, OCR scope và secret được cấp riêng; không dùng cloud access key tĩnh | Platform IAM + Backend/owner dependency | IAM matrix, unused/excess permission review, rotation/revocation test |
| PostgreSQL security | Private endpoint, TLS, DB role riêng, encryption at rest/backup, audit và PITR | App role không DDL/superuser; Liquibase role tách; connection limit/timeout; administrative access qua approved path; row/data export có audit | DBA/Platform | TLS/role grant evidence, encryption/PITR attestation, restore drill và access review |
| Redis security | Private endpoint, TLS, ACL, command/keyspace tối thiểu và HA theo replay requirement | Cấm public/anonymous/default credential; persistence/backup chỉ khi policy cho phép; không lưu OCR/PII; replay store lỗi thì fail closed | Platform Redis + Backend/ANBM | ACL/TLS/HA config, failover/replay test và data scan |
| Kafka security | Private broker, TLS, producer ACL đúng topic, consumer group riêng và retention/DLQ có policy | Không auto-create topic ở PROD; event schema allowlist không PII/path/URL; quota, partition và retention `TBD` theo capacity | Kafka Platform + Backend | Topic/ACL/schema inventory, duplicate/replay test, payload/DLQ scan và lag alert |
| File/OCR boundary | Private service/API hoặc approved enterprise ingress; presigned URL đúng object/method và TTL ngắn | Core credential File chỉ cho tài liệu không OCR; media OCR đi qua `vhm-ocr-ekyc`; owner/upload-grant bắt buộc trước attach/download; Core không có provider OCR credential | File/OCR owner + Backend/ANBM | Cross-owner negative E2E, URL expiry/scope, encryption/delete evidence và contract approval |
| Secrets, key và certificate | Secret manager/KMS/certificate manager cấp runtime; không đưa vào source/image/manifest/ConfigMap/log | Secret tách environment/client, owner + rotation + emergency revoke; certificate auto-renew; key decrypt chỉ cho workload cần thiết | Platform KMS + Vận hành + ANBM | Secret scan, runtime injection, key/cert rotation và revocation drill |
| Container/Kubernetes hardening | Immutable signed image, non-root, read-only filesystem khi khả thi, dropped capabilities, seccomp, resource request/limit và admission policy | Không privileged/host network/host path; image digest allowlist, vulnerability SLA, namespace/service-account tách; temp volume có quota và không giữ PII | Platform/DevSecOps + Backend | Image signature/SBOM, admission/IaC scan, runtime policy và vulnerability report |
| Security monitoring/SIEM | Thu thập WAF/IAM/network/Kubernetes/auth failure và policy denial dưới dạng metadata | Không capture body/PII/token/path/presigned URL; alert auth/replay/IDOR-like spike, denied egress, privilege/config drift; retention theo policy | SOC/ANBM + Vận hành | SIEM field allowlist, alert routing/on-call, tabletop/incident drill và log DLP scan |
| Data zone, backup và export | Backup/log/trace/report/ZIP/XLSX ở region/bucket/private store được duyệt, mã hóa và có retention | Không tải production data sang local/non-prod; export có TTL, quota, authorization và audit; backup key/restore access tách quyền | DBA/Vận hành/File/Privacy | Data-residency approval, access log, purge/restore-and-repurge và export expiry evidence |

### Ma trận luồng mạng

| **Nguồn** | **Đích** | **Giao thức/dữ liệu** | **Kiểm soát bắt buộc** |
| --- | --- | --- | --- |
| Market client | Market API | HTTPS business request/upload orchestration | Customer authentication, data-subject/object authorization, WAF/DDoS/rate limit và session control |
| Agent/Back Office client | Agent API | HTTPS business request/upload orchestration | User authentication, BUS role/scope, WAF/DDoS/rate limit và session control |
| Market API | Dossier Core | HTTPS JSON + signed Market subject context | Allowlisted workload/client ID, TLS, HMAC/body hash, timestamp/JTI/nonce, channel/owner binding và body limit |
| Agent API | Dossier Core | HTTPS JSON + signed Agent actor context | Allowlisted workload/client ID, TLS, HMAC/body hash, timestamp/JTI/nonce, BUS role/scope/visibility và body limit |
| Dossier Core | PostgreSQL | TLS database protocol; aggregate/outbox metadata | Private network, app/Liquibase role tách, least privilege, pool/timeout, encryption/audit/PITR |
| Dossier Core | Redis | TLS; nonce/replay/counter/cache metadata | Private endpoint, ACL/keyspace, TTL, no PII và fail-closed cho replay control |
| Dossier Core | Kafka | TLS; opaque domain event metadata | Producer topic ACL, schema allowlist, idempotent/outbox relay, quota/retention và no PII/path/URL |
| Dossier Core | File Management | HTTPS metadata/prepare/verify/download/store artefact | Workload identity, exact object/grant, MIME/size/checksum, timeout và no arbitrary URL |
| Dossier Core | `vhm-ocr-ekyc` | HTTPS prepare-upload, `POST /ocr` với `subjectRef` đầu vào và `/ocr/result`; không có OCR-confirm riêng | Workload IAM + signed actor/purpose, timeout/idempotency, no-cache/no-body-log; không gọi provider trực tiếp |
| Dossier Core | TTOL/Market/Message Delivery | HTTPS/Thrift business metadata hoặc notification intent | Workload identity, endpoint allowlist, field minimization, timeout/retry policy và no credential/PII log |
| Dossier Core và platform | Observability/SIEM | TLS, technical telemetry allowlist | Chỉ metadata; không body/PII/token/file URL; retention, RBAC và DLP scan |

Mọi `TBD` về CIDR/namespace, port, certificate issuer, WAF/rate-limit threshold,
Kafka ACL/retention, KMS key và region phải được Platform, Vận hành, ANBM và data
owner phê duyệt trước production; sơ đồ logic không thay thế IaC/network policy
evidence.

## 10.5 Migration Strategy

Liquibase quản lý schema theo phiên bản. Baseline hiện có các nhóm migration cho dossier, permissions, pipeline projection, notification outbox, checklist, source/PIC/audit và race guards. Migration production phải:

1. Chạy thử trên snapshot có kích thước gần production và kiểm tra lock duration.
2. Dùng expand/migrate/contract cho thay đổi không tương thích.
3. Xác minh unique partial index bằng preflight duplicate report trước create index.
4. Có backup/PITR marker và rollback application plan; rollback DDL chỉ dùng khi thực sự an toàn.
5. Đối soát row count, constraint/index, version và sample business query sau deploy.

Phát hành pipeline: version mới được validate và deploy ở trạng thái inactive, chạy contract/E2E trên STAG rồi mới activate cho hồ sơ tạo mới. Rollback chỉ chuyển active version về definition trước đó cho hồ sơ mới; không đổi version của hồ sơ đã tạo. Definition cũ phải tiếp tục được phục vụ cho đến khi hết hồ sơ active và hoàn tất retention/audit. Migration hồ sơ đang chạy là quy trình ngoại lệ, cần mapping state/reviewer, dry-run, đối soát, audit và rollback riêng.

Migration OCR: Dossier Core thay direct OCR provider bằng client `vhm-ocr-ekyc` sau khi OCR contract, IAM và E2E được duyệt; các client File Management tiếp tục phục vụ tài liệu không OCR. Agent API và Market API tiếp tục chỉ gọi Dossier Core. Có thể chạy so sánh OCR ở chế độ quan sát nếu được phê duyệt nhưng không dual-write kết quả vào hai nguồn; sau khi ổn định phải loại bỏ provider credential, endpoint và client OCR legacy khỏi Dossier Core.

Migration tập trung OCR/PII dùng expand/migrate/contract: đóng băng issuer/format `subjectRef`, bind/backfill reference từ nguồn định danh domain có thẩm quyền và truyền cùng giá trị vào `POST /ocr`; chuyển đọc kết quả CCCD sang `/ocr/result`, sau đó chặn new-write PII và purge các trường OCR/PII legacy khỏi `form_data`, checklist, outbox, audit/report staging. Phone/email legacy phải chuyển sang nguồn Applicant/Customer Profile được phê duyệt hoặc purge, không chuyển sang OCR. Job purge phải idempotent, có checkpoint/đối soát và tuyệt đối không ghi giá trị bị xóa vào log/dead-letter. Unique guard chỉ chuyển từ CCCD sang `subjectRef+projectId` sau khi mapping đầy đủ và constraint mới đã validate. Backup cũ chứa PII phải được cô lập và hết hạn theo policy phối hợp với OCR/File/Privacy; rollback không được tái tạo luồng dual-write.

# 11. Cost & Capacity/Performance

## 11.1 Capacity/Performance

Không dùng mục tiêu `200 req/s` hoặc P95 làm cam kết khi chưa có workload model được phê duyệt. Capacity plan phải tách ít nhất:

| **Workload** | **Đơn vị đo bắt buộc** | **Điểm nghẽn cần kiểm thử** |
| --- | --- | --- |
| Create/update/submit | TPS, P95/P99, error rate | DB transaction, JSONB, File validation và external project call. |
| List/detail/statistics | Concurrent users, page size, P95 | JSONB query/index, visibility predicate, N+1. |
| Pipeline action | Actions/minute, contention | Optimistic lock, reviewer/unit unique, notification intent. |
| Outbox/reminder | Events/minute, oldest age, recovery time | Batch lock, Kafka/Message quota. |
| Report/download | Rows/file size/concurrency | Memory, temp storage, File/Syncfusion latency. |
| OCR | Request/minute và polling rate | Dossier Core và `vhm-ocr-ekyc`; Core truyền `Retry-After`, Agent API/Market API phải tuân thủ. |

Trước production, Product cung cấp MAU/DAU, hồ sơ/ngày, peak factor, tài liệu/hồ sơ, retention và report size. Vận hành/DBA chốt pool, timeout, batch, resource request/limit và headroom; QA lưu bằng chứng load/soak test.

## 11.2 Cost

### Giả định estimate sơ bộ

Estimate dưới đây là **budget envelope, không phải báo giá**. Baseline giả định
10.000 hồ sơ hoàn tất/tháng, workload target mục 4.1, Core chạy 2 pods thường trực
(mỗi pod khoảng 1 vCPU/2 GiB) và HPA tối đa 6 pods, PostgreSQL HA/PITR, Redis HA,
Kafka dùng chung, 100–300 GiB log/metric/trace mỗi tháng và một region. Giá chưa
gồm thuế, enterprise support, private discount/Savings Plan, second-region DR và
nhân sự vận hành.

Để có order-of-magnitude, bảng dùng mô hình public on-demand kiểu AWS tại thời điểm
soạn thảo; cloud/region thực tế chưa được chốt. AWS tính riêng EKS control plane và
worker resource; RDS, ElastiCache và MSK thay đổi theo region/instance/storage;
observability và object storage tính theo usage. Nguồn tham chiếu:
[EKS](https://aws.amazon.com/eks/pricing/),
[RDS PostgreSQL](https://aws.amazon.com/rds/postgresql/pricing/),
[ElastiCache](https://aws.amazon.com/elasticache/pricing/),
[MSK](https://aws.amazon.com/msk/pricing/),
[CloudWatch](https://aws.amazon.com/cloudwatch/pricing/),
[S3](https://aws.amazon.com/s3/pricing/) và
[AWS Pricing Calculator](https://calculator.aws/). FinOps phải thay bằng
rate card/contract chính thức của VHM trước khi phê duyệt.

| **Cost driver trực tiếp của Dossier Core** | **Giả định sizing** | **Estimate USD/tháng** | **Biến số chính/ghi chú** |
| --- | --- | ---: | --- |
| Compute/Kubernetes allocation | 2 × 1 vCPU/2 GiB baseline, HPA tối đa 6; rolling/canary headroom; phần phân bổ control plane/worker | **150–500** | Kiến trúc node/Fargate, utilization, shared cluster allocation, Savings Plan và thời gian scale-out |
| PostgreSQL HA, storage và PITR | Managed Multi-AZ tương đương 2–4 vCPU/8–16 GiB, 100–300 GiB storage + backup | **350–1.000** | Region, instance family, IOPS, backup retention, cross-AZ/PITR và growth JSONB/history/outbox |
| Redis HA allocation | Hai node hoặc managed/serverless tương đương, dataset <5 GiB; TLS/backup nếu bật | **80–300** | Dedicated so với shared, engine/node class, cross-AZ traffic và retention snapshot |
| Kafka allocation | Topic/consumer group dùng chung, metadata event nhỏ, retention 3–7 ngày dự kiến | **80–400** | Shared chargeback so với dedicated broker, partitions, throughput, retention/DLQ và cross-AZ traffic |
| Observability/SIEM | 100–300 GiB log/metric/trace mỗi tháng, dashboard/alert và DLP scan | **75–350** | Log verbosity/retention, custom metrics cardinality, trace sampling và SIEM ingestion |
| Network, load balancing, KMS, secret và certificate | Internal ingress/egress, cross-AZ, KMS/secret operations và certificate lifecycle | **75–300** | Region/topology, private endpoint/service mesh, cross-AZ/egress volume và key count |
| Backup/export/temp artefact allocation | DB backup ngoài free allowance, report/ZIP/XLSX private TTL store và restore drill overhead | **50–250** | Report volume/size, retention, backup growth, request/transfer và lifecycle tiering |
| **Subtotal trực tiếp** | Không gồm các capability/service bên ngoài bên dưới | **860–3.100** | Khoảng rộng do chưa chốt platform/region và tỷ lệ shared-service allocation |
| Contingency | 20% subtotal cho peak/headroom và sai số sizing ban đầu | **170–620** | Không thay thế FinOps reserve hay incident/DR budget |
| **Budget sơ bộ PROD/tháng** | Subtotal + contingency | **1.030–3.720 USD** | Làm tròn khi lập ngân sách: **~1,0k–3,7k USD/tháng** |

### Chi phí ngoài phạm vi tổng trên

| **Capability/chi phí** | **Cách hạch toán yêu cầu** |
| --- | --- |
| `vhm-ocr-ekyc` và provider OCR | OCR service owner cung cấp fixed allocation + unit price/request/document CCCD; Dossier không nhân đôi provider cost |
| File Management/object binary | File owner tính storage/request/egress theo số file, dung lượng trung bình và retention; bảng trên chỉ tính artefact export nhỏ thuộc Core |
| Message Delivery | Message owner tính theo email/ZNS/SMS/notification accepted/delivered và retry; Core chỉ tính relay compute nhỏ |
| Agent API/Market API/UI | Hạch toán tại repository/channel owner; không đưa compute/WAF/CDN của hai BFF vào Core |
| Shared Kafka/Redis/Kubernetes/observability | Platform/FinOps cung cấp chargeback rule; không vừa tính full cluster vừa tính allocation cho Core |
| Nhân sự, license và support | Syncfusion/license enterprise, on-call/DBA/SOC, support plan, VAT và commercial discount phải được Procurement/FinOps bổ sung |

Với baseline 10.000 hồ sơ hoàn tất/tháng, unit cost hạ tầng trực tiếp dự kiến là
**0,10–0,37 USD/hồ sơ** (`budget PROD / completed dossiers`), chưa gồm OCR, file,
message và channel. Chỉ số này không tuyến tính ở volume thấp do chi phí HA cố định;
ở volume cao phải cộng storage/egress, DB/Kafka/Redis scale step và capacity
headroom. Tổng ngân sách gồm STAG production-like dự kiến tăng thêm khoảng
**30–50% chi phí PROD**, tùy mức chia sẻ data store/dependency.

Trước khi chuyển TDD sang `APPROVED`, Product cung cấp hồ sơ/tháng, file/hồ sơ,
dung lượng/retention, report/export và peak factor; Platform cung cấp topology;
FinOps chạy calculator/rate card chính thức và chốt ba kịch bản `P50`, `P90` và
stress. Budget alert đề xuất ở 80%/100% forecast và theo dõi riêng cost/hồ sơ,
PostgreSQL storage growth, log ingestion, Kafka retention và external capability
unit cost.

# 12. Scalability & Reliability

## 12.1 Scaling Strategy

- Scale ngang HTTP replicas; không lưu session/actor state trong process.
- Scale relay/scanner theo leader/DB claim để không xử lý một row đồng thời.
- Dùng pagination có giới hạn và index cho filter phổ biến; report lớn chạy với quota/batch phù hợp.
- Cache chỉ tối ưu read; DB vẫn là authority. Cache miss/stale không được mở rộng quyền.
- Dossier Core điều phối polling và truyền `Retry-After`; Agent API/Market API phải tuân thủ để tránh polling storm. OCR worker/capacity thuộc service dùng chung.
- Auto-assignment Redis counter không được chặn manual assignment khi Redis/roster suy giảm.

## 12.2 Reliability

| **Failure mode** | **Hành vi an toàn** | **Phục hồi** |
| --- | --- | --- |
| Core replica dừng giữa request | Transaction rollback hoặc commit nguyên tử | Client retry create bằng idempotency key; reload version. |
| PostgreSQL không sẵn sàng | Fail request; không nhận mutation giả thành công | HA/failover/PITR theo runbook. |
| Kafka không sẵn sàng | Outbox backlog tăng, hồ sơ vẫn commit | Relay phát lại; alert oldest age. |
| Message Delivery lỗi | Notification retry/FAILED | Manual replay/runbook; không rollback transition. |
| Redis lỗi | Security replay fail closed; assignment/cache suy giảm | HA/failover; manual assignment. |
| TTOL lỗi | Không lấy được danh sách nhân sự để auto-assign PTT/SXD | Giữ hồ sơ ở trạng thái chưa phân công; lead phân công thủ công và có cảnh báo. |
| `vhm-ocr-ekyc` lỗi | Không prepare/tạo/poll/đọc được kết quả OCR | Retry theo `Retry-After` hoặc xác minh thủ công theo runbook OCR; không nhập/lưu PII thay thế tại Dossier và không tự reject. |
| File Management lỗi | Không prepare/verify/download tài liệu không OCR; nhánh OCR cũng có thể suy giảm | Retry hữu hạn; không attach path chưa xác minh; không bypass qua đường tích hợp khác. |

## 12.3 Sao lưu và phục hồi

- PostgreSQL cần automated backup, PITR và restore drill có bằng chứng.
- RPO phải bao phủ dossier, pipeline history, checklist, reviewer và outbox trong cùng database.
- Object file do File Management backup/retention; restore phải bảo toàn reference của cả tài liệu thường và media OCR hoặc có reconciliation.
- Redis cache/counter có thể rebuild; nonce/replay trong cửa sổ security cần fail-safe khi phục hồi.
- Kafka không thay thế DB backup; outbox là nguồn replay event trong retention window.
- Kiểm thử DR phải bao gồm backlog relay, notification và consistency sau restore, không chỉ khởi động ứng dụng.

# 13. Observability & Monitoring

## 13.1 Yêu cầu nền tảng

- Correlation ID xuyên kênh → Agent API/Market API → Core → external calls/outbox, không dùng PII.
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
| External | File Management, `vhm-ocr-ekyc`, TTOL và Message availability, latency, timeout và error class. |
| Security | Invalid signature, stale timestamp, nonce replay, Market subject-binding denial hoặc Agent actor expiry/role/scope/visibility denial. |

Label không được chứa dossier ID, actor ID, project ID có cardinality cao hoặc PII. Business drill-down dùng log/audit có kiểm soát thay vì metric label.

## 13.3 Cảnh báo

| **Cảnh báo** | **Điều kiện nguyên tắc** | **Ưu tiên** |
| --- | --- | --- |
| Mutation error/latency burn | Vượt SLO theo nhiều cửa sổ | P1/P2 theo burn rate |
| PostgreSQL saturation/lock | Pool gần cạn, lock/transaction bất thường | P1 |
| Outbox/notification backlog | Oldest age vượt delivery SLO | P1/P2 |
| Reviewer chưa được assign | Stage age vượt ngưỡng nghiệp vụ | P2 |
| Signature/replay anomaly | Tăng đột biến hoặc client bị deny liên tục | Security incident |
| Media/OCR/TTOL dependency | Error/timeout vượt budget | P2; P1 nếu chặn toàn bộ submit |
| Reminder missed | Scan không chạy hoặc due row quá hạn | P2 |

## 13.4 SLI/SLO

SLI bắt buộc: availability của read/mutation, successful submit/transition, event delivery latency, notification intent delivery, reminder timeliness và data consistency. Giá trị tại mục 4.1 là target **DỰ KIẾN**; measurement window/error budget/maintenance exclusion/owner và baseline production cuối vẫn `TBD`. Không được chuyển trạng thái `APPROVED` nếu workload forecast chưa được chốt hoặc STAG/load/soak test không chứng minh đạt target trên cấu hình production-like.

# 14. Operational Readiness

## 14.1 RTO & RPO

| **Hạng mục** | **Mục tiêu** | **Trạng thái** |
| --- | --- | --- |
| RTO dossier core | **DỰ KIẾN ≤60 phút** | Chưa là baseline; System Owner/Vận hành phải phê duyệt và diễn tập |
| RPO PostgreSQL | **DỰ KIẾN ≤5 phút** | Chưa là baseline; DBA phải phê duyệt và có bằng chứng PITR |
| Event/notification recovery | Trong delivery SLO được duyệt | Cần backlog replay drill |
| File/object recovery | Theo File Management SLA và retention | Contract ngoài |
| OCR recovery | Theo `vhm-ocr-ekyc` SLO; OCR capability giữ lifecycle/result theo `referenceId` | Contract ngoài |

## 14.2 Runbook bắt buộc

- Invalid HMAC/Agent actor/Market subject signature, subject-binding denial, nonce store lỗi và luân chuyển/revoke secret.
- PostgreSQL failover/PITR, Liquibase lỗi, unique-index migration và data reconciliation.
- Kafka down, outbox backlog, duplicate publish và poison event.
- Message Delivery lỗi, notification `FAILED`, dedupe và manual replay.
- Reviewer roster/Redis counter lỗi, unassigned dossier và manual assignment.
- Nguồn lịch nghiệp vụ/cache lỗi ảnh hưởng SLA reminder.
- File ownership/existence/download incident và object mồ côi.
- `vhm-ocr-ekyc` unavailable, OCR stuck/terminal error, `subjectRef` mismatch và migration rollback.
- OCR/PII xuất hiện trong Dossier DB/cache/log/export staging/event hoặc credential xuất hiện ngoài secret manager.
- Rollback release khi migration đã chạy.

## 14.3 Danh sách kiểm tra sẵn sàng cơ sở

- Owner/on-call/escalation matrix cho core và từng dependency.
- Production config review chứng minh signature, actor, file validation, schema validation và relays đúng default.
- Dashboard/cảnh báo đã fire thử và route đúng người.
- Backup/restore, deployment rollback, purge dữ liệu OCR/PII legacy và backlog replay đã diễn tập.
- OpenAPI/event/schema/pipeline version được đóng băng và contract test.
- Privacy retention/deletion/export access được phê duyệt.
- Mọi vấn đề Critical/High tại mục 16 đã đóng hoặc có risk acceptance hữu hạn, owner và expiry.

# 15. Testing & Quality Strategy

## 15.1 Phạm vi kiểm thử bắt buộc

Bộ kiểm thử phải bao phủ invariant domain, database concurrency, API/security contract, external integration, outbox/recovery, performance và OAT/DR. Unit test không thay thế PostgreSQL integration test, contract test, E2E hoặc failure drill.

## 15.2 Cổng chất lượng

| **Lớp kiểm thử** | **Phạm vi bắt buộc** | **Cổng** |
| --- | --- | --- |
| Unit/domain | State/action, Market subject ownership, Agent BUS role/scope/ownership, checklist, code/unit, OCR/PII persistence denylist và error mapping | Bắt buộc |
| Database integration | Migration/purge legacy, `subjectRef` partial index, advisory/row lock, optimistic lock, cascade | Bắt buộc |
| Concurrency | Concurrent idempotent create, duplicate identity/project, allocate unit, reviewer claim | Bắt buộc |
| Pipeline definition | Schema, graph integrity, reachability, role/stage reference, revision loop và published-version immutability | Bắt buộc |
| API/contract | Agent API/Market API ↔ Core: DTO/header/status/error, channel-specific signed claims, source mapping và backward compatibility | Bắt buộc |
| Security | Market subject binding/IDOR; Agent role/scope/visibility; signature, nonce replay, actor expiry, body context injection và no-copy PII | Bắt buộc |
| External contract | File Management, `vhm-ocr-ekyc`, TTOL và Message Delivery | Bắt buộc |
| Outbox/reliability | Rollback, broker/send lỗi, publish lặp, retry/FAILED, backlog recovery | Bắt buộc |
| E2E | Create DRAFT → upload → PATCH → submit → multi-stage decision/revision | Bắt buộc |
| Performance/soak | Workload model mục 11, report/download và polling OCR | Bắt buộc |
| OAT/DR | Deploy/rollback, restore, legacy PII purge, OCR outage/degradation, alert/runbook | Bắt buộc |

## 15.3 Kịch bản kiểm thử trọng yếu

- Create `{}` trả DRAFT/version đúng DB; create không sinh checklist.
- Concurrent create cùng actor/key trả cùng dossier; actor khác không replay được key.
- `MARKET` thiếu key bị từ chối; Agent API/Market API phải gán đúng source theo channel context và từ chối client tự khai nguồn khác.
- Market Customer không có role vẫn tạo/xem/sửa/submit đúng dossier của mình theo state; customer khác hoặc subject mismatch bị từ chối ở list/detail/mutation/export.
- Market request mang `APPLICANT_AGENT`, `ALL/TEAM/ASSIGNED`, owner hoặc data subject tự khai phải bị từ chối; Core không yêu cầu role Agent hợp lệ cho Market owner action.
- Agent action phải khớp BUS role, project/team scope và ownership; role hợp lệ nhưng sai scope, role hết hiệu lực hoặc role lạ đều bị deny-by-default.
- Agent reviewer xử lý dossier `source=MARKET` chỉ khi policy BUS/project scope cho phép; `source` không tự cấp quyền.
- Hai hồ sơ active cùng `subjectRef` authoritative+dự án bị chặn ở create khi đã có reference, bước bind trước OCR, full update, submit và DB race guard.
- Client tự khai/sửa `subjectRef` hoặc gửi CCCD, họ tên, ngày sinh, contact, OCR result/media metadata vào persistence API phải bị từ chối; Core bind reference từ signed domain identity context và truyền đúng cùng giá trị trong `POST /ocr`.
- Database dump, outbox, audit, cache, log và report staging không chứa media/kết quả/PII applicant; chỉ có opaque reference và business metadata allowlist.
- List/detail/statistics/export chỉ có business metadata allowlist; contact không được gửi sang OCR và không có search/hydrate PII nếu chưa có contract nguồn Applicant/Customer Profile riêng.
- Full update file không tồn tại rollback cả dossier/checklist/outbox; xóa documents reset/delete projection đúng.
- Submit không checklist/thiếu required trả `11017/11018`; required complete mới chuyển trạng thái.
- If-Match cũ, concurrent command, allocate cùng căn và double claim không làm lost update.
- Mọi state/action/role/ownership branch của pipeline, gồm revision loop và revoke sau approve.
- Thêm một cấp duyệt vào version mới: hồ sơ mới đi qua đủ assignment/approve/revision/notification/timeline; hồ sơ version cũ không đổi hành vi.
- Bỏ một cấp duyệt ở version mới: transition được nối lại, không có state cụt; hồ sơ cũ đang ở cấp bị bỏ vẫn tiếp tục xử lý bằng version cũ.
- Definition lỗi, sửa published version hoặc thiếu policy bắt buộc phải fail trước activation; rollback active version không remap hồ sơ đã tạo.
- Signature/body hash/timestamp/nonce/actor JTI/visibility negative matrix và không có body actor spoofing.
- Outbox crash trước/sau broker acknowledgement có thể phát lặp nhưng không mất event.
- Notification retry/dedupe/FAILED và reminder T+6/T+18 qua ngày nghỉ/cycle mới.
- OCR: media reference sai quyền/không tồn tại, idempotent create, `202`/`Retry-After`, polling `QUEUED/PROCESSING/terminal`, domain `CONFIRM_AND_APPLY` đọc lại `COMPLETED`, không gọi OCR-confirm; `subjectRef` đầu vào ổn định/non-reversible, timeout và service unavailable.
- File Management: cross-owner/cross-reference, path traversal, MIME/magic/checksum, expired presign và upload grant cho tài liệu không OCR.
- Migration trên dữ liệu trùng, rollback application và restore/PITR reconciliation.

## 15.4 Dữ liệu kiểm thử và quản lý bằng chứng

Test tự động/SIT chỉ dùng CCCD/file tổng hợp hoặc đã làm sạch. Dữ liệu cá nhân thật cần phê duyệt, kho cô lập, purpose/retention đích danh và bằng chứng xóa. Fixture của dependency phải có version, không chứa credential/PII và bao phủ cả success, timeout, malformed response, duplicate và permission denial.

Bằng chứng quality gate phải được lưu theo release, tối thiểu gồm test report, migration rehearsal, contract/E2E result, security scan, load/soak report, restore drill, dashboard/alert verification và danh sách risk acceptance còn hiệu lực.

# 16. Risks & Open Issues

## 16.1 Architecture Risks

| **Mã** | **Nhóm** | **Mô tả/ảnh hưởng** | **Mức độ** | **Giảm thiểu/điều kiện đóng** |
| --- | --- | --- | --- | --- |
| AR-001 | Toàn vẹn nghiệp vụ | Tin `documents[].isRequired` từ client có thể làm sai bộ hồ sơ được phép submit | Nghiêm trọng | Tích hợp nguồn Checklist chuẩn, snapshot/version server-side, contract test. |
| AR-002 | An toàn file | File response chưa chứng minh uploader/upload-grant owner | Nghiêm trọng | File Contract trả owner/grant và verify khi attach; negative E2E. |
| AR-003 | Determinism | Thiếu pipeline ID/version authoritative có thể route hồ sơ sai quy trình | Cao | Bắt buộc unique selection rule và fail khi kết quả không xác định. |
| AR-004 | Security | Tin `source=MARKET` từ request body có thể làm sai chính sách theo kênh | Cao | Agent API/Market API gán source server-side; Core kiểm tra signed context/client identity và có negative test chéo kênh. |
| AR-005 | Tích hợp OCR | Duy trì direct synchronous OCR sẽ phá vỡ ranh giới capability dùng chung | Cao | Chỉ tích hợp `vhm-ocr-ekyc`; E2E/contract đạt và loại bỏ legacy trước go-live. |
| AR-006 | Audit/PIC | Có `picId`/audit table nhưng chưa có use case gán/chuyển PIC hoàn chỉnh | Trung bình | Chốt owner, API, permission và audit semantics hoặc bỏ khỏi contract. |
| AR-007 | Contract | Dùng không nhất quán `required` và `isRequired` có thể tạo quyết định duyệt khác nhau | Cao | Chỉ công bố một field chuẩn trong Form/Checklist Contract và có regression test. |
| AR-008 | Notification | Schema có nhiều kênh nhưng relay hiện chỉ dispatch email | Trung bình | Chốt scope kênh; triển khai hoặc loại khỏi contract công bố. |
| AR-009 | Validation | JSON Schema enforcement có thể mặc định tắt | Cao | Chốt production default bật, compatibility test và alert config drift. |
| AR-010 | Event delivery | Kafka outbox publish có thể mặc định tắt | Cao | Production config gate, readiness/metric và backlog verification. |
| AR-011 | Privacy | Retention/deletion/legal hold/audit access giữa Dossier và OCR source chưa được định nghĩa | Nghiêm trọng | DPIA/policy, delete orchestration/runbook và bằng chứng purge/restore liên hệ thống. |
| AR-012 | Availability | Full E2E phụ thuộc File Management, `vhm-ocr-ekyc` và nhiều enterprise dependency | Cao | Sandbox/SLA, timeout/degradation, synthetic probe và runbook. |
| AR-013 | Linh động pipeline | State machine có cấu hình nhưng assignment, notification, reminder, report hoặc audit vẫn gắn cứng tên stage sẽ làm cấp duyệt mới chạy thiếu side effect | Cao | Mọi consumer dùng stage policy/metadata; quality gate bắt buộc chứng minh add/remove một cấp chỉ bằng definition mới. |
| AR-014 | Trùng lặp dữ liệu | `form_data`, checklist, outbox/reviewer/note/audit/report legacy có thể nhân bản OCR/PII khỏi nguồn tập trung | Nghiêm trọng | Chỉ lưu opaque reference, persistence denylist, migrate/purge bản sao và data scan gate trước dữ liệu thật. |
| AR-015 | Phân quyền theo kênh | Gộp Market Customer vào role `APPLICANT_AGENT` hoặc áp visibility Agent cho Market có thể gây cấp quyền sai/IDOR | Nghiêm trọng | Tách signed context/matrix; Market subject binding, Agent BUS role/scope và negative E2E riêng. |

## 16.2 Vấn đề thiết kế cần quyết định

| **Vấn đề cần quyết định** | **Owner đề xuất** | **Điều kiện đóng** |
| --- | --- | --- |
| Nguồn Checklist chuẩn, contract và version/snapshot | BA/Checklist Team/Backend | API/schema/authority được duyệt; client không tự quyết `isRequired`. |
| File ownership/upload-grant cho checklist, ZIP và XLSX | File Team/ANBM | File Contract bao phủ prepare, verify, store và download; E2E cross-owner đạt. |
| Pipeline selection authoritative | BA/Kiến trúc/Backend | Một pipeline ID/version rõ ràng từ contract/config. |
| Governance activate/deactivate/retention và migration hồ sơ đang chạy | Product/Kiến trúc/Vận hành | Quy trình phê duyệt version, rollback activation, thời gian giữ definition cũ và tiêu chí migration ngoại lệ được ký duyệt. |
| Quy tắc ánh xạ kênh sang `source=AGENT\|MARKET` | Agent API/Market API/Backend | Mỗi BFF chỉ được gán source của chính kênh mình, được ký và có negative contract test. |
| Claim định danh data subject của Market và cách bind `ownerSubject` | Market API/ANBM/Backend | Claim authoritative, ổn định, signed; create/list/detail/mutation và cross-customer E2E được duyệt. |
| Catalogue role BUS, runtime source, version và mapping role → scope/action | BUS/Agent API/Backend/ANBM | Role codes/semantics/expiry, signed context, pipeline mapping và deny-by-default contract được duyệt. |
| Agent/Back Office review dossier `source=MARKET` | BUS/Product/Backend | Chốt role/project scope và policy cross-channel; source không được dùng thay authorization. |
| Contract Dossier Core ↔ `vhm-ocr-ekyc` cho CCCD hai mặt | OCR Team/Backend/ANBM | OpenAPI L3 chốt prepare-upload, `POST /ocr` với `subjectRef` đầu vào, `/ocr/result`, IAM, failure/SLA và E2E; không giả định confirm/search/export/delete API. |
| Issuer và cơ chế bind `subjectRef` | Product/Identity owner/Agent API/Market API/Backend/ANBM | Nguồn domain có thẩm quyền, signed context, format opaque ổn định/unique/non-reversible và cross-dossier mapping được ký duyệt; client không tự khai. |
| Liên kết `ocrId` với dossier để poll/apply/submit readiness | OCR Team/Backend/Privacy | L3 chốt truyền lại hay persist opaque `ocrId`; nếu persist tại Core thì không kèm status/result/media reference, có retention/unlink và kiểm thử cross-dossier. |
| Ý nghĩa/ownership của `picId` so với stage reviewer | Product/BA/Backend | Use case và permission/audit rõ hoặc bỏ field. |
| SLO, peak workload, RTO/RPO và capacity/cost | Product/Vận hành/DBA/FinOps | Baseline số được duyệt và load/DR đạt. |
| Retention, deletion, legal hold, encryption và audit access tại OCR source | Privacy/Pháp chế/ANBM/OCR Team | Policy/DPIA/runbook và orchestration với Dossier được phê duyệt. |
| Nguồn contact và presentation applicant ngoài OCR | Product/Applicant Profile owner/Message Team/BFF/Backend/Privacy | Chốt source of truth cho phone/email, authorization/masking/notification contract; Dossier không persist và OCR không thu thập contact. |
| Notification channels và recipient authority | Product/Message Team | Contract channel/dedupe/template/address và test đạt. |
| Form schema enforcement và backward compatibility | BA/Backend/QA | Bật trên STAG, clean data report và regression đạt. |

Vấn đề mở không mặc nhiên được chấp nhận. Risk acceptance phải có owner, phạm vi, kiểm soát bù trừ, người phê duyệt và ngày hết hạn.

# Appendix

## A. Glossary

| **Thuật ngữ** | **Định nghĩa** |
| --- | --- |
| NOXH | Nhà ở Xã hội. |
| Dossier | Aggregate hồ sơ đăng ký một applicant cho một project. |
| BFF | Public boundary xác thực kênh và gọi Core bằng workload identity cùng signed security context theo channel; gồm Agent API và Market API. |
| BUS role | Role catalogue và semantics nghiệp vụ cho Agent/Back Office do BUS làm authority; được map tới scope/action của pipeline. |
| Data subject | Customer/chủ thể dữ liệu đã được Market API xác thực và dùng để giới hạn object access. |
| `ownerSubject` | Định danh data subject opaque được Core bind server-side cho dossier MARKET; khác `subjectRef` dùng liên kết OCR/duplicate. |
| PKD/PTT/SXD | Các cấp Sales/Procedure/Department-of-Construction trong pipeline. |
| Checklist | Projection tài liệu bắt buộc/trạng thái upload/review của dossier; không lưu OCR status/result. |
| Pipeline | Cấu hình state/action/role/ownership có phiên bản, thực thi trong core. |
| Agent actor context | Payload actor, BUS role, scope và visibility được Agent API ký và Core xác minh. |
| Market subject context | Payload customer/data-subject ownership không chứa business role, được Market API ký và Core xác minh. |
| Visibility | Phạm vi hồ sơ Agent/Back Office được phép đọc/xử lý; không dùng cho Market Customer. |
| Idempotency key | Khóa opaque để replay an toàn create/OCR. |
| Transactional outbox | Ghi business state và ý định phát/gửi trong cùng transaction DB. |
| `vhm-ocr-ekyc` | Capability OCR dùng chung trong phạm vi Dossier, sở hữu media reference, lifecycle và kết quả OCR chuẩn; File Management giữ byte media. |
| Kênh Market | Kênh nghiệp vụ đi qua Market API; Market API là BFF ngang hàng với Agent API và gắn `source=MARKET`. |
| Opaque reference | Identifier tương quan không nhúng PII hoặc secret. |
| PII | Dữ liệu có thể nhận diện trực tiếp hoặc gián tiếp một cá nhân. |
| `subjectRef` | Định danh applicant opaque, ổn định, không đảo ngược về PII; nguồn định danh domain có thẩm quyền cấp/bind server-side và Core truyền vào `vhm-ocr-ekyc` để tương quan/phân quyền. |

## B. References

| **Tài liệu/artefact** | **Tham chiếu** |
| --- | --- |
| L2 - Capability OCR dùng chung | [L2 - VHMKDO2O - Capability OCR dùng chung](https://vin3s.atlassian.net/wiki/spaces/VARW/pages/3014268156/L2+-+VHMKDO2O+-+D+ch+v+OCR+eKYC) |
| Pipeline Definition Social Housing v1 | Tài liệu L3 chính thức: TBD |
| Form Data Contract Social Housing v1 | Tài liệu L3 chính thức: TBD |
| Database model và migration plan | Tài liệu L3/DBA chính thức: TBD |

## C. Đầu vào bắt buộc trước production

| **Đầu vào** | **Chủ sở hữu** | **Cổng** |
| --- | --- | --- |
| Checklist authority/version/snapshot | BA/Checklist Team | Submit/UAT |
| File Management contract và ownership/upload-grant | File Team/ANBM | Attachment security và export/download không OCR |
| Pipeline schema, ID/version selection, activation và retention | BA/Kiến trúc/Vận hành | Cấu hình và thay đổi số cấp duyệt |
| OCR OpenAPI, IAM, CCCD hai mặt, `subjectRef` đầu vào, prepare-upload, `POST /ocr`, `/ocr/result` và retention/delete runbook | OCR Team/Backend/ANBM | E2E media/OCR và data ownership |
| Domain Identity contract phát hành/bind `subjectRef` | Product/Identity owner/Agent API/Market API/Backend/ANBM | Duplicate guard và OCR correlation |
| Channel-to-source mapping cho AGENT/MARKET | Agent API/Market API/Backend | Security/API approval |
| Market subject claim và `ownerSubject` binding contract | Market API/ANBM/Backend | Market authorization/IDOR gate |
| BUS role catalogue, scope và signed Agent context | BUS/Agent API/ANBM/Backend | Agent authorization/pipeline gate |
| Privacy retention/deletion, no-copy policy và purge dữ liệu legacy | Privacy/Pháp chế/ANBM/OCR Team | Dữ liệu thật |
| Workload/SLO/capacity/cost | Product/Vận hành/FinOps | Load/OAT |
| RTO/RPO/backup/restore | DBA/Vận hành | DR/OAT |
| Dashboard/alert/on-call/runbook | Vận hành | Go-live |
| Contract test File/`vhm-ocr-ekyc`/TTOL/Message/Kafka | Tích hợp/QA | Release |

## D. Danh mục quyết định kiến trúc (ADR)

| **ID** | **Quyết định** | **Cơ sở/hệ quả** | **Trạng thái** |
| --- | --- | --- | --- |
| ADR-001 | PostgreSQL là source of truth | Dossier/checklist/pipeline/history/outbox nhất quán; cần HA/PITR | CHẤP NHẬN |
| ADR-002 | Pipeline versioned thực thi trong process | Không cần Camunda/Zeebe; transition nguyên tử với dossier | CHẤP NHẬN |
| ADR-003 | Create luôn DRAFT, submit là command riêng | Hỗ trợ upload và hoàn thiện snapshot trước nộp | CHẤP NHẬN |
| ADR-004 | JSONB snapshot + schema version | Linh hoạt form; đổi lại cần schema/guard/index JSON rõ ràng | CHẤP NHẬN |
| ADR-005 | Advisory lock + actor-scoped replay + DB unique | Chống concurrent forwarding race và key reuse sai actor | CHẤP NHẬN |
| ADR-006 | Partial unique index là race guard cuối cho `subjectRef`+dự án | Không persist CCCD; nguồn định danh domain bảo đảm reference ổn định/non-reversible, OCR lưu cùng reference để tương quan, DB bảo đảm invariant local | ĐỀ XUẤT — chờ Domain Identity/OCR Contract L3 |
| ADR-007 | Transactional outbox cho event/notification | Không mất intent sau commit; chấp nhận at-least-once | CHẤP NHẬN |
| ADR-008 | Signed context theo channel và deny-by-default | Market ký data subject không role; Agent ký BUS role/scope; Core không tin security claim từ body | ĐỀ XUẤT — chờ contract ANBM/BUS/Market |
| ADR-009 | File path opaque, không kiểm tra dossier-prefix | Upload namespace độc lập; ownership phải dựa File Contract | CHẤP NHẬN có điều kiện |
| ADR-010 | OCR qua capability dùng chung `vhm-ocr-ekyc` | Dossier không sở hữu provider/worker/raw result; cần migration legacy | ĐỀ XUẤT — chờ phê duyệt |
| ADR-011 | Kết quả OCR lưu tại `vhm-ocr-ekyc`, còn `CONFIRM_AND_APPLY` thuộc domain và không PATCH PII vào snapshot dossier | Core chỉ proxy projection `/ocr/result`, đọc lại `COMPLETED` khi apply và giữ `subjectRef` đã bind; không có OCR-confirm riêng | ĐỀ XUẤT — chờ phê duyệt |
| ADR-012 | Phân tuyến file theo mục đích nghiệp vụ | Tài liệu không OCR đi File Management; media OCR đi `vhm-ocr-ekyc`, không để client tự chọn đường | ĐỀ XUẤT — chờ phê duyệt |
| ADR-013 | Cấp duyệt cấu hình và pipeline version bất biến | Add/remove bằng version mới; hồ sơ pin đúng version, consumer dùng policy/metadata thay vì tên stage | ĐỀ XUẤT — chờ phê duyệt |
| ADR-014 | Không nhân bản OCR/PII tại Dossier Core | Kết quả/payload encryption thuộc OCR, byte media/object key thuộc File; Core lưu opaque reference và baseline chỉ dùng prepare-upload, `POST /ocr`, `/ocr/result` | ĐỀ XUẤT — chờ OCR/File/ANBM/Privacy phê duyệt |
| ADR-015 | Tách ma trận authorization Market và Agent | Market authorization tại Market API theo authenticated data subject và Core recheck owner binding; Agent dùng BUS role/scope/pipeline ownership | ĐỀ XUẤT — chờ BUS/Market/ANBM phê duyệt |
