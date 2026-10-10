# NOXH — Thông tin Kafka cho consumer Data

## 1. Topic và cách nhận dữ liệu

- **Topic:** `vap.historical.dossier`.
- **Cấu hình producer/GitOps:** `OUTBOX_DATA_CHANGE_TOPIC: "vap.historical.dossier"`
  (property `outbox.kafka.data-change-topic`).
- **Value:** JSON UTF-8; chung một topic cho 4 bảng, phân loại bằng `aggregateType`.
- **Kafka key:** `aggregateId` — ID bản ghi dạng chuỗi; khóa ghép là JSON string.
- **NOXH:** lọc hồ sơ `payload.productCode = "SOCIAL_HOUSING"`, join các bảng con theo dossier ID.

| aggregateType | aggregateId (string) | Field join hồ sơ trong payload |
| --- | --- | --- |
| dossier | `UUID` | `id` |
| dossier_note | `UUID` | `dossierId` |
| dossier_status_history | `"123"` (ID BIGINT dạng chuỗi) | `dossierId` |
| dossier_stage_reviewer | `{ "dossierId": "UUID", "stageCode": "SALES" }` | `id.dossierId`; stage tại `id.stageCode` |

## 2. Payload

| Field | Kiểu / ý nghĩa |
| --- | --- |
| id | integer (BIGINT), ID outbox; dùng dedup khi retry |
| aggregateType | string, tên bảng |
| aggregateId | string, ID bản ghi; khóa ghép là JSON string |
| eventType | INSERT / UPDATE / DELETE |
| eventVersion | string, v1 |
| createdAt | string ISO-8601, thời điểm tạo event |
| traceId | string hoặc null |
| publishedAt | null tại thời điểm gửi |
| payload | Dữ liệu entity; DELETE chứa snapshot trước khi xóa |
| payload.sourceSystem | string, vhm-dossier-core |

Ví dụ UPDATE reviewer (dữ liệu minh họa):

```json
{
  "id": 123,
  "aggregateId": "{\"dossierId\":\"00000000-0000-4000-8000-000000000001\",\"stageCode\":\"SALES\"}",
  "aggregateType": "dossier_stage_reviewer",
  "eventType": "UPDATE",
  "eventVersion": "v1",
  "payload": {
    "id": {
      "dossierId": "00000000-0000-4000-8000-000000000001",
      "stageCode": "SALES"
    },
    "reviewerId": "reviewer-demo",
    "reviewerName": "Reviewer Demo",
    "reviewerEmail": null,
    "reviewerRole": null,
    "claimedAt": null,
    "assignedBy": "assigner-demo",
    "assignedAt": "2026-10-10T02:00:00Z",
    "reviewedAt": "2026-10-10T03:00:00Z",
    "decision": "APPROVED",
    "comment": null,
    "sourceSystem": "vhm-dossier-core"
  },
  "traceId": null,
  "createdAt": "2026-10-10T03:00:00Z",
  "publishedAt": null
}
```

## 3. Mapping metric

Database/schema nguồn: `vhmmarket_db.dossier_db`. Field Kafka trong bảng dưới tính từ **`payload`**.

### Hồ sơ

| Metric | Bảng | Cột / JSON path DB | Field trong payload | Ghi chú |
| --- | --- | --- | --- | --- |
| NOXH_profile_id | dossier | id | id | UUID hồ sơ. |
| NOXH_customer_name | dossier | form_data #>> '{applicant,fullName}' | formData.applicant.fullName | — |
| NOXH_project_name | dossier | form_data #>> '{projectRegistration,projectId}' | formData.projectRegistration.projectId | Chỉ có ID; tên lấy từ Market. |
| NOXH_agency_name | dossier | form_data #>> '{projectRegistration,agencyId}' | formData.projectRegistration.agencyId | Chỉ có ID; tên lấy từ AgentProfile. |
| NOXH_proposed_unit_info | dossier | form_data #>> '{projectRegistration,proposedUnitCode}' | formData.projectRegistration.proposedUnitCode | Mã căn; thuộc tính khác lấy từ master căn. |
| NOXH_assigned_unit_code | dossier | form_data #>> '{projectRegistration,assignedUnitCode}' | formData.projectRegistration.assignedUnitCode | — |
| NOXH_marital_status | dossier | form_data #>> '{applicant,maritalStatus}' | formData.applicant.maritalStatus | — |
| NOXH_housing_status | dossier | form_data #>> '{applicant,housingStatus}' | formData.applicant.housingStatus | — |
| NOXH_customer_eligible_group | dossier | form_data ->> 'subjectGroup' | formData.subjectGroup | — |
| NOXH_spouse_eligible_group | dossier | form_data #>> '{spouse,subjectCoApplicant}' | formData.spouse.subjectCoApplicant | Giá trị subjectCoApplicant trong thông tin spouse. |
| NOXH_created_date | dossier | created_at | createdAt | — |
| NOXH_updated_date | dossier | updated_at | updatedAt | updatedAt: cập nhật bản ghi; lastEventAt: sự kiện gần nhất. |
| NOXH_approver_sales_dept | dossier_stage_reviewer | reviewer_id; reviewer_name; decision; reviewed_at | reviewerId / reviewerName / decision / reviewedAt; id.stageCode=SALES | stage=SALES; người duyệt phải xét decision/reviewedAt. |
| NOXH_approver_procedure_dept | dossier_stage_reviewer | reviewer_id; reviewer_name; decision; reviewed_at | reviewerId / reviewerName / decision / reviewedAt; id.stageCode=PROCEDURE | stage=PROCEDURE; người duyệt phải xét decision/reviewedAt. |
| NOXH_profile_status | dossier | status; current_stage_code | status + currentStageCode | — |
| NOXH_created_source | dossier | source | source | — |
| NOXH_profile_progress | dossier | form_data -> 'documents'; current_stage_code | formData.documents + currentStageCode; phải tính theo nghiệp vụ | currentStageCode: trạng thái pipeline; documents: đầu vào tính % giấy tờ. |
| NOXH_sale_owner | dossier | owner | owner | Username; tên lấy từ master người dùng. |
| NOXH_reject_supplement_reason | dossier_status_history | reason; at; to_status | history: reason / at / toStatus; reviewer: comment / decision / id.stageCode | Bổ sung: history mới nhất. Từ chối: comment reviewer có decision=REJECTED. |
| NOXH_first_deadline_date | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_first_overdue_days | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_second_deadline_date | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_second_overdue_days | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_submitter_name | dossier | created_by; dossier_status_history.actor_id | dossier: createdBy; history: actorId | createdBy: người tạo; actorId: người thực hiện chuyển trạng thái. |
| NOXH_submitted_date | dossier | submitted_at | dossier: submittedAt; note: occurredAt / kind nếu bản cứng | submittedAt: nộp online; note.kind phân biệt các mốc bản cứng. |
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

| Metric | Bảng | Cột / JSON path DB | Field trong payload | Ghi chú |
| --- | --- | --- | --- | --- |
| NOXH_report_proposed_unit_info | dossier | form_data #>> '{projectRegistration,proposedUnitCode}' | formData.projectRegistration.proposedUnitCode | Mã căn; thuộc tính khác lấy từ master căn. |
| NOXH_report_assigned_unit_code | dossier | form_data #>> '{projectRegistration,assignedUnitCode}' | formData.projectRegistration.assignedUnitCode | — |
| NOXH_report_agency_name | dossier | form_data #>> '{projectRegistration,agencyId}' | formData.projectRegistration.agencyId | Chỉ có ID; tên lấy từ AgentProfile. |
| NOXH_report_customer_name | dossier | form_data #>> '{applicant,fullName}' | formData.applicant.fullName | — |
| NOXH_report_created_date | dossier | created_at | createdAt | — |
| NOXH_report_updated_date | dossier | last_event_at | lastEventAt | — |
| NOXH_report_sxd_approved_date | dossier_stage_reviewer | reviewed_at | reviewedAt; id.stageCode=SXD; decision | stage=SXD; reviewedAt là thời điểm xử lý, decision là kết quả. |
| NOXH_report_arrival_date | dossier | submitted_at; dossier_note.occurred_at | dossier: submittedAt; note: occurredAt / kind | submittedAt: nộp online; note HARDCOPY_SUBMISSION: gửi bản cứng; HARDCOPY_RECEIPT: nhận bản cứng. |
| NOXH_report_due_date_1 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_report_overdue_days_1 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_report_due_date_2 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_report_overdue_days_2 | dossier | entered_stage_at; pipeline_code; pipeline_version; current_stage_code | enteredStageAt, pipelineCode, pipelineVersion, currentStageCode + cấu hình SLA | Tính từ pipeline, rule SLA và lịch ngày nghỉ. |
| NOXH_report_status | dossier | status; current_stage_code | status + currentStageCode | — |
