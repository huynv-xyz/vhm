# Database ERD — vhm-cobroker-core

> As-built từ Liquibase migrations và JPA entities, ngày 14/09/2026.  
> PostgreSQL schema: `cobroker_db`.  
> Đây là logical ERD của schema cuối cùng sau toàn bộ changelog; bảng đã bị drop không xuất hiện.

## Quy ước

- `PK`: khóa chính vật lý.
- `FK`: khóa ngoại được khai báo trong database.
- `UK`: unique constraint/index quan trọng.
- `REF`: liên kết logic qua ID nhưng database không khai báo FK.
- Hầu hết bảng nghiệp vụ dùng UUID và có các cột audit `created_at`, `created_by`, `updated_at`, `updated_by`; sơ đồ lược bớt các cột này để dễ đọc.
- `organization_id` là tenant boundary và phải đi cùng các truy vấn nghiệp vụ.

## 1. Sơ đồ tổng quan domain

```mermaid
flowchart LR
    A[Agency & Sale] --> PA[Project Assignment]
    A --> D[Property Distribution]
    PA --> D
    A --> S[Agency Scoring]
    D --> S
    A --> IV[Applicant / eKYC / OCR]
    A --> U[Update Request & Audit]
    D --> O[Outbox / Import / Async]
    IV --> O
```

## 2. Agency, Sale và yêu cầu cập nhật

```mermaid
erDiagram
    AGENCY_PROFILES ||--o{ AGENCY_BRANCH : "REF agency_profile_id"
    AGENCY_PROFILES ||--o{ AGENCY_COBROKER : "REF agency_profile_id"
    COBROKER_PROFILES ||--o{ AGENCY_COBROKER : "REF cobroker_profile_id"
    AGENCY_PROFILES ||--o{ UPDATE_REQUESTS : "REF agency_profile_id"
    UPDATE_REQUESTS ||--o{ UPDATE_REQUEST_RESPONSES : "REF update_request_id"
    AGENCY_PROFILES ||--o{ AUDIT_LOGS : "REF agency_profile_id"
    COBROKER_PROFILES o|--o{ COBROKER_APPLICANT_AGENCIES_SUBMISSION : "profile / live submission"
    COBROKER_APPLICANT_AGENCIES_SUBMISSION ||--o{ OUTBOX_COBROKER_APPLICANT : "FK submission_id"

    AGENCY_PROFILES {
        uuid id PK
        int organization_id UK
        varchar agency_code UK
        varchar tax_code UK
        varchar brand_name
        int team_id
        varchar profile_status
        varchar rank
        varchar exclusivity
        jsonb contact_metadata
        jsonb legal_metadata
        jsonb document_metadata
    }

    COBROKER_PROFILES {
        uuid id PK
        uuid account_id "REF"
        varchar cobroker_id UK
        varchar identity_no
        varchar full_name
        varchar phone
        varchar email
        varchar status
    }

    AGENCY_COBROKER {
        uuid id PK
        int organization_id
        uuid agency_profile_id "REF"
        uuid cobroker_profile_id "REF"
        varchar_array role_type
        varchar agent_profile_id UK
        varchar status
        jsonb identity_metadata
        jsonb employment_metadata
        jsonb project_metadata
        jsonb document_metadata
        jsonb violation_metadata
        varchar lock_reason
        timestamptz locked_at
        jsonb lock_history
    }

    AGENCY_BRANCH {
        uuid id PK
        int organization_id
        uuid agency_profile_id "REF"
        varchar branch_type
        varchar name
        varchar address
        varchar phone
        varchar status
    }

    UPDATE_REQUESTS {
        uuid id PK
        int organization_id
        uuid agency_profile_id "REF"
        varchar requester_user
        varchar status
        jsonb receiver_metadata
        jsonb proposed_changes
        jsonb proposed_branch_ops
        jsonb proposed_cobroker_ops
        varchar assignee_user
    }

    UPDATE_REQUEST_RESPONSES {
        uuid id PK
        int organization_id
        uuid update_request_id "REF"
        int seq
        text content
        varchar responder_user
    }

    AUDIT_LOGS {
        uuid id PK
        int organization_id
        uuid agency_profile_id "REF"
        varchar entity_type
        uuid entity_id "REF"
        varchar actor
        varchar action
        varchar field
        text value_before
        text value_after
        timestamptz changed_at
    }

    COBROKER_APPLICANT_AGENCIES_SUBMISSION {
        uuid id PK
        varchar submission_code UK
        uuid cobroker_profile_id "REF"
        varchar agent_profile_id
        varchar identity_no
        varchar phone
        varchar email
        varchar status
        int ekyc_verified
        jsonb identity_verification_progress
        jsonb identity_verification_metadata
        jsonb ocr_documents_verified
        jsonb manual_decisions
        jsonb audit_snapshots
    }

    OUTBOX_COBROKER_APPLICANT {
        uuid id PK
        uuid submission_id FK
        varchar event_type
        varchar status
        jsonb payload
        varchar dedup_key UK
        varchar claim_token
        int version
    }
```

### Quan hệ quan trọng

- Một người trong `cobroker_profiles` có thể có nhiều liên kết làm việc lịch sử trong `agency_cobroker`; trạng thái làm việc nằm trên liên kết.
- `agency_cobroker.project_metadata` là projection JSONB dự án của Sale, liên quan trực tiếp BDSKD-6590.
- Nhiều quan hệ agency/profile là REF mềm trong code; không có FK vật lý nên service phải luôn kiểm tra tenant và scope.

## 3. Project assignment

```mermaid
erDiagram
    AGENCY_PROFILES ||--o{ PROJECT_ASSIGNMENT_POLICY_AGENCY : "FK org + agency"
    PROJECT_ASSIGNMENT_POLICY ||--o{ PROJECT_ASSIGNMENT_POLICY_AGENCY : "FK org + policy"
    PROJECT_ASSIGNMENT_POLICY ||--o{ PROJECT_ASSIGNMENT_WINDOW : "FK policy_id"
    AGENCY_PROFILES ||--o{ PROJECT_ASSIGNMENT_JOB : "FK org + agency"
    PROJECT_ASSIGNMENT_POLICY o|--o{ PROJECT_ASSIGNMENT_JOB : "FK accepted policy"
    PROJECT_ASSIGNMENT_JOB ||--o{ PROJECT_ASSIGNMENT_JOB_ITEM : "FK job_id"
    AGENCY_COBROKER ||--o{ PROJECT_ASSIGNMENT_JOB_ITEM : "FK agency_cobroker_id"
    AGENCY_COBROKER ||--o{ USER_REGISTERED_SCOPE : "logical owner"

    PROJECT_ASSIGNMENT_POLICY {
        uuid id PK
        int organization_id UK
        varchar code
        varchar name
        boolean active
        varchar timezone
        bigint version
    }

    PROJECT_ASSIGNMENT_POLICY_AGENCY {
        uuid id PK
        uuid policy_id FK
        int organization_id FK
        uuid agency_profile_id FK
    }

    PROJECT_ASSIGNMENT_WINDOW {
        uuid id PK
        uuid policy_id FK
        varchar code UK
        varchar from_day_spel
        time from_time
        varchar to_day_spel
        time to_time
        boolean to_end_of_day
        int sort_order
    }

    PROJECT_ASSIGNMENT_JOB {
        uuid id PK
        varchar request_mode
        int organization_id FK
        uuid agency_profile_id FK
        varchar actor
        varchar idempotency_key UK
        varchar status
        jsonb assigned_projects
        jsonb additional_projects
        jsonb filter_snapshot
        uuid accepted_policy_id FK
        uuid matched_window_id "REF"
        jsonb accepted_limits
        int resolved_count
        int processed_count
        int succeeded_count
        int failed_count
        bigint version
    }

    PROJECT_ASSIGNMENT_JOB_ITEM {
        uuid id PK
        uuid job_id FK
        uuid agency_cobroker_id FK
        uuid cobroker_profile_id "REF"
        varchar username
        varchar status
        varchar error_code
        varchar error_message
        bigint version
    }

    USER_REGISTERED_SCOPE {
        uuid id UK
        int organization_id
        uuid cobroker_profile_id "REF"
        uuid agency_profile_id "REF"
        varchar username
        varchar registration_type
        varchar scope_type
        varchar scope_id
        varchar status
        varchar row_owner
        varchar source
        timestamptz valid_from
        timestamptz valid_to
    }
```

`user_registered_scope` là projection phạm vi dự án hiệu lực dùng cho query/report; `agency_cobroker.project_metadata` là metadata trên liên kết Sale–đại lý. `ProjectAssignmentScopeStore` chịu trách nhiệm giữ hai nguồn nhất quán.

## 4. Property distribution — đợt, quỹ căn và phân bổ

```mermaid
erDiagram
    SALE_BATCHES ||--o{ SALE_BATCH_UNITS : "batch_id"
    SALE_BATCHES ||--o{ SALE_BATCH_AGENCIES : "batch_id"
    AGENCY_PROFILES ||--o{ SALE_BATCH_AGENCIES : "FK agency_profile_id"
    SALE_BATCH_AGENCIES ||--o{ SALE_BATCH_AGENCY_ROOMS : "batch + agency + project"
    SALE_BATCH_AGENCIES ||--o{ ROOM_LEDGER : "logical batch agency"
    SALE_BATCHES ||--o{ UNIT_ALLOCATION_REQUESTS : "FK batch_id"
    SALE_BATCH_AGENCIES ||--o{ UNIT_ALLOCATION_REQUESTS : "FK batch_agency_id"
    UNIT_ALLOCATION_REQUESTS ||--o{ UNIT_ALLOCATION_REQUEST_ITEMS : "FK request_id"
    SALE_BATCH_UNITS o|--o{ UNIT_ALLOCATION_REQUEST_ITEMS : "FK batch_unit_id"
    UNIT_ALLOCATION_REQUESTS ||--o{ UNIT_ALLOCATION_REQUEST_HISTORY : "request history"
    SALE_BATCH_UNITS ||--o{ SALE_BATCH_UNIT_HISTORY : "unit history"
    UNIT_ALLOCATION_REQUESTS ||--o{ ALLOCATION_PROCESS_QUEUE : "processing queue"

    SALE_BATCHES {
        uuid id PK
        varchar code UK
        varchar name
        varchar status
        jsonb projects
        jsonb baskets
        jsonb rules
        timestamptz start_at
        timestamptz end_at
        bigint version
    }

    SALE_BATCH_UNITS {
        uuid id PK
        uuid batch_id "REF"
        varchar unit_id
        varchar unit_code
        varchar project_id
        varchar basket_id
        varchar category
        varchar allocation_status
        uuid allocated_by_item_id "REF"
        uuid allocated_to_agency_id "REF"
        timestamptz allocated_at
        timestamptz sold_at
        bigint version
    }

    SALE_BATCH_AGENCIES {
        uuid id PK
        uuid batch_id "REF"
        uuid agency_profile_id FK
        varchar participation_status
        int sales_count
        int room_base
        int advance_room
        int bonus_room
        int exchange_room
        jsonb projects_registered
        bigint version
    }

    SALE_BATCH_AGENCY_ROOMS {
        uuid id PK
        uuid batch_id "REF"
        uuid agency_profile_id "REF"
        varchar project_id
        int entitled
        int remaining
        int allocated
        int sold
        bigint version
    }

    ROOM_LEDGER {
        uuid id PK
        uuid batch_id "REF"
        uuid agency_profile_id "REF"
        varchar project_id
        int delta
        varchar reason
        varchar ref_type
        varchar ref_id
    }

    UNIT_ALLOCATION_REQUESTS {
        uuid id PK
        varchar code UK
        uuid batch_id FK
        uuid batch_agency_id FK
        uuid agency_profile_id "REF"
        varchar project_id
        varchar request_type
        varchar status
        varchar access_mode
        varchar approved_by
        timestamptz approved_at
        bigint version
    }

    UNIT_ALLOCATION_REQUEST_ITEMS {
        uuid id PK
        uuid request_id FK
        uuid batch_unit_id FK
        varchar unit_code
        jsonb criteria
        varchar line_status
        numeric score
        varchar return_unit_code
        uuid return_batch_unit_id "REF"
        boolean active
        timestamptz allocated_at
    }

    UNIT_ALLOCATION_REQUEST_HISTORY {
        uuid id PK
        uuid request_id "REF"
        varchar action
        varchar status_before
        varchar status_after
        jsonb snapshot
        varchar performer
    }

    SALE_BATCH_UNIT_HISTORY {
        uuid id PK
        uuid batch_unit_id "REF"
        uuid batch_id "REF"
        varchar action
        varchar status_before
        varchar status_after
        jsonb snapshot
    }

    ALLOCATION_PROCESS_QUEUE {
        uuid id PK
        uuid request_id "REF"
        varchar status
        int retry_count
        timestamptz next_retry_at
        varchar claim_token
    }
```

Chi tiết state machine, constraint và index của domain này xem [Property Distribution Database ERD](./property-distribution/dataflows/08-database-erd.md).

## 5. Agency scoring

```mermaid
erDiagram
    AGENCY_PROFILES ||--o{ AGENCY_SCORE_STATES : "FK agency_profile_id"
    AGENCY_SCORE_STATES ||--o{ AGENCY_SCORE_STATE_HISTORY : "state history"
    AGENCY_PROFILES ||--o{ AGENCY_POINT_LEDGER : "agency score events"
    AGENCY_SCORING_CONFIGS ||--o{ AGENCY_SCORING_CONFIG_HISTORY : "config versions"
    SALE_BATCHES ||--o{ SALE_BATCH_AGENCY_SCORES : "batch snapshot"
    AGENCY_PROFILES ||--o{ SALE_BATCH_AGENCY_SCORES : "agency snapshot"
    SALE_BATCHES ||--|| SALE_BATCH_SCORING_CONFIGS : "config snapshot"

    AGENCY_SCORE_STATES {
        uuid id PK
        int organization_id
        uuid agency_profile_id FK
        numeric capability_score
        numeric compliance_score
        numeric average_score
        varchar rank
        varchar period_key
        bigint version
    }

    AGENCY_SCORE_STATE_HISTORY {
        uuid id PK
        uuid agency_score_state_id "REF"
        uuid agency_profile_id "REF"
        varchar period_key
        jsonb snapshot
        timestamptz captured_at
    }

    AGENCY_POINT_LEDGER {
        uuid id PK
        int organization_id
        uuid agency_profile_id "REF"
        varchar type
        varchar content_code
        numeric delta
        varchar reason
        varchar period_key
        varchar performer
    }

    AGENCY_SCORING_CONFIGS {
        uuid id PK
        int organization_id
        int version UK
        jsonb payload
        varchar status
    }

    AGENCY_SCORING_CONFIG_HISTORY {
        uuid id PK
        int organization_id
        int config_version
        varchar config_group
        varchar entity_key
        text value_before
        text value_after
        varchar reason
        varchar performer
    }

    SALE_BATCH_AGENCY_SCORES {
        uuid id PK
        uuid batch_id "REF"
        uuid agency_profile_id "REF"
        int sale_count
        numeric capability_score
        numeric compliance_score
        numeric final_score
        varchar rank
        jsonb score_detail
    }

    SALE_BATCH_SCORING_CONFIGS {
        uuid id PK
        uuid batch_id UK
        int source_config_version
        jsonb payload
        varchar snapshot_hash
    }
```

## 6. Import, async và outbox

```mermaid
erDiagram
    IMPORT_JOB ||--o{ IMPORT_JOB_ITEM : "FK job_id"

    IMPORT_JOB {
        uuid id PK
        varchar job_type
        varchar status
        varchar file_name
        varchar source_file_key
        varchar result_file_key
        int total_count
        int success_count
        int failure_count
        varchar actor
        bigint version
    }

    IMPORT_JOB_ITEM {
        uuid id PK
        uuid job_id FK
        int row_number
        varchar status
        jsonb raw_data
        jsonb normalized_data
        varchar error_code
        varchar error_message
    }

    ASYNC_JOB {
        uuid id PK
        varchar job_type
        varchar status
        jsonb payload
        int retry_count
        timestamptz next_retry_at
        varchar claim_token
    }

    NOTIFICATION_OUTBOX {
        uuid id PK
        varchar event_type
        varchar aggregate_type
        varchar aggregate_id
        jsonb payload
        varchar status
        int retry_count
        timestamptz next_retry_at
    }

    PROCESS_SYNC_OUTBOX {
        uuid id PK
        varchar aggregate_type
        uuid aggregate_id
        varchar event_type
        jsonb payload
        varchar status
        int retry_count
        varchar dedup_key
    }

    HISTORICAL_OUTBOX {
        uuid id PK
        varchar entity_type
        varchar entity_id
        varchar action
        jsonb payload
        varchar status
        int retry_count
    }

    DISTRIBUTION_REMINDER_LOG {
        uuid id PK
        uuid batch_id "REF"
        uuid agency_profile_id "REF"
        varchar reminder_type
        varchar recipient
        timestamptz sent_at
        varchar status
    }

    SHEDLOCK {
        varchar name PK
        timestamptz lock_until
        timestamptz locked_at
        varchar locked_by
    }
```

Các bảng outbox độc lập, không nên nối FK tới aggregate nghiệp vụ vì phải giữ event kể cả khi trạng thái aggregate thay đổi và để tránh coupling transaction ngoài ý muốn.

## 7. Danh mục bảng đang hoạt động

| Domain | Bảng |
|---|---|
| Agency/Sale | `agency_profiles`, `agency_branch`, `agency_cobroker`, `cobroker_profiles` |
| Applicant/IV | `cobroker_applicant_agencies_submission`, `outbox_cobroker_applicant` |
| Change/Audit | `update_requests`, `update_request_responses`, `audit_logs` |
| Project assignment | `project_assignment_policy`, `project_assignment_policy_agency`, `project_assignment_window`, `project_assignment_job`, `project_assignment_job_item`, `user_registered_scope` |
| Distribution | `sale_batches`, `sale_batch_units`, `sale_batch_agencies`, `sale_batch_agency_rooms`, `room_ledger`, `unit_allocation_requests`, `unit_allocation_request_items`, `allocation_request_history`, `sale_batch_unit_history`, `allocation_process_queue` |
| Scoring | `agency_score_states`, `agency_score_state_history`, `agency_point_ledger`, `agency_scoring_configs`, `agency_scoring_config_history`, `sale_batch_agency_scores`, `sale_batch_scoring_configs` |
| Jobs/Integration | `import_job`, `import_job_item`, `async_job`, `notification_outbox`, `process_sync_outbox`, `historical_outbox`, `distribution_reminder_log`, `shedlock` |

## 8. Bảng đã retired

Các bảng dưới đây có migration tạo trong lịch sử nhưng đã bị drop bởi migration sau:

| Bảng | Trạng thái cuối |
|---|---|
| `identity_verification_manual` | Đã drop; manual decision chuyển vào JSONB của submission |
| `user_projection` | Đã drop; agency/cobroker trở thành nguồn chính |
| `team_projection` | Đã drop sau khi chuyển sang agency source |
| `agency_participation_logs` | Đã drop; thay bằng state history/ledger tương ứng |

## 9. Lưu ý dành cho BDSKD-6590

Hiện schema chỉ thể hiện dự án Sale qua hai projection:

```text
agency_cobroker.project_metadata
              |
              v
user_registered_scope(scope_type=PROJECT, scope_id=OCP2/OCP3)
```

Hai nguồn này chỉ lưu dự án vật lý, chưa có nơi biểu diễn rõ lựa chọn logic `OCP23_GROUP`. Nếu triển khai BDSKD-6590 bằng cách chỉ ghi OCP2 và OCP3, hệ thống sẽ không phân biệt được:

- Sale chọn hai dự án riêng; và
- Sale chọn một combo chiếm một slot.

ERD đề xuất bổ sung:

```mermaid
erDiagram
    AGENCY_COBROKER ||--o{ COBROKER_PROJECT_SELECTION : "logical selections"
    COBROKER_PROJECT_SELECTION ||--o{ USER_REGISTERED_SCOPE : "resolve effective scopes"
    PROJECT_GROUP_MAPPING ||--o{ PROJECT_GROUP_MEMBER : "group definition"
    PROJECT_GROUP_MAPPING ||--o{ COBROKER_PROJECT_SELECTION : "selection group"

    COBROKER_PROJECT_SELECTION {
        uuid id PK
        int organization_id
        uuid agency_cobroker_id FK
        varchar selection_id
        varchar assignment_type
        int mapping_version
        timestamptz created_at
        varchar created_by
    }

    PROJECT_GROUP_MAPPING {
        uuid id PK
        int organization_id
        varchar group_code UK
        varchar display_name
        int quota_cost
        int version
        boolean active
    }

    PROJECT_GROUP_MEMBER {
        uuid id PK
        uuid group_id FK
        varchar project_id
        int sort_order
    }

    USER_REGISTERED_SCOPE {
        uuid id PK
        uuid agency_profile_id
        uuid cobroker_profile_id
        varchar registration_type
        varchar scope_id
        varchar status
    }
```

Đây là schema đề xuất, chưa tồn tại trong migrations hiện tại.

## 10. Nguồn đối chiếu

- Liquibase master: `src/main/resources/db.changelog/db.changelog-master.yaml`
- Migrations: `src/main/resources/db.changelog/ddl/`
- JPA models: `src/main/java/vn/vinhomes/cobroker/core/model/`
- JPA entities: `src/main/java/vn/vinhomes/cobroker/core/entity/`
- ERD distribution chi tiết: `docs/property-distribution/dataflows/08-database-erd.md`
