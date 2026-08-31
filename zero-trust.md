# Technical Design Document: Edge Gateway & Authorization Platform

| Thuộc tính | Giá trị |
|---|---|
| Trạng thái | Proposed |
| Cấp độ | Architect / Principal Engineer |
| Phiên bản | 1.0 |
| Ngày | 2026-08-31 |
| Phạm vi | Thay thế các BFF lặp AuthN/AuthZ bằng Edge Gateway, Authorization Platform/PDP, policy-as-code, Istio/mTLS và business authorization tại domain service |
| Chủ sở hữu đề xuất | Security Platform + Application Platform |
| Đối tượng phê duyệt | Architecture Council, Security, SRE, đại diện các domain Agent/Market/Broker |

## 1. Executive summary

Hệ thống hiện có nhiều BFF như `agent-api`, `market-api`, `core-broker-api`. Các BFF này lặp lại authentication, authorization, chuyển đổi claim, routing và một phần logic nghiệp vụ. Mô hình đó làm tăng chi phí thay đổi, tạo sai lệch policy, khó audit và buộc mỗi domain mới phải sinh thêm một lớp API trung gian.

Thiết kế này đưa các năng lực dùng chung thành platform capability:

- **Edge Gateway** xác thực token, bảo vệ perimeter, định tuyến, rate limit và áp dụng coarse-grained policy.
- **Authorization Platform** quản lý policy-as-code, biên dịch/kiểm thử/phân phối policy và thu thập decision log.
- **PDP cục bộ hoặc gần workload** đánh giá policy trên data path; không phụ thuộc bắt buộc vào một PDP từ xa cho mọi request.
- **Istio + mTLS** cung cấp workload identity, mã hóa east-west traffic và policy mạng/service-to-service.
- **Domain service** vẫn sở hữu business authorization gắn với dữ liệu và invariant của domain.
- Mọi quyết định tuân theo mô hình chuẩn `Actor + Action + Resource + Context -> Decision`.

Mục tiêu không phải tạo một “gateway thông minh” mới. Gateway và platform chỉ xử lý concern dùng chung; quyền nghiệp vụ vẫn nằm gần domain model. Migration diễn ra từng endpoint, dùng shadow evaluation, so sánh quyết định và rollback độc lập.

## 2. Bối cảnh và problem statement

### 2.1 Hiện trạng giả định

```text
Clients
  ├──> agent-api --------> Agent services
  ├──> market-api -------> Market services
  └──> core-broker-api --> Broker services

Mỗi BFF thường tự thực hiện:
- xác thực/parse JWT
- ánh xạ role/permission
- kiểm tra access theo tenant/đơn vị
- routing và orchestration
- audit/log theo cách riêng
- đôi khi chứa business rules
```

### 2.2 Vấn đề

1. **Code và policy bị nhân bản:** cùng một rule được cài lại ở nhiều BFF, framework và phiên bản khác nhau.
2. **Không có semantic chuẩn:** `role`, `permission`, `scope`, resource ID và tenant context được hiểu khác nhau giữa các hệ thống.
3. **Policy drift:** thay đổi quyền không được triển khai đồng bộ; khó biết policy nào đang có hiệu lực ở đâu.
4. **Boundary sai:** BFF vừa làm security, routing, orchestration, vừa làm business authorization; ownership không rõ.
5. **Khó audit:** không có decision ID và decision log thống nhất để trả lời ai đã truy cập gì, với policy/version nào.
6. **Tăng latency và blast radius:** các tầng trung gian dư thừa, chuỗi gọi dài, scaling gắn với từng nhóm API.
7. **East-west trust yếu:** service có thể tin header do upstream truyền vào mà không xác minh workload identity hoặc provenance.
8. **Khó mở rộng tổ chức:** mỗi domain hoặc channel mới kéo theo một BFF mới và một bản sao security logic.

### 2.3 Câu hỏi thiết kế

Làm thế nào để chuẩn hóa AuthN/AuthZ cho nhiều domain mà không biến Edge Gateway hoặc PDP trung tâm thành bottleneck, single point of failure hay nơi chứa toàn bộ business logic?

## 3. Mục tiêu và tiêu chí thành công

### 3.1 Mục tiêu

- Một mô hình authorization thống nhất cho user, machine và workload.
- Centralized policy lifecycle, distributed enforcement/evaluation.
- Loại bỏ phần AuthN/AuthZ dùng chung khỏi các BFF cũ.
- Enforce zero-trust cho north-south và east-west traffic.
- Domain team tự chủ business authorization trong guardrail chung.
- Policy thay đổi có review, test, provenance, version, rollback và audit.
- Migration không big-bang, không làm gián đoạn client hiện hữu.

### 3.2 Key results đề xuất

- 100% external request đi qua Edge Gateway được xác thực và gắn request/trace identity.
- 100% service-to-service traffic thuộc phạm vi migration dùng mTLS STRICT.
- >= 95% shared authorization rule được loại khỏi BFF cũ sau migration.
- 100% quyết định authorization quan trọng có `decision_id`, policy version và actor/resource metadata phù hợp chính sách dữ liệu.
- Không có policy production được phát hành nếu chưa qua test và approval bắt buộc.
- Giảm ít nhất 30% lead time khi thêm API/domain mới so với baseline.

## 4. Non-goals

- Không thay thế Identity Provider/IAM hiện có.
- Không thiết kế lại toàn bộ domain model hoặc API contract trong một lần.
- Không đưa business invariant vào Edge Gateway hoặc service mesh.
- Không dùng network location như tín hiệu tin cậy duy nhất.
- Không biến role thành mô hình duy nhất; RBAC chỉ là một input của ABAC/ReBAC khi cần.
- Không xây một workflow/entitlement UI đầy đủ trong phase đầu.
- Không bảo đảm exactly-once cho decision log; ưu tiên không ảnh hưởng request path và có khả năng đối soát.
- Không loại bỏ BFF nào còn thực hiện composition/channel-specific transformation có giá trị; chỉ loại bỏ BFF thuần proxy/auth wrapper.

## 5. Design principles

1. **Never trust, always verify:** user token, workload identity, transport và context đều được xác minh tại boundary phù hợp.
2. **Centralize management, distribute enforcement:** policy lifecycle tập trung; PEP/PDP được đặt gần traffic để giảm latency và dependency.
3. **Authentication không đồng nghĩa authorization:** token hợp lệ không tự động có quyền trên resource.
4. **Policy chung ở platform, invariant ở domain:** platform quyết định quyền coarse/standard; service quyết định quyền phụ thuộc state và dữ liệu nghiệp vụ.
5. **Default deny, least privilege:** thiếu identity, context hoặc policy hợp lệ mặc định từ chối.
6. **No trusted identity headers from clients:** mọi header identity bên ngoài bị xóa; chỉ gateway/mesh được phép tạo metadata nội bộ đã ký hoặc được ràng buộc với mTLS identity.
7. **Explicit action/resource vocabulary:** tránh kiểm tra URL hoặc role rải rác trong code.
8. **Policy is code:** versioned, reviewed, tested, signed, promoted và rollback được.
9. **Fail safely by risk class:** fail-close là mặc định; ngoại lệ fail-open phải được phê duyệt, time-bound và observable.
10. **Compatibility before cleanup:** migrate theo strangler pattern, đo parity trước khi chuyển enforcement.
11. **No synchronous audit dependency:** audit pipeline không được chặn request; deny/allow vẫn tạo local event buffer.
12. **Bounded context:** PDP chỉ nhận thuộc tính tối thiểu, có nguồn gốc rõ ràng và giới hạn độ tươi.

## 6. Architecture Decision Records

### ADR-001 — Thin Edge Gateway, không phải business gateway

- **Quyết định:** Edge xử lý TLS termination, OIDC/JWT validation, request normalization, WAF/rate limit, routing và coarse authorization. Không truy vấn database nghiệp vụ và không chứa invariant của domain.
- **Lý do:** giữ latency và blast radius thấp; tránh tái tạo monolithic BFF.
- **Hệ quả:** domain service phải có PEP/library hoặc sidecar integration chuẩn.

### ADR-002 — Hybrid authorization: platform policy + domain authorization

- **Quyết định:** shared policy do Authorization Platform quản lý; kiểm tra ownership, trạng thái giao dịch, hạn mức và relationship động do domain service quyết định hoặc cung cấp facts.
- **Lý do:** PDP chung không nên trở thành bản sao của mọi domain database.
- **Hệ quả:** một request có thể cần hai bước: platform decision rồi domain invariant check; cả hai dùng chung decision context.

### ADR-003 — Distributed PDP trên hot path

- **Quyết định:** PDP chạy sidecar/daemon/node-local hoặc embedded library tùy runtime; policy bundle được push/pull từ control plane. Remote centralized PDP chỉ dùng cho use case không nhạy latency hoặc policy cần graph/facts tập trung.
- **Lý do:** giảm network hop và tránh single synchronous dependency.
- **Hệ quả:** phải quản lý bundle freshness, compatibility và fleet rollout.

### ADR-004 — Policy-as-code với pipeline bắt buộc

- **Quyết định:** policy nằm trong Git, có schema, unit test, scenario test, static analysis, review bởi owner và Security, ký artifact trước khi phân phối.
- **Lý do:** traceability và rollback tốt hơn chỉnh policy trực tiếp trên production UI.
- **Hệ quả:** thay đổi khẩn cấp dùng break-glass workflow có TTL và hậu kiểm.

### ADR-005 — Istio mTLS STRICT và workload identity

- **Quyết định:** mesh cấp workload identity, mã hóa east-west traffic; namespace đã migrate dùng `STRICT`, AuthorizationPolicy giới hạn service caller.
- **Lý do:** user authorization không thay thế service authentication và network segmentation.
- **Hệ quả:** rollout theo namespace/workload; cần kiểm kê traffic ngoài mesh và egress.

### ADR-006 — Chuẩn hóa Actor–Action–Resource–Context

- **Quyết định:** mọi PEP ánh xạ request sang một authorization contract chuẩn, độc lập transport/URL.
- **Lý do:** policy có thể tái sử dụng và audit có nghĩa nghiệp vụ.
- **Hệ quả:** vocabulary và ownership phải được governance chặt.

### ADR-007 — Không truyền token người dùng tùy tiện qua toàn bộ call chain

- **Quyết định:** gateway xác thực external token. Nội bộ ưu tiên workload identity cộng delegated user context có audience, TTL và phạm vi hẹp; token exchange nếu IAM hỗ trợ. Không forward bearer token nguyên bản sang service không đúng audience.
- **Lý do:** giảm replay và confused-deputy risk.
- **Hệ quả:** cần chuẩn hóa delegation contract và SDK.

### ADR-008 — Decision log bất đồng bộ, chống rò rỉ dữ liệu

- **Quyết định:** PEP/PDP phát audit event bất đồng bộ qua buffer/collector; redact/hash thuộc tính nhạy cảm, có retention và access control riêng.
- **Lý do:** audit đầy đủ mà không đặt logging backend trên critical path.
- **Hệ quả:** cần đo dropped events và cơ chế backpressure có giới hạn.

## 7. Target architecture

```text
                          ┌──────────────────────────┐
                          │ IAM / IdP                │
                          │ OIDC, OAuth2, JWKS       │
                          └────────────┬─────────────┘
                                       │ access token
                                       ▼
┌──────────┐   TLS   ┌───────────────────────────────────────┐
│ Clients  ├────────►│ Edge Gateway / Ingress               │
└──────────┘         │ AuthN, WAF, rate limit, routing, PEP  │
                     └──────────────────┬────────────────────┘
                                        │ mTLS + bounded identity context
                   ┌────────────────────┴────────────────────┐
                   │           Istio Service Mesh            │
                   │                                         │
                   │  ┌────────────┐  ┌────────────┐          │
                   │  │ Agent      │  │ Market     │  ┌───────┴──────┐
                   │  │ Domain     │  │ Domain     │  │ Broker Domain│
                   │  │ PEP + biz  │  │ PEP + biz  │  │ PEP + biz   │
                   │  │ AuthZ      │  │ AuthZ      │  │ AuthZ        │
                   │  └─────┬──────┘  └─────┬──────┘  └──────┬──────┘
                   │        │ local/nearby PDP evaluation     │
                   └────────┼─────────────────┼────────────────┘
                            ▼                 ▼
                     ┌───────────────────────────────┐
                     │ Distributed PDP instances     │
                     │ cached signed policy bundles │
                     └───────────────┬───────────────┘
                                     │ bundle/status/telemetry
                                     ▼
┌────────────────────────────────────────────────────────────────────┐
│ Authorization Control Plane                                        │
│ Policy repo -> CI/test -> compiler -> signer -> bundle registry     │
│ Vocabulary/schema registry | rollout controller | policy inventory  │
└───────────────────────────────┬────────────────────────────────────┘
                                │ async decision events
                                ▼
                      ┌──────────────────────┐
                      │ Audit / SIEM / DWH   │
                      └──────────────────────┘
```

### 7.1 Thành phần

| Thành phần | Trách nhiệm | Không chịu trách nhiệm |
|---|---|---|
| IAM/IdP | Danh tính, credential, MFA, token issuance, lifecycle | Quyền trên resource động của domain |
| Edge Gateway | North-south PEP, token validation, routing, rate limit, request hygiene | Business workflow/DB lookup |
| Istio | mTLS, workload identity, traffic policy, L4/L7 service allowlist | End-user business authorization |
| Authorization Control Plane | Policy lifecycle, test, build, sign, distribute, inventory, rollout | Phục vụ bắt buộc mọi request runtime |
| PDP | Evaluate input với policy bundle/facts cho phép | Sửa dữ liệu nghiệp vụ |
| PEP/SDK | Chuẩn hóa input, gọi PDP, enforce decision, emit telemetry | Tự phát minh role/action |
| Domain service | Resource lookup, ownership/relationship, invariant và field filtering | Parse external token theo cách riêng |
| Audit pipeline | Lưu, tìm kiếm, cảnh báo, đối soát quyết định | Tham gia đồng bộ vào quyết định |

## 8. Trust boundaries và threat assumptions

### 8.1 Trust boundaries

| Boundary | Từ -> đến | Kiểm soát bắt buộc |
|---|---|---|
| TB-1 Internet | Client -> Edge | TLS, WAF, DDoS/rate limit, token validation, xóa identity header không tin cậy |
| TB-2 Edge-to-mesh | Gateway -> service | mTLS, gateway workload allowlist, audience/route binding |
| TB-3 East-west | Service -> service | mTLS STRICT, workload authorization, egress control |
| TB-4 Policy supply chain | Git/CI -> registry -> PDP | protected branch, approval, tests, artifact signing, digest verification |
| TB-5 Context/data | Domain/data source -> PDP/PEP | schema validation, source identity, freshness/TTL, data minimization |
| TB-6 Audit | Runtime -> collector/SIEM | encryption, integrity, buffering, restricted access, retention |
| TB-7 Admin plane | Operator -> control plane | SSO/MFA, privileged role, four-eyes approval, immutable admin audit |

### 8.2 Threats chính và mitigations

- **Forged identity header:** strip tại Edge; nội bộ chỉ chấp nhận metadata từ trusted proxy hoặc signed delegation token.
- **JWT replay/confused deputy:** audience validation, short TTL, nonce/jti khi cần, token exchange và caller workload binding.
- **Stale permission:** giới hạn bundle/context TTL; revoke path ưu tiên; high-risk action có fresh lookup.
- **Policy tampering:** signed bundle, checksum, protected pipeline, separation of duties.
- **PDP bypass:** network policy chỉ cho traffic qua PEP path; service SDK enforce ở handler; conformance test.
- **Over-permissive wildcard:** linter cấm wildcard production không có exception/expiry.
- **Cross-tenant access:** tenant là thuộc tính bắt buộc ở Actor và Resource; mismatch deny trước business logic.
- **Decision-log leakage:** allowlist field, tokenization/hash, không log raw token hoặc payload.

## 9. Authentication model

### 9.1 Human/client authentication

- OIDC/OAuth 2.x access token do trusted issuer phát hành.
- Gateway xác minh chữ ký, issuer, audience, expiry/not-before, thuật toán và key rotation qua JWKS cache.
- Token chỉ chứa stable identity và coarse claims cần thiết; entitlement biến động nhanh không nên nhồi toàn bộ vào token.
- External identity headers (`x-user-id`, `x-roles`, `x-tenant-id`, tương tự) bị loại bỏ trước khi tạo trusted context.
- Với browser/BFF thật sự cần session handling, session boundary có thể giữ lại nhưng không sở hữu policy nghiệp vụ.

### 9.2 Workload authentication

- Mỗi workload có identity riêng do mesh/trust domain cấp, ví dụ SPIFFE-compatible identity.
- mTLS xác thực cả hai chiều; certificate ngắn hạn và tự động rotate.
- Không dùng shared API key giữa các service làm định danh chính.
- Service account được tách theo workload và environment; không dùng chung mặc định theo namespace.

### 9.3 Delegation

Khi Service A gọi Service B thay mặt user, quyết định phải phân biệt:

- `actor`: user/principal gốc;
- `caller`: workload đang thực hiện lời gọi;
- `delegation_chain`: chuỗi đã được xác minh, giới hạn độ dài;
- `audience` và `scope`: dành riêng cho callee/action;
- `request_id`/`trace_id`: liên kết audit.

Callee chỉ allow khi **cả actor và caller** hợp lệ cho action. Không để caller dùng quyền hệ thống để vượt quyền user trừ workflow được khai báo rõ.

## 10. Authorization model

### 10.1 Contract chuẩn

```json
{
  "actor": {
    "type": "user",
    "id": "usr_123",
    "tenant_id": "tenant_a",
    "roles": ["broker_operator"],
    "assurance_level": "mfa"
  },
  "action": "broker.order.approve",
  "resource": {
    "type": "broker_order",
    "id": "ord_789",
    "tenant_id": "tenant_a",
    "attributes": {"risk_tier": "high", "owner_id": "usr_456"}
  },
  "context": {
    "caller": "spiffe://prod.example/ns/broker/sa/broker-service",
    "channel": "web",
    "environment": "prod",
    "request_time": "2026-08-31T10:00:00Z",
    "trace_id": "...",
    "authn_age_seconds": 120
  }
}
```

Output tối thiểu:

```json
{
  "decision": "ALLOW",
  "decision_id": "dec_...",
  "policy_id": "broker-order-approval",
  "policy_version": "sha256:...",
  "reason_code": "ALLOW_APPROVER_SAME_TENANT",
  "obligations": {
    "mask_fields": ["customer.tax_id"],
    "max_rows": 100
  }
}
```

### 10.2 Semantics

- **Actor:** user, service, device hoặc job đã xác thực; không đồng nhất actor với caller.
- **Action:** động từ nghiệp vụ có namespace, ví dụ `market.quote.read`; không dùng trực tiếp `GET /v1/...` làm action.
- **Resource:** loại, ID, tenant và thuộc tính tối thiểu cần quyết định.
- **Context:** environment, channel, workload caller, assurance, time, network/device risk và delegation.
- **Decision:** `ALLOW` hoặc `DENY`; `INDETERMINATE/ERROR` được PEP ánh xạ theo fail policy, mặc định deny.
- **Obligation:** yêu cầu bắt buộc sau allow như masking, row limit, step-up auth. PEP không hiểu obligation thì phải deny.

### 10.3 Phân lớp quyết định

1. **Edge coarse policy:** route/action có yêu cầu authenticated, tenant, scope và client class phù hợp không?
2. **Mesh workload policy:** caller workload có được gọi callee/port/path class này không?
3. **Platform/domain policy:** actor có quyền action trên resource class/instance không?
4. **Business invariant:** state hiện tại có cho phép thao tác không, ví dụ order đã settled thì không thể sửa.
5. **Response authorization:** field/row filtering nếu policy trả obligation hoặc domain yêu cầu.

Quyết định cuối cùng là phép AND của mọi enforcement layer áp dụng. Một layer allow không thể override deny ở layer khác.

## 11. Policy model và policy-as-code lifecycle

### 11.1 Cấu trúc repository đề xuất

```text
policies/
  vocabulary/
    actions.yaml
    resources.yaml
    context.schema.json
  platform/
    tenant-isolation/
    workload-baseline/
  domains/
    agent/
    market/
    broker/
  tests/
    conformance/
    regression/
  bundles/
    manifest.yaml
```

Ngôn ngữ policy có thể là Rego/OPA, Cedar hoặc engine tương đương, nhưng phải hỗ trợ deterministic evaluation, test automation, bundle/version và explainability đủ dùng. Phase đầu khuyến nghị **OPA/Rego** nếu hệ sinh thái Kubernetes/Istio và năng lực đội ngũ phù hợp; quyết định engine cuối cùng cần benchmark bằng policy thật.

### 11.2 Quy tắc authoring

- Mặc định deny; allow phải explicit.
- Action/resource phải tồn tại trong vocabulary registry.
- Tenant isolation là guardrail bắt buộc, domain policy không được override.
- Policy không gọi network tùy ý trong lúc evaluate.
- Policy không chứa secret hoặc PII thô.
- Rule high-risk cần test deny, cross-tenant, stale context và privilege escalation.
- Exception phải có owner, ticket, reason và expiry.

### 11.3 Pipeline

```text
Pull request
 -> schema/lint
 -> unit + negative tests
 -> scenario/regression tests
 -> impact analysis trên decision samples đã ẩn danh
 -> owner + security approval
 -> compile/build immutable bundle
 -> sign + publish
 -> canary distribution
 -> health/parity gate
 -> progressive rollout
 -> promote hoặc rollback digest
```

### 11.4 Policy versioning

- Bundle được định danh bằng immutable digest; semantic label chỉ là alias.
- PDP báo `active_digest`, `loaded_at`, `last_successful_sync`.
- PEP ghi digest vào decision log và metric.
- Breaking schema change cần dual-read/dual-evaluate trong thời gian tương thích.
- Control plane duy trì N phiên bản gần nhất và nút rollback một bước có kiểm soát.

## 12. Request flows

### 12.1 External read request

```text
1. Client -> Edge: token + request
2. Edge: validate token, strip untrusted headers, map route -> action/resource hint
3. Edge PEP/PDP: coarse decision
4. Edge -> Domain Service: mTLS + verified bounded identity/delegation context
5. Service: load resource facts cần thiết
6. Service PEP/PDP: fine-grained decision
7. Service: enforce business invariant và obligations (field/row filtering)
8. Response -> Edge -> Client
9. Mỗi PEP phát decision event bất đồng bộ cùng trace_id
```

### 12.2 Mutating/high-risk request

- Yêu cầu idempotency key khi phù hợp.
- High-risk policy có thể yêu cầu MFA mới, device assurance hoặc fresh entitlement lookup.
- Authorization và mutation phải hạn chế TOCTOU: kiểm tra state/version ngay trong transaction hoặc optimistic concurrency control.
- Decision ALLOW không được cache qua thay đổi state nếu invariant phụ thuộc state.

### 12.3 Service-to-service request

```text
Service A --mTLS--> Service B
  mesh kiểm tra workload A được gọi B
  B xác minh delegation context/audience
  B đánh giá actor + caller + action + resource
  B enforce domain invariant
```

### 12.4 Async/event flow

- Producer ghi actor/caller/delegation metadata tối thiểu vào event envelope đã ký hoặc provenance được broker bảo đảm.
- Consumer authorization kiểm tra quyền tại thời điểm consume nếu hành động có side effect; không mặc định tái sử dụng ALLOW cũ vô thời hạn.
- Retry giữ correlation ID nhưng tạo decision ID mới.
- Dead-letter access bị giới hạn và audit.

## 13. Control plane và data plane

### 13.1 Control plane

Bao gồm policy repository, CI, compiler, signing, bundle registry, distribution controller, schema/vocabulary registry, inventory và admin API/UI. Control plane có thể tạm unavailable mà data plane vẫn phục vụ bằng last-known-good bundle trong giới hạn freshness.

### 13.2 Data plane

Bao gồm Gateway PEP, mesh enforcement, service PEP/SDK và PDP runtime. Không phụ thuộc synchronous vào Git, CI hoặc registry. Mỗi instance:

- chỉ load bundle có chữ ký hợp lệ;
- atomic swap bundle, không có trạng thái nửa cập nhật;
- giữ last-known-good bundle;
- expose readiness theo policy age và risk profile;
- giới hạn input size/evaluation time;
- không log secret/raw token.

## 14. Deployment topology

### 14.1 Môi trường

- Control plane tách dev/staging/prod; artifact được promote theo digest, không rebuild giữa môi trường.
- Policy data và audit data có residency phù hợp từng region.
- Trust domain, issuer và signing key tách production/non-production.

### 14.2 PDP placement

| Mô hình | Dùng khi | Trade-off |
|---|---|---|
| Sidecar | Isolation cao, policy/domain riêng, latency thấp | Tốn tài nguyên và vận hành nhiều instance |
| Node-local daemon | Nhiều workload đồng nhất, cần tối ưu resource | Blast radius theo node, cần auth channel local |
| Embedded library | Runtime ổn định, cực nhạy latency | Coupling version/language, rollout khó hơn |
| Remote regional PDP | Policy graph/context tập trung | Thêm network hop và dependency |

Mặc định đề xuất: **sidecar hoặc node-local PDP cho synchronous hot path**, remote PDP chỉ cho bài toán relationship graph hoặc administration cần dữ liệu tập trung. Chọn topology sau benchmark latency/cost và threat model.

### 14.3 Istio rollout

1. Inventory traffic và service accounts.
2. Bật telemetry, sau đó mTLS `PERMISSIVE` tạm thời.
3. Sửa traffic plaintext/ngoài mesh.
4. Chuyển namespace/workload sang `STRICT`.
5. Thêm allowlist theo caller identity; bắt đầu audit/shadow trước enforce.
6. Egress qua controlled gateway cho external dependency quan trọng.

## 15. Scaling, high availability và disaster recovery

### 15.1 Scaling

- Edge scale ngang theo RPS/concurrency/CPU; không giữ session nếu không bắt buộc.
- PDP stateless đối với request; policy bundle immutable trong memory.
- Bundle registry/CDN/object store phân phối theo region; jitter polling và backoff tránh thundering herd.
- Audit ingestion partition theo time/domain/tenant hash; producer buffer có giới hạn.
- Benchmark policy theo p50/p95/p99, input size, số rule và bundle size trước production.

### 15.2 HA

- Edge và remote PDP chạy tối thiểu đa AZ, anti-affinity và PodDisruptionBudget.
- Không có leader trên request path.
- JWKS và policy bundle dùng last-known-good cache với expiry rõ.
- Health check phân biệt process health, bundle health và dependency degradation.
- Circuit breaker cho remote PDP/context provider; retry chỉ với request idempotent và budget nhỏ.

### 15.3 DR

- Policy Git và bundle registry có replication/backup; signing key trong KMS/HSM, có rotation/recovery runbook.
- RPO control-plane metadata <= 15 phút; RTO <= 60 phút.
- Data plane tiếp tục bằng last-known-good bundle theo risk-tier TTL.
- Diễn tập mất region, expired JWKS, corrupt bundle và compromised signing key tối thiểu hai lần/năm.

## 16. Caching và freshness

### 16.1 Loại cache

- **JWKS cache:** theo HTTP cache header, proactive refresh, giữ key cũ trong cửa sổ rotation hợp lý.
- **Policy bundle cache:** immutable theo digest, atomic activation, last-known-good.
- **Decision cache:** chỉ cho quyết định thuần deterministic với key đầy đủ.
- **Attribute cache:** TTL theo nguồn dữ liệu và risk; có invalidation khi khả thi.

### 16.2 Decision-cache key tối thiểu

`actor_id + actor_entitlement_version + caller_id + action + resource_type + resource_id/version + tenant + relevant_context + policy_digest`

Không cache ALLOW nếu bỏ sót thuộc tính ảnh hưởng quyết định. High-risk mutation mặc định không cache ALLOW. DENY cache TTL ngắn để tránh kéo dài quyền đã vừa được cấp; chống enumeration bằng response thống nhất.

### 16.3 Freshness classes

| Class | Ví dụ | Bundle/context stale tối đa | Hành vi quá hạn |
|---|---|---:|---|
| Critical | approve payout, admin grant | 1–5 phút | Fail-close |
| High | mutate order/customer | 5–15 phút | Fail-close |
| Standard | read business data | 30–60 phút | Theo policy, mặc định close |
| Public/low-risk | public catalog | nhiều giờ | Có thể fail-open nếu explicit |

Các giá trị cuối cùng phải được Security và domain owner phê duyệt dựa trên risk assessment.

## 17. Failure semantics: fail-open và fail-close

### 17.1 Mặc định

- AuthN lỗi/không xác định: **fail-close**.
- Không có policy, input sai schema, obligation không hiểu, chữ ký bundle sai: **fail-close**.
- PDP timeout/crash: dùng local last-known-good nếu còn hợp lệ; nếu không, **fail-close**.
- Audit sink lỗi: request vẫn chạy nếu local buffer còn capacity; cảnh báo và degrade theo runbook, không tự fail-open authorization.

### 17.2 Ngoại lệ fail-open

Chỉ cho endpoint public hoặc read-only low-risk đã được đăng ký. Mỗi exception cần:

- owner và Security approval;
- phạm vi action/resource cụ thể;
- TTL/expiry;
- metric/alert riêng;
- response marker và audit reason;
- quarterly review.

Không fail-open cho cross-tenant access, privileged/admin action, secret/PII, money movement hoặc write/delete.

### 17.3 Degraded-mode matrix

| Sự cố | Hành vi |
|---|---|
| IdP unavailable, token còn hợp lệ và key cached | Tiếp tục đến hết token/key safety window |
| JWKS key mới chưa tải được | Từ chối token dùng key chưa biết; không bỏ qua signature |
| Control plane unavailable | Dùng last-known-good bundle còn trong freshness window |
| Bundle mới lỗi | Không activate; giữ bundle cũ và alert |
| Local PDP unavailable | Restart/fallback instance; mặc định deny |
| Attribute provider unavailable | Dùng cache còn hạn; hết hạn thì deny trừ exception low-risk |
| Audit backend unavailable | Buffer cục bộ có giới hạn, alert; không block request mặc định |

## 18. Observability và audit

### 18.1 Metrics

- Request count/latency/error theo gateway, service và action class.
- AuthN failure theo reason: expired, issuer, audience, signature, malformed.
- AuthZ decision count theo allow/deny/error/reason code, policy digest và enforcement mode.
- PDP evaluation p50/p95/p99, timeout, bundle load success/failure/age.
- Cache hit/miss/eviction và stale use.
- Shadow mismatch: legacy allow/new deny và legacy deny/new allow.
- Audit buffer depth/drop count và ingestion lag.
- mTLS coverage, plaintext attempt và denied workload call.

Không gắn raw actor/resource ID vào metric label để tránh cardinality và rò rỉ; chi tiết nằm trong audit log có kiểm soát.

### 18.2 Tracing

- Propagate `trace_id`/`request_id`; mỗi authorization evaluation có `decision_id`.
- Span ghi engine latency, policy digest, decision/reason code; không ghi raw token hoặc PII.
- Cho phép nối Edge -> service -> PDP -> audit bằng ID, không bằng payload nhạy cảm.

### 18.3 Decision audit schema tối thiểu

- timestamp, decision_id, trace_id;
- actor pseudonymous ID/type/tenant;
- caller workload identity và delegation chain digest;
- action, resource type và resource ID tokenized khi cần;
- decision, reason code, obligations;
- policy ID/digest, enforcement mode (`shadow|enforce`);
- PEP/PDP identity, environment/region;
- latency, cache status, context freshness;
- break-glass/exception metadata.

### 18.4 Audit controls

- Append-oriented storage, integrity protection, encryption at rest/in transit.
- RBAC riêng cho audit; truy cập audit cũng được audit.
- Retention theo pháp lý và phân loại dữ liệu; delete/anonymize theo policy.
- Alert cho privilege escalation, cross-tenant deny spike, break-glass, wildcard policy và shadow regression.

## 19. Security controls

- TLS hiện đại ở Edge; mTLS STRICT trong mesh sau migration.
- Key/signing secret trong KMS/HSM; rotation và revocation runbook.
- Supply-chain security: pinned dependencies, SBOM, image signing, admission policy.
- Policy bundle ký số và xác minh digest trước activate.
- Namespace/service-account least privilege, NetworkPolicy và controlled egress.
- Token không xuất hiện trong log, trace, error hoặc downstream header không cần thiết.
- Input validation và giới hạn kích thước/depth để chống policy-engine DoS.
- Timeout/evaluation budget và circuit breaker.
- Separation of duties giữa policy author, approver và production promoter cho high-risk policy.
- Break-glass yêu cầu MFA mạnh, ticket, TTL, scope hẹp, alert tức thời và retrospective.
- Penetration test tập trung bypass PEP, confused deputy, tenant isolation, cache poisoning và policy supply chain.

## 20. SLO và NFR

Các target dưới đây là baseline để validation qua load test và business criticality review.

| Hạng mục | Target |
|---|---|
| Edge availability | >= 99.99%/tháng cho production critical path |
| Local PDP availability | >= 99.99% theo workload SLI |
| AuthZ evaluation latency | p95 <= 5 ms, p99 <= 10 ms cho local cached policy, không gồm domain data lookup |
| Edge overhead do AuthN + coarse AuthZ | p95 <= 15 ms |
| Remote PDP latency nếu dùng | p95 <= 30 ms nội vùng, timeout budget explicit |
| Policy propagation | p95 <= 2 phút standard; emergency revoke <= 30 giây nếu hạ tầng hỗ trợ |
| Bundle activation correctness | 100% bundle phải verify signature/schema; activation atomic |
| Decision audit delivery | >= 99.99% trong 5 phút; dropped event = 0 mục tiêu |
| mTLS coverage | 100% traffic in-scope sau migration |
| Capacity | >= 2x peak dự báo, chịu mất một AZ mà vẫn giữ SLO |
| Scalability | Không yêu cầu coordination per-request; scale ngang |
| Data isolation | Không có cross-tenant ALLOW ngoài policy được phê duyệt |

Error budget của Edge/PDP được quản lý cùng SRE. Policy rollout tự động dừng khi latency, error hoặc mismatch vượt threshold.

## 21. Migration strategy

### 21.1 Nguyên tắc migration

- Strangler pattern theo route/action, không thay toàn bộ BFF cùng lúc.
- Giữ external contract ổn định trước; tối ưu contract sau khi traffic đã chuyển an toàn.
- Tách ba loại logic trong BFF: shared security, channel composition, business rule.
- Chỉ shared security chuyển vào platform; business rule chuyển về domain; composition có giá trị được giữ thành Experience API/BFF mỏng.
- Mọi cutover có rollback route và owner trực.

### 21.2 Pha 0 — Discovery và baseline

- Inventory endpoint, client, issuer/audience, role/scope, downstream, data classification và SLO của `agent-api`, `market-api`, `core-broker-api`.
- Static/runtime analysis để lập authorization matrix hiện tại.
- Ghi nhận baseline latency, availability, deny rate và incident.
- Phân loại endpoint: public, read, write, privileged, cross-tenant-sensitive.
- Xác định BFF nào là pure proxy, composition BFF hay chứa business logic.

**Exit criteria:** 100% endpoint in-scope có owner, action/resource mapping và risk class.

### 21.3 Pha 1 — Foundation

- Triển khai Edge Gateway, trusted issuer config và header sanitization.
- Thiết lập Istio identity/mTLS ở chế độ quan sát/permissive.
- Xây policy repo, CI, signing, registry, PDP runtime, PEP SDK và audit schema.
- Định nghĩa vocabulary ban đầu cho Agent/Market/Broker.
- Viết conformance suite và golden decision cases từ hành vi legacy.

**Exit criteria:** end-to-end non-production, signed bundle, audit nối được trace, chaos test cơ bản đạt.

### 21.4 Pha 2 — Shadow evaluation

Legacy BFF vẫn enforce. PEP mới evaluate cùng input nhưng không ảnh hưởng response:

```text
Request -> legacy decision (enforced)
       \-> new PDP decision (shadow) -> comparison/audit
```

Phân loại mismatch:

- legacy ALLOW / new DENY: có thể thiếu mapping/context hoặc policy mới chặt hơn;
- legacy DENY / new ALLOW: rủi ro privilege expansion, chặn rollout;
- error/indeterminate: lỗi integration hoặc freshness.

**Exit criteria đề xuất:** tối thiểu 7–14 ngày hoặc đủ chu kỳ nghiệp vụ; 0 mismatch chưa giải thích ở action high-risk; >= 99.99% parity ở standard actions; PDP SLO đạt ở peak test.

### 21.5 Pha 3 — Enforce theo cohort

1. Internal/test clients.
2. Read-only low-risk routes.
3. Write routes theo tenant/client cohort 1% -> 5% -> 25% -> 50% -> 100%.
4. Privileged/high-risk routes sau cùng.

Mỗi bước có automated gate dựa trên deny delta, error, latency và business KPI. Rollback bằng route flag/policy digest, không cần redeploy toàn hệ thống.

### 21.6 Pha 4 — Decompose BFF

#### `agent-api`

- Di chuyển token validation/coarse scopes lên Edge.
- Chuyển agent ownership/hierarchy checks về Agent domain hoặc domain facts provider.
- Giữ composition endpoint nếu phục vụ UX cụ thể; đổi tên/ownership rõ nếu cần.

#### `market-api`

- Chuyển public/read market routes trực tiếp qua Edge nếu không có composition.
- Market entitlement/licensing là domain policy/fact; cache theo TTL phù hợp dữ liệu thị trường.
- Tách public-data fail behavior khỏi licensed/private data.

#### `core-broker-api`

- Ưu tiên cuối do risk cao.
- Approval, money movement, customer/portfolio access luôn fail-close.
- Business state check nằm trong Broker domain transaction để tránh TOCTOU.
- Yêu cầu step-up auth và audit đầy đủ cho privileged action.

### 21.7 Pha 5 — Decommission

- Ngừng route mới vào BFF cũ; theo dõi zero traffic qua ít nhất một retention window hợp lý.
- Thu hồi client credential, service account, network policy và secret cũ.
- Archive code/config/audit mapping theo retention; cập nhật runbook và dependency map.
- Không xóa BFF còn composition value; chuyển nó thành thin Experience API với guardrail platform.

## 22. Rollout, feature flags và rollback

Mỗi action/route có trạng thái:

```text
OFF -> OBSERVE -> SHADOW -> ENFORCE_CANARY -> ENFORCE -> LEGACY_REMOVED
```

Guardrail:

- Flag được scope theo environment, route, action, tenant/client cohort.
- Không cho flag bật fail-open ngoài registry đã duyệt.
- Promotion yêu cầu policy digest cố định và dashboard health.
- Auto-pause khi new-allow/legacy-deny mismatch > 0 với critical action, hoặc deny/error/latency vượt threshold.
- Rollback ưu tiên về last-known-good digest hoặc legacy enforcement; mọi rollback tạo audit event và incident review nếu production impact.

## 23. Testing strategy

- **Policy unit tests:** allow/deny, boundary, missing attribute, wildcard, tenant isolation.
- **Golden tests:** tái hiện quyết định legacy đã được xác nhận đúng.
- **Property tests:** không actor nào vượt tenant; unknown action luôn deny; privilege monotonicity theo rule đã định.
- **Contract tests:** PEP input/output, obligation compatibility, schema evolution.
- **Integration tests:** gateway, token issuer/JWKS rotation, mesh identity, PDP, domain lookup.
- **Replay tests:** traffic sample đã ẩn danh để impact analysis.
- **Performance tests:** peak/2x peak, bundle lớn, cold start, cache miss.
- **Chaos tests:** PDP crash, registry outage, stale/corrupt bundle, audit outage, AZ loss, clock skew.
- **Security tests:** forged header, wrong audience, token replay, caller spoofing, confused deputy, cross-tenant IDOR, policy tampering.
- **Migration tests:** shadow parity, cohort rollback và dual-version compatibility.

## 24. Governance và ownership

### 24.1 RACI tóm tắt

| Năng lực | Accountable | Responsible | Consulted |
|---|---|---|---|
| IAM/token profile | Identity/Security | IAM team | Platform, domains |
| Edge Gateway | Application Platform | Gateway team/SRE | Security, domains |
| Istio/workload identity | Platform/SRE | Service Mesh team | Security |
| AuthZ control plane/PDP/SDK | Security Platform | Authorization team | SRE, domains |
| Global guardrail/tenant isolation | CISO/Security Architecture | Security Platform | Domain owners |
| Domain policy/vocabulary | Domain owner | Domain team | Security Platform |
| Business invariant | Domain owner | Domain service team | Security |
| Audit/SIEM | Security Operations | SecOps/Data Platform | Legal, Privacy, SRE |
| SLO/on-call | Platform + owning domain | SRE/service team | Security |

### 24.2 Policy ownership model

- Platform policy có `CODEOWNERS` của Security Platform.
- Domain policy cần domain owner; high-risk rule cần Security đồng phê duyệt.
- Vocabulary addition cần Architecture/API governance để tránh trùng nghĩa.
- Quarterly access/policy review; exception và break-glass review thường xuyên hơn.
- Deprecation có notice period, usage telemetry và migration guide.

### 24.3 Operating model

- Authorization Platform cung cấp paved road: SDK, templates, examples, test harness, dashboards và runbooks.
- Domain team không tự fork SDK/policy engine; extension thông qua versioned interface.
- Architecture Council giải quyết tranh chấp boundary: shared policy hay business invariant.
- Security incident có kill switch theo action/resource/tenant và emergency bundle pipeline.

## 25. Risks và mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Gateway trở thành monolith | Coupling, blast radius | Thin gateway rule, architecture fitness tests, cấm domain DB access |
| PDP/control plane thành SPOF | Outage diện rộng | Distributed PDP, LKG bundle, multi-AZ/region, no control-plane dependency on hot path |
| Policy drift/stale bundle | Quyền sai | Digest telemetry, freshness classes, revoke channel, rollout health gate |
| Sai mapping legacy | Deny hợp lệ hoặc privilege expansion | Inventory, golden tests, shadow mode, cohort rollout |
| Context provider tăng latency | SLO miss | Domain prefetch, bounded attributes, cache, local invariant check |
| Confused deputy | Service vượt quyền user | Actor + caller check, audience-bound delegation, mTLS identity |
| Cross-tenant IDOR | Data breach | Mandatory tenant guardrail, negative/property tests, fail-close |
| Audit chứa PII | Compliance exposure | Data minimization, tokenization, access control, retention |
| Policy language khó dùng | Adoption thấp, lỗi policy | SDK/templates, lint/explain, training, review, limited constructs |
| Mesh complexity | Operational incidents | Phased permissive->strict, inventory, SRE ownership, runbooks |
| Chi phí sidecar cao | Resource overhead | Benchmark sidecar vs node-local, right-sizing, shared bundle |
| Dual enforcement kéo dài | Complexity | Deadline/exit criteria per route, migration scorecard |

## 26. Alternatives considered

### A. Tiếp tục một BFF cho mỗi domain/channel

- **Ưu:** thay đổi nhỏ, team autonomy ngắn hạn.
- **Nhược:** tiếp tục copy policy, drift, khó audit và tăng hop.
- **Kết luận:** không chọn làm target; chỉ giữ BFF có composition/experience value thật.

### B. Một API Gateway làm toàn bộ AuthZ và business logic

- **Ưu:** điểm enforce rõ, client đơn giản.
- **Nhược:** gateway thành monolith, cần domain data, latency/blast radius lớn.
- **Kết luận:** loại bỏ.

### C. Central remote PDP cho mọi request

- **Ưu:** policy/facts tức thời, vận hành tập trung.
- **Nhược:** network hop, bottleneck, dependency và failure amplification.
- **Kết luận:** chỉ dùng có chọn lọc; hot path ưu tiên distributed evaluation.

### D. Chỉ dùng Istio AuthorizationPolicy

- **Ưu:** gần traffic, tốt cho workload/network policy.
- **Nhược:** không đủ cho resource-level business authorization và domain facts phức tạp.
- **Kết luận:** dùng như một lớp, không phải giải pháp toàn bộ.

### E. Chỉ dùng authorization library trong từng service

- **Ưu:** latency thấp, gần business logic.
- **Nhược:** dễ version drift, thiếu central governance và cross-language consistency.
- **Kết luận:** SDK/embedded có thể là PEP/PDP runtime, nhưng policy lifecycle vẫn phải tập trung.

### F. SaaS/managed authorization service

- **Ưu:** time-to-market nhanh, UI/relationship model sẵn.
- **Nhược:** data residency, latency, cost, lock-in và availability dependency.
- **Kết luận:** đánh giá bằng PoC nếu relationship-based authorization là nhu cầu chính; vẫn phải giữ abstraction PEP contract.

## 27. Open questions và decision gates

Các câu hỏi này không chặn phê duyệt kiến trúc tổng thể nhưng phải đóng trước production enforcement:

1. IAM/IdP nào là nguồn chuẩn, có hỗ trợ token exchange và workload federation không?
2. Policy engine nào đạt benchmark với policy Agent/Market/Broker thực tế?
3. Relationship data nào cần centralized graph, data nào phải ở domain?
4. Data classification và audit retention chính thức cho từng domain/region?
5. Emergency revocation SLA thực tế của IAM, bundle distribution và context cache?
6. Endpoint nào thực sự cần channel-specific BFF composition?
7. Ownership/on-call 24x7 cho Edge, PDP và signing pipeline?
8. Ngưỡng parity/rollout cuối cùng theo business criticality?

## 28. Implementation roadmap đề xuất

| Giai đoạn | Thời lượng tham khảo | Deliverable chính |
|---|---:|---|
| Discovery | 3–5 tuần | Inventory, action/resource vocabulary, baseline, risk classes |
| Foundation | 6–10 tuần | Edge/mesh baseline, policy CI, signed bundles, PDP/SDK, audit |
| Pilot | 4–6 tuần | Một read path của Agent hoặc Market ở shadow rồi enforce |
| Domain rollout | 2–4 quý | Agent -> Market -> Broker theo risk và readiness |
| Decommission/optimize | liên tục | Xóa pure auth proxy, right-size, tighten mTLS/policy |

Thời lượng phụ thuộc maturity của IAM, Kubernetes/Istio, telemetry và mức business logic đang nằm trong các BFF.

## 29. Acceptance criteria

Giải pháp được coi là sẵn sàng production cho một action khi:

- action/resource có owner, schema và risk class;
- AuthN issuer/audience và workload identity được xác minh;
- policy có positive/negative/cross-tenant tests và signed digest;
- PEP không bypass được qua route/network thông thường;
- shadow parity đạt threshold và mọi privilege-expansion mismatch đã đóng;
- latency/availability/load/chaos test đạt SLO;
- audit event đầy đủ, không chứa token/PII ngoài allowlist;
- fail mode, freshness, cache và rollback đã được test;
- dashboard, alert, runbook và on-call owner tồn tại;
- Security, domain owner và SRE ký duyệt cho high-risk action.

## 30. Recommended first slice

Chọn một endpoint **read-only, authenticated, single-tenant, lưu lượng vừa và không có composition phức tạp** từ `agent-api` hoặc `market-api`. Pilot phải đi hết chuỗi Edge -> mTLS -> domain PEP/PDP -> audit, không chỉ demo policy engine. Sau khi đạt parity và SLO, mở rộng sang write path mức standard; `core-broker-api` và privileged action đi sau khi fail-close, fresh-context và transactional invariant đã được chứng minh.

## 31. Kết luận

Kiến trúc đề xuất loại bỏ sự lặp lại AuthN/AuthZ bằng cách biến authorization thành một platform capability nhưng không tước quyền sở hữu nghiệp vụ khỏi domain. Edge giữ mỏng, mesh xác thực workload, PDP đánh giá policy gần workload, control plane quản lý policy tập trung, còn domain service enforce invariant dựa trên dữ liệu thật. Mô hình này giảm policy drift và BFF proliferation, đồng thời tránh tạo một centralized authorization bottleneck mới.

Quyết định quan trọng nhất để giữ kiến trúc bền vững là ranh giới: **shared access policy thuộc Authorization Platform; business truth và invariant thuộc domain service; mọi boundary đều được xác minh và audit bằng một contract thống nhất.**
