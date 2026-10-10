# NOXH — Thông tin Kafka cho consumer Data

## 1. Topic và cách nhận dữ liệu

- **Topic mặc định:** `dossier.data_changed.v1` (có thể override theo môi trường).
- **Value:** JSON UTF-8; chung một topic cho 4 bảng, phân loại bằng `sourceTable`.
- **Kafka key:** `aggregateId` — ID bản ghi dạng chuỗi; khóa ghép là JSON string.
- **NOXH:** lọc hồ sơ `data.productCode = "SOCIAL_HOUSING"`, join các bảng con theo dossier ID.
- Broker, quyền truy cập và thời điểm mở luồng cần xác nhận khi bàn giao môi trường; **chưa xác nhận traffic staging/prod**.

| sourceTable | key | Field join hồ sơ trong data |
| --- | --- | --- |
| dossier | `{ "id": "UUID" }` | `id` |
| dossier_note | `{ "id": "UUID" }` | `dossierId` |
| dossier_status_history | `{ "id": 123 }` (BIGINT) | `dossierId` |
| dossier_stage_reviewer | `{ "dossierId": "UUID", "stageCode": "SALES" }` | `id.dossierId`; stage tại `id.stageCode` |

## 2. Payload

| Field | Kiểu / ý nghĩa |
| --- | --- |
| eventId | UUID string, dùng dedup khi retry |
| schemaVersion | integer, hiện tại `1` |
| sourceSystem | string, `vhm-dossier-core` |
| sourceTable | string, tên một trong bốn bảng trên |
| operation | `INSERT` / `UPDATE` / `DELETE` |
| aggregateId | string, ID bản ghi; khóa ghép là JSON string |
| key | object chứa khóa chính |
| occurredAt | string ISO-8601, thời điểm tạo event |
| data | Snapshot entity khi INSERT/UPDATE; `null` khi DELETE |

Field trong `data` dùng **camelCase**; tên cột DB dùng **snake_case**. JSON bên trong `formData` giữ nguyên cấu trúc nguồn. Field có thể null/thiếu dữ liệu theo hồ sơ. Timestamp dùng ISO-8601.

Ví dụ UPDATE reviewer (dữ liệu minh họa):

```json
{
  "eventId": "00000000-0000-4000-8000-000000000002",
  "schemaVersion": 1,
  "sourceSystem": "vhm-dossier-core",
  "sourceTable": "dossier_stage_reviewer",
  "operation": "UPDATE",
  "aggregateId": "{\"dossierId\":\"00000000-0000-4000-8000-000000000001\",\"stageCode\":\"SALES\"}",
  "key": {"dossierId": "00000000-0000-4000-8000-000000000001", "stageCode": "SALES"},
  "occurredAt": "2026-10-10T03:00:00Z",
  "data": {
    "id": {"dossierId": "00000000-0000-4000-8000-000000000001", "stageCode": "SALES"},
    "reviewerId": "reviewer-demo", "reviewerName": "Reviewer Demo",
    "reviewerEmail": null, "reviewerRole": null,
    "claimedAt": null, "assignedBy": "assigner-demo",
    "assignedAt": "2026-10-10T02:00:00Z", "reviewedAt": "2026-10-10T03:00:00Z",
    "decision": "APPROVED", "comment": null
  }
}
```

## 3. Mapping 58 metric

Database/schema nguồn: `vhmmarket_db.dossier_db`. Field Kafka trong bảng dưới tính từ **`data`**. Ghi chú “cần chốt” nghĩa là chưa thể coi mapping nghiệp vụ đã được xác nhận.

### Hồ sơ

| Metric | Bảng | Cột / JSON path DB | Field trong data | Ghi chú |
| --- | --- | --- | --- | --- |
| NOXH_profile_id | dossier | id | id; mã BO: formData.code | UUID hồ sơ; mã BO là formData.code. |
| NOXH_customer_name | dossier | form_data #>> '{applicant,fullName}' | formData.applicant.fullName | — |
| NOXH_project_name | dossier | form_data #>> '{projectRegistration,projectId}' | formData.projectRegistration.projectId | Chỉ có ID; tên lấy từ Market. |
| NOXH_agency_name | dossier | form_data #>> '{projectRegistration,agencyId}' | formData.projectRegistration.agencyId | Chỉ có ID; tên lấy từ AgentProfile. |
| NOXH_proposed_unit_info | dossier | form_data #>> '{projectRegistration,proposedUnitCode}' | formData.projectRegistration.proposedUnitCode | Có mã căn; thông tin khác cần master căn. |
| NOXH_assigned_unit_code | dossier | form_data #>> '{projectRegistration,assignedUnitCode}' | formData.projectRegistration.assignedUnitCode | — |
| NOXH_marital_status | dossier | form_data #>> '{applicant,maritalStatus}' | formData.applicant.maritalStatus | — |
| NOXH_housing_status | dossier | form_data #>> '{applicant,housingStatus}' | formData.applicant.housingStatus | — |
| NOXH_customer_eligible_group | dossier | form_data ->> 'subjectGroup' | formData.subjectGroup | — |
| NOXH_spouse_eligible_group | dossier | form_data #>> '{spouse,subjectCoApplicant}' | formData.spouse.subjectCoApplicant | Cần TUHS xác nhận nghĩa nhóm đối tượng. |
| NOXH_created_date | dossier | created_at | createdAt | — |
| NOXH_updated_date | dossier | updated_at | updatedAt (BO report: lastEventAt) | Cần chốt updatedAt hay lastEventAt. |
| NOXH_approver_sales_dept | dossier_stage_reviewer | reviewer_id; reviewer_name; decision; reviewed_at | reviewerId / reviewerName / decision / reviewedAt; id.stageCode=SALES | stage=SALES; người duyệt phải xét decision/reviewedAt. |
| NOXH_approver_procedure_dept | dossier_stage_reviewer | reviewer_id; reviewer_name; decision; reviewed_at | reviewerId / reviewerName / decision / reviewedAt; id.stageCode=PROCEDURE | stage=PROCEDURE; người duyệt phải xét decision/reviewedAt. |
| NOXH_profile_status | dossier | status; current_stage_code | status + currentStageCode | Label trạng thái map theo BO. |
| NOXH_created_source | dossier | source | source | — |
| NOXH_profile_progress | dossier | form_data -> 'documents'; current_stage_code | formData.documents + currentStageCode; phải tính theo nghiệp vụ | Cần chốt tiến độ pipeline hay % giấy tờ. |
| NOXH_sale_owner | dossier | owner | owner | Username; tên lấy từ master người dùng. |
| NOXH_reject_supplement_reason | dossier_status_history | reason; at; to_status | history: reason / at / toStatus; reviewer: comment / decision / id.stageCode | Bổ sung: history mới nhất. Reject BO: comment reviewer REJECTED. |
| NOXH_first_deadline_date | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_first_overdue_days | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_second_deadline_date | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_second_overdue_days | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_submitter_name | dossier | created_by; dossier_status_history.actor_id | dossier: createdBy; history: actorId (chưa chốt người nộp) | Cần chốt khách hàng hay actor SUBMIT; không mặc định createdBy. |
| NOXH_submitted_date | dossier | submitted_at | dossier: submittedAt; note: occurredAt / kind nếu bản cứng | Cần chốt online hay nộp bản cứng. |
| NOXH_supplement_request_date | dossier | entered_stage_at; dossier_status_history.at | dossier: enteredStageAt; history: at khi toStatus=ADD_INFO_REQUESTED | Phân biệt chu kỳ hiện tại và lịch sử. |
| NOXH_assignment_date | dossier_stage_reviewer | assigned_at; claimed_at | assignedAt / claimedAt; id.stageCode=SALES | assignedAt=giao, claimedAt=tự nhận; không có toàn bộ lịch sử. |
| NOXH_customer_dob | dossier | form_data #>> '{applicant,dateOfBirth}' | formData.applicant.dateOfBirth | — |
| NOXH_customer_gender | dossier | form_data #>> '{applicant,gender}' | formData.applicant.gender | — |
| NOXH_customer_id_number | dossier | form_data #>> '{applicant,idNumber}' | formData.applicant.idNumber | — |
| NOXH_id_issue_date | dossier | form_data #>> '{applicant,idIssueDate}' | formData.applicant.idIssueDate | — |
| NOXH_id_issue_place | dossier | form_data #>> '{applicant,idIssuePlace}' | formData.applicant.idIssuePlace | — |
| NOXH_customer_phone | dossier | form_data #>> '{applicant,phone}' | formData.applicant.phone | — |
| NOXH_customer_email | dossier | form_data #>> '{applicant,email}' | formData.applicant.email | — |
| NOXH_permanent_address | dossier | form_data #>> '{applicant,permanentAddress}' | formData.applicant.permanentAddress | Theo khảo sát 09/10; không thay bằng contactAddress. |
| NOXH_contact_address | dossier | form_data #>> '{applicant,contactAddress}' | formData.applicant.contactAddress | — |

### Báo cáo

| Metric | Bảng | Cột / JSON path DB | Field trong data | Ghi chú |
| --- | --- | --- | --- | --- |
| NOXH_report_proposed_unit_info | dossier | form_data #>> '{projectRegistration,proposedUnitCode}' | formData.projectRegistration.proposedUnitCode | Có mã căn; thông tin khác cần master căn. |
| NOXH_report_assigned_unit_code | dossier | form_data #>> '{projectRegistration,assignedUnitCode}' | formData.projectRegistration.assignedUnitCode | — |
| NOXH_report_agency_name | dossier | form_data #>> '{projectRegistration,agencyId}' | formData.projectRegistration.agencyId | Chỉ có ID; tên lấy từ AgentProfile. |
| NOXH_report_customer_name | dossier | form_data #>> '{applicant,fullName}' | formData.applicant.fullName | — |
| NOXH_report_created_date | dossier | created_at | createdAt | — |
| NOXH_report_updated_date | dossier | last_event_at | lastEventAt | BO report dùng lastEventAt. |
| NOXH_report_sxd_approved_date | dossier_stage_reviewer | reviewed_at | reviewedAt; id.stageCode=SXD; cần chốt decision | stage=SXD; cần chốt lọc decision=APPROVED. |
| NOXH_report_arrival_date | dossier | submitted_at; dossier_note.occurred_at | dossier: submittedAt; note: occurredAt / kind (chưa chốt) | Cần chốt nộp online/gửi bản cứng/nhận bản cứng. |
| NOXH_report_due_date_1 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_report_overdue_days_1 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_report_due_date_2 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_report_overdue_days_2 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Derived từ pipeline/rule/lịch nghỉ; cần chốt với TUHS. |
| NOXH_report_status | dossier | status; current_stage_code | status + currentStageCode | Label trạng thái map theo BO. |


## 4. Quy tắc xử lý

- Dedup bằng `eventId`; upsert theo `(sourceTable, key)` cho INSERT/UPDATE. Event con có thể đến trước event hồ sơ.
- DELETE: xóa theo key, `data=null` (Kafka value vẫn là JSON). Note soft-delete là UPDATE có `data.deletedAt`; loại khỏi view active.
- Không đảm bảo thứ tự giữa các event; `occurredAt` không phải thứ tự commit. Chốt cách đối soát latest state và snapshot ban đầu trước khi chạy chính thức; luồng chưa tự backfill dữ liệu cũ.
- Reviewer là trạng thái hiện tại theo hồ sơ × stage, không phải lịch sử mọi lần phân công. Chọn/aggregate bảng con trước khi join để tránh nhân số hồ sơ.
- SLA/overdue phải tính lại theo thời gian; tên dự án/đại lý/người dùng và master template/checklist cần nguồn bổ sung như ghi chú mapping.

Cập nhật 10/10/2026; mapping dựa trên `NOXH_Datamart_Mapping_20261009.md`, payload đối chiếu source hiện tại của dossier-core.
