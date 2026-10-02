# Đối chiếu ConfigMap vhm-cobroker-core: staging và prod

Ngày đối chiếu: 2026-10-02.

Nguồn staging: `market-k8s-manifest-staging/applications/stag/vhm-marketplace-agent-stag/vhm-cobroker-core/configmap.yaml`, branch staging, commit `f2f6e295`.
Nguồn prod: toàn bộ đoạn config người dùng gửi trong cuộc hội thoại. Chưa đọc ConfigMap/Secret thực tế trong cluster.

“Không thấy” chỉ nghĩa là không có trong nguồn đối chiếu; biến có thể được cấp từ Secret, Vault, deployment hoặc nguồn config khác. Chỉ liệt kê biến thiếu hoặc khác; không liệt kê biến giống nhau. Các dòng comment không được coi là cấu hình đang bật.

## Có trên staging, không thấy trong config prod

| Biến | Staging | Prod |
|---|---|---|
| `MANAGEMENT_ENDPOINT_HEALTH_PROBES_ENABLED` | `true` | Không thấy |
| `SECURITY_BASIC_ALLOWED_CIDRS` | `0.0.0.0/0,::1/128` | Không thấy |
| `DELIVERY_EMAIL_FROM_NAME` | `VHM Agent` | Không thấy |
| `DISTRIBUTION_EMAIL_FROM` | `system` | Không thấy |
| `DISTRIBUTION_EMAIL_CC_DEFAULTS` | `""` (rỗng) | Không thấy |
| `DISTRIBUTION_AGENT_WEB_BASE_URL` | `https://stag-agent.vinhomes.vn` | Không thấy |
| `KAFKA_TOPIC_IDENTITY_VERIFICATION_HISTORICAL` | `vap.historical.applicant_identity_verification` | Không thấy |
| `KAFKA_TOPIC_COBROKER_PROFILE_HISTORICAL` | `vap.historical.cobroker_profile` | Không thấy |
| `KAFKA_TOPIC_AGENCY_TAX_CODE_HISTORICAL` | `vap.historical.agency_tax_code` | Không thấy |
| `KAFKA_TOPIC_SALE_ACCOUNT_EVENT` | `vap.event.agency_distribute.sale_account` | Không thấy |
| `KAFKA_TOPIC_AGENCY_SCORE_STATE_HISTORICAL` | `vap.historical.agency_distribute.agency_score_state` | Không thấy |
| `KAFKA_CONSUMER_AGENCY_SCORE_STATE_HISTORICAL_ENABLED` | `true` | Không thấy |
| `KAFKA_CONSUMER_SALE_BATCH_UNIT_HISTORICAL_ENABLED` | `true` | Không thấy |
| `SPRINGDOC_API_DOCS_CONFIG_URL` | `http://market-svc.stg01.vinhomes.internal/vhm-cobroker-core-service/v3/api-docs/swagger-config` | Không thấy |
| `SPRINGDOC_API_DOCS_UI_URL` | `http://market-svc.stg01.vinhomes.internal/vhm-cobroker-core-service/v3/api-docs` | Không thấy |
| `SPRINGDOC_API_DOCS_ENABLED` | `true` | Không thấy |
| `SPRINGDOC_SWAGGER_UI_ENABLED` | `true` | Không thấy |
| `MIGRATION_MULTIPART_THRESHOLD` | `60MB` | Không thấy |
| `KAFKA_TOPIC_PROJECT_SCOPE_MIGRATION_CONCLUDE` | `project-scope-migration-conclude` | Không thấy |
| `PROJECT_ASSIGNMENT_MIGRATION_ENABLED` | `true` | Không thấy |
| `COBROKER_APPLICANT_EKYC_ENABLED` | `true` | Không thấy |
| `COBROKER_APPLICANT_EKYC_AT_MODE_ENABLED` | `false` | Không thấy |
| `COBROKER_APPLICANT_EKYC_DELEGATED_SESSION_TTL` | `10m` | Không thấy |
| `COBROKER_APPLICANT_EKYC_REQUIRE_IDENTITY_DOCUMENT` | `false` | Không thấy |
| `COBROKER_APPLICANT_EKYC_MAX_ATTEMPTS_EKYC` | `999999` | Không thấy |
| `VHM_OCR_EKYC_BASE_URL` | `http://ocr-ekyc-service:8080` | Không thấy |
| `VHM_OCR_EKYC_USERNAME` | `cobroker-core` | Không thấy |
| `VHM_OCR_EKYC_POLL_INTERVAL_MS` | `2000` | Không thấy |
| `VHM_OCR_EKYC_POLL_MAX_ATTEMPTS` | `30` | Không thấy |
| `COBROKER_APPLICANT_EKYC_MAX_ATTEMPTS` | `{"DAILY_ENTRANCE": 999999, "DAILY_FAILED": 5, "TOTAL_FAILED": 5}` | Không thấy |
| `AGENCY_SYNC_ENABLED` | `true` | Không thấy |
| `AGENCY_SYNC_ORGANIZATION_ID` | `1` | Không thấy |
| `AGENCY_SYNC_LEVELS` | `15` | Không thấy |
| `AGENCY_SYNC_PARENT_IDS` | `1066` | Không thấy |
| `AGENCY_SYNC_STATUSES` | `1` | Không thấy |
| `COBROKER_APPLICANT_EKYC_INTEGRITY_CHECK_PROFILE` | `true` | Không thấy |
| `PROJECT_ASSIGNMENT_CATALOG_CAP_CHECK_ENABLED` | `true` | Không thấy |
| `DISTRIBUTION_SCORE_PERFORMANCE_CRON` | `0 30 2 * * TUE,THU` | Không thấy |
| `COBROKER_AGENCY_MAX_ASA_PER_AGENCY` | `20` | Không thấy |
| `COBROKER_APPLICANT_EKYC_MISMATCH_VALIDATION_IGNORE_LIST` | `issue_place` | Không thấy |

## Khác giá trị giữa staging và prod

Khác endpoint, Redis, team ID và công tắc tính năng có thể là chủ đích theo môi trường; không sao chép nguyên giá trị staging sang prod.

| Biến | Staging | Prod |
|---|---|---|
| `ENV` | `stag` | `prod` |
| `SECURITY_BASIC_PATH_PATTERNS` | `/internal/v1/**,/profiler/**` | `/internal/v1/**` |
| `THRIFT_PROFILE_MW_HOST` | `profile-mw-service.vhm-marketplace-stag.svc.cluster.local` | `profile-mw-service.vhm-marketplace-prod.svc.cluster.local` |
| `KAFKA_BOOTSTRAP_SERVER` | `b-1.vhmmarketplacesta.dv80ln.c2.kafka.ap-southeast-1.amazonaws.com:9096,b-2.vhmmarketplacesta.dv80ln.c2.kafka.ap-southeast-1.amazonaws.com:9096` | `b-1.vhmmarketplacepro.z5hpt4.c2.kafka.ap-southeast-1.amazonaws.com:9096,b-2.vhmmarketplacepro.z5hpt4.c2.kafka.ap-southeast-1.amazonaws.com:9096` |
| `KAFKA_LEGACY_BOOTSTRAP_SERVER` | `10.250.237.230:9092` | `10.250.192.235:9092` |
| `SPRINGDOC_API_DOCS_SERVER_URL` | `http://market-svc.stg01.vinhomes.internal/vhm-cobroker-core-service` | `""` (rỗng) |
| `OCR_CLIENT_BASE_URL` | `http://ai-integration-service.vhm-marketplace-data-stag.svc.cluster.local:11510/ocr` | `http://ai-integration-service.vhm-marketplace-data-prod.svc.cluster.local:11510/ocr` |
| `FILE_CLIENT_BASE_URL` | `http://file-management-service.vhm-marketplace-stag.svc.cluster.local:10108` | `http://file-management-service.vhm-marketplace-prod.svc.cluster.local:10108` |
| `MESSAGE_DELIVERY_BASE_URL` | `http://message-delivery-service.vhm-marketplace-stag.svc.cluster.local:11144/internal` | `http://message-delivery-service.vhm-marketplace-prod.svc.cluster.local:11144/internal` |
| `THRIFT_AUTH_SERVICE_HOST` | `auth-admin-service.vhm-marketplace-stag.svc.cluster.local` | `auth-admin-service.vhm-marketplace-prod.svc.cluster.local` |
| `COBROKER_IAM_BASE_URL` | `http://vhm-cobroker-iam-service.vhm-marketplace-agent-stag.svc.cluster.local:8080` | `http://vhm-cobroker-iam-service.vhm-marketplace-agent-prod.svc.cluster.local:8080` |
| `COBROKER_IAM_CONNECT_TIMEOUT` | `5s` | `10s` |
| `COBROKER_IAM_READ_TIMEOUT` | `10s` | `20s` |
| `AGENCY_PROFILE_TEAM_PARENT_ID` | `1066` | `377` |
| `MARKET_CORE_BASE_URL` | `http://vhm-market-core-service-service.vhm-marketplace-backend-stag.svc.cluster.local:8080` | `http://vhm-market-core-service-service.vhm-marketplace-backend-prod.svc.cluster.local:8080` |
| `REDISSON_CONFIG_USE_CLUSTER_SERVERS` | `true` | `false` |
| `REDISSON_CONFIG_ADDRESSES` | `clustercfg.vhm-marketplace-stag-redis-cluster.zgwdit.apse1.cache.amazonaws.com:6379` | `master.vhm-marketplace-prod-redis-node-based.uwcmeg.apse1.cache.amazonaws.com:6379` |
| `REDISSON_CONFIG_PREFIX` | `miniapp_vhm_market_api` | `vhm_cobroker_core` |
| `PROPERTY_DISTRIBUTE_BASE_URL` | `http://property-distribute-service.vhm-marketplace-backend-stag.svc.cluster.local:8080` | `http://property-distribute-service.vhm-marketplace-backend-prod.svc.cluster.local:8080` |
| `HOUSING_DICTIONARY_BASE_URL` | `http://vhm-housing-dictionary-core.vhm-marketplace-housing-stag.svc.cluster.local:8080` | `http://vhm-housing-dictionary-core.vhm-marketplace-housing-prod.svc.cluster.local:8080` |
| `KAFKA_DISTRIBUTION_CONSUMER_PROPERTY_SOLD_ENABLED` | `true` | `false` |
| `KAFKA_DISTRIBUTION_CONSUMER_DISTRIBUTE_TURN_ENABLED` | `true` | `false` |
| `KAFKA_DISTRIBUTION_CONSUMER_MARKET_PROPERTY_HISTORICAL_ENABLED` | `true` | `false` |
| `KAFKA_CONSUMER_PROJECT_SCOPE_MIGRATION_CONCLUDE_ENABLED` | `true` | `false` |
| `DISTRIBUTION_EMAIL_ENABLED` | `true` | `false` |
| `OCR_MODEL_ACTIVE` | `in_house` | `vinbigdata` |
| `FEATURE_PROPERTY_DISTRIBUTION_ENABLED` | `true` | `false` |

## Có trên prod, không khai báo trực tiếp trong ConfigMap staging

APM và JAVA_TOOL_OPTS trên staging đang nằm trong comment. Database/biến khác có thể được cấp ở nguồn riêng.

| Biến | Staging | Prod |
|---|---|---|
| `POSTGRESQL_URL` | Không khai báo trực tiếp | `jdbc:postgresql://vhm-marketplace-prod-aurora-postgres17.cluster-c5oka6ya2krv.ap-southeast-1.rds.amazonaws.com:5432/vhmmarket_db?currentSchema=cobroker_db&serverTimezone=Asia/Ho_Chi_Minh&useLegacyDatetimeCode=false` |
| `POSTGRESQL_USER` | Không khai báo trực tiếp | `cobroker_user` |
| `DB_DEFAULT_SCHEMA` | Không khai báo trực tiếp | `cobroker_db` |
| `ELASTIC_APM_SERVICE_NAME` | Không khai báo trực tiếp | `vhm-cobroker-core` |
| `ELASTIC_APM_ENVIRONMENT` | Không khai báo trực tiếp | `prod` |
| `ELASTIC_APM_SERVER_URL` | Không khai báo trực tiếp | `https://uat-apm-server.vinhomes.vn` |
| `ELASTIC_APM_APPLICATION_PACKAGES` | Không khai báo trực tiếp | `vn.vinhomes,qtvhbds` |
| `ELASTIC_APM_LOG_CORRELATION_ENABLED` | Không khai báo trực tiếp | `true` |
| `ELASTIC_APM_CAPTURE_BODY` | Không khai báo trực tiếp | `off` |
| `ELASTIC_APM_TRANSACTION_SAMPLE_RATE` | Không khai báo trực tiếp | `1` |
| `ELASTIC_APM_IGNORE_URLS` | Không khai báo trực tiếp | `/actuator/**,/v3/api-docs/**,/swagger-ui/**,/ping/**` |
| `JAVA_TOOL_OPTS` | Không khai báo trực tiếp | `-javaagent:/usr/app/elastic-apm-agent.jar` |
| `KAFKA_PROFILE_CONSUMER_USER_ENABLED` | Không khai báo trực tiếp | `true` |
| `KAFKA_PROFILE_CONSUMER_TEAM_ENABLED` | Không khai báo trực tiếp | `true` |
| `KAFKA_PROFILE_TOPIC_USER` | Không khai báo trực tiếp | `profile_user` |
| `KAFKA_PROFILE_TOPIC_TEAM` | Không khai báo trực tiếp | `profile_team` |
| `COBROKER_PROJECT_SCOPE_OWNERSHIP_ENABLED` | Không khai báo trực tiếp | `false` |
| `COBROKER_PROJECT_SCOPE_MIGRATION_ENABLED` | Không khai báo trực tiếp | `true` |

## Khác nguồn cấp hoặc chưa có giá trị để so sánh

| Biến | Staging | Prod |
|---|---|---|
| `FILE_PRIVATE_CLIENT_USERNAME` | `cobroker-svc` | Vault; chưa có giá trị |
| `MESSAGE_DELIVERY_USERNAME` | `vhm-campaign` | Vault; chưa có giá trị |
| `VINBIGDATA_CLIENT_APP_ID` | `""` (rỗng) | Vault; chưa có giá trị |
| `KAFKA_SASL_JAAS_CONFIG` | Comment chỉ định cấp qua Vault | Không thấy trong đoạn gửi; cần kiểm tra Secret/Vault |
| `VHM_OCR_EKYC_PASSWORD` | Không khai báo trực tiếp | Không thấy; cần kiểm tra Secret/Vault |

## Thay đổi đang chờ merge và ưu tiên xử lý

| Biến | Staging hiện tại | Staging sau MR manifest !6532 | Prod đã gửi |
|---|---|---|---|
| `KAFKA_CONSUMER_AGENCY_PROFILE_HISTORICAL_ENABLED` | Không thấy | `true` | Không thấy |

- Cần cấp `KAFKA_CONSUMER_AGENCY_PROFILE_HISTORICAL_ENABLED` trước khi deploy code MR core !869 vì code đã bỏ default của biến này. MR manifest !6532 chỉ bổ sung cho staging.
- `VHM_OCR_EKYC_BASE_URL` và nhóm `AGENCY_SYNC_*` được tham chiếu không có fallback trong code. Xác nhận nguồn cấp thực tế trên prod; không copy host/team ID staging.
- Bốn topic identity-verification/cobroker-profile/agency-tax-code/sale-account có fallback đúng trong code. Có thể khai báo rõ trong prod để đồng bộ cách quản lý với staging.
- `KAFKA_TOPIC_AGENCY_PROFILE_HISTORICAL` không khai báo trên cả hai ConfigMap; code có fallback `vap.historical.agency_profile`.
- `FEATURE_PROPERTY_DISTRIBUTION_ENABLED=false` trên prod là chủ đích theo comment; không tự bật các consumer distribution theo staging.
- `MARKET_SCHEDULER_ENABLED=true`, các cron historical outbox và bốn consumer identity-verification/cobroker-profile/agency-tax-code/sale-account giống nhau. Không thấy thiếu config bật scheduler trong đoạn prod đã gửi.
- `KAFKA_AUTO_OFFSET_RESET=latest` giống nhau. Consumer group mới chưa có offset sẽ bắt đầu ở cuối topic, nên không tự xử lý event cũ.
- Comment historical outbox ghi Kafka legacy, nhưng code hiện tại publish qua Kafka MSK mới.

Chưa sửa hay áp dụng config production. Báo cáo so sánh ConfigMap, không phải toàn bộ cấu hình hiệu lực của ứng dụng.
