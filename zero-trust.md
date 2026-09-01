# L2 — AP-CBFF — Centralized BFF và Zero Trust Platform

| Thuộc tính | Giá trị |
|---|---|
| Document ID | AP-CBFF-L2 |
| Phiên bản | 2.6 |
| Trạng thái | **READY FOR ARCHITECTURE REVIEW** |
| Ngày | 01/09/2026 |
| Architecture Owner | Chief Architect |
| Phạm vi | Đánh giá hợp nhất `agent-api`, `market-api`, `core-broker-api` thành một BFF logic tập trung |
| Route ứng viên pilot | `market.order.read` — owner-declared, chưa xác minh |
| Quyết định được yêu cầu | Conditional approval cho kiến trúc đích, G0 và reversible pilot |
| Không thuộc quyết định này | Production rollout, product adoption, numeric SLO hoặc ROM rollout chưa có bằng chứng |
| TDD chính thức | Một tài liệu duy nhất: `zero-trust.md` |

---

## 0. Quản trị tài liệu

### 0.1 Mục đích, thẩm quyền và cách đọc

Đây là L2 TDD duy nhất của AP-CBFF: ranh giới, Zero Trust guardrail, contract,
migration và exit. OpenAPI, policy, manifest, benchmark, threat model, runbook và
route plan là L3 evidence trong system of record; chúng không phải TDD thứ hai và
không được làm yếu L2.

Board review §1, §3.1–3.3, §7.2–7.4 và §8. Board phê duyệt `D-01..05`;
change authority chỉ hoạt động trong guardrail. Evidence producer không là sole
gate approver. Tài liệu không tự cấp production approval; route chỉ chuyển trạng
thái khi gate `PASS` còn hiệu lực và change authority phê duyệt.

### 0.2 Ngôn ngữ quyết định và bằng chứng

`MUST`, `MUST NOT`, `SHOULD`, `MAY` là normative. Tiếng Việt là ngôn ngữ
chính; tên chuẩn, field, product và status giữ nguyên khi dịch làm mất nghĩa.

Stage gate chỉ có bốn outcome:

| Outcome | Nghĩa |
|---|---|
| `NOT_READY` | Thiếu owner, target hoặc evidence bắt buộc |
| `PASS` | Đạt target đã duyệt, có locator, thời điểm đo và reviewer |
| `FAIL` | Không đạt target hoặc vi phạm security floor |
| `N/A` | Không áp dụng, có rationale và approver |

`OWNER_DECLARED`, `HYPOTHESIS`, `NOT_MEASURED`, `OBSERVED` và
`MEASURED` là claim-evidence level, không phải gate outcome. Numeric target
chỉ binding khi có nguồn, phương pháp/window đo, owner và approver.

Decision status: `FOR_APPROVAL`, `CONDITIONALLY_APPROVED`, `APPROVED`,
`SUPERSEDED`. Record ghi decision/version/status, approver, conditions,
condition verifier, date và minutes locator. Chỉ Board đổi status `D-01..05`.

### 0.3 Lịch sử phiên bản

| Phiên bản | Trạng thái | Thay đổi |
|---|---|---|
| 2.0–2.2 | Superseded | Thiết kế ban đầu và các vòng bổ sung control/gate |
| 2.3 | Superseded | Board brief, alternatives, fact base, complexity budget và exit strategy |
| 2.4 | Superseded | Executive narrative; một decision source; G0 learning cap; nhập operational controls vào core; rút L2 |
| 2.5 | Superseded | Topology-neutral D-01; pre-registered measurement; final-PDP/delegation PoC; proportional gate review |
| 2.6 | Current | One-page Board TL;DR; quantified value floor; mandatory no-reframe abort boundaries |

---

## 1. Câu chuyện điều hành và quyết định của Board

### 1.1 TL;DR cho Board — quyết định trong một trang

**Vấn đề cần kiểm chứng.** `HYPOTHESIS`: `agent-api`, `market-api` và
`core-broker-api` đang lặp hoặc diễn giải khác các control identity, action,
authorization, error, resilience và telemetry. Hệ quả nghi ngờ là cùng actor/action
qua hai entry point có decision hoặc audit khác nhau; một security change phải sửa,
phối hợp và phát hành nhiều nơi. Chưa có code, trace, incident, traffic, cost hoặc
ownership evidence đủ để xác nhận. G0 phải chứng minh hoặc bác bỏ giả thuyết; tên
service không được dùng thay bằng chứng.

**Board được xin gì.** Approve tám guardrail làm floor: không singleton hoặc domain
state/database trong BFF; stable route/action; per-hop workload identity và bounded
delegation; final authorization tại resource boundary; signed/LKG configuration;
bounded failure/consistency; migration reversible. Conditional-approve năm quyết
định: tập trung logical contract/control nhưng defer deployment topology; giữ
domain/workflow ownership; cấm raw-token forwarding; giữ policy/mesh
product-neutral; migration theo route có stop-loss. Authorize G0 tối đa **10 ngày
làm việc/30 person-days**, ba API identifier và một route ứng viên. Board chưa duyệt
production traffic, rollout, numeric SLO, procurement hoặc topology.

**Board mua được gì trong cap.** Ngày 5 kiểm tra access, owner và scope. Trước khi
xem kết quả, `SG-0` khóa tập alternative × scope, comparator, công thức/threshold
value, decision mapping và owner. Ngày 10 trả measured fact base, `H-01`
pilot-funding verdict, evidence status `H-02..05`, final-PEP/delegation feasibility,
topology comparison plan và pilot ROM. Không procurement, persistent platform
dependency hoặc production traffic; quá cap/scope cần decision record mới.

**Rủi ro lớn nhất.** Governance/runtime có thể lớn hơn năng lực vận hành trong khi
owner, fact base hoặc domain final PEP chưa tồn tại. Vì vậy success không phải “HTTP
proxy works” hoặc tạo một service lớn: phải không có unexpected authorization allow,
giữ isolation/rollback và vượt comparator trên value, latency, reliability,
operational load và unit cost theo target đăng ký trước. OPA, Istio, SPIRE và các
product khác chỉ là candidate.

**Bốn kết cục tại ngày 10.** `CONTINUE` mở một pilot có cap/funding/gates riêng khi
security, ownership và value floor đạt; `PAUSE` đóng băng thêm G0 work nhưng không
gia hạn G0—muốn discovery/remediation thêm phải `RESTART` bằng decision/`SG-0` mới;
`PIVOT` chọn alternative còn compliant và đạt value floor, gồm secure-in-place như
một kết quả thành công; `ABORT` bắt buộc khi hết cap mà không có funded
owner/compliant path hoặc không permitted scope nào đạt pre-registered value floor.
Sunk cost, đổi tên scope hoặc threshold hậu nghiệm không được biến `ABORT` thành
`PIVOT`/`CONTINUE`; chi tiết tại §7.4.

### 1.2 Hồ sơ quyết định duy nhất

Board là approval authority cho toàn bộ bảng. Endorser và change authority không
được làm yếu `GR-01..08` hoặc tự đổi decision status.

| ID / status | Decision | Alternatives considered | Trade-off, condition và exit | Endorsers / change authority |
|---|---|---|---|---|
| `D-01` / `FOR_APPROVAL` | Centralize logical contract/controls; production deployment topology deferred | Pilot compares modular shared runtime và deployable theo ownership; retain as-built/no consolidation là abort; gateway-only cho pure proxy | Tránh premature topology nhưng tăng learning scope; G0 pre-register comparator, pilot đóng `H-02` | Program, Runtime, Domain / Board chọn production topology; Architecture Owner chỉ trong topology đã duyệt |
| `D-02` / `FOR_APPROVAL` | BFF coarse PEP/composition; domain giữ data/invariant/final PEP; workflow giữ saga state | Baseline in-domain transaction; relationship PDP/local domain evaluator là conditional; BFF/mesh-only và BFF-owned saga bị reject | Option phải chứng minh snapshot/version consistency, latency/failure và ownership; không cutover trước `SG-2` | Domain, Security / Domain + Security |
| `D-03` / `FOR_APPROVAL` | Per-hop workload identity + bounded delegation; mechanism do PoC chọn; cấm raw token | Acquisition: RFC 8693 hoặc IAM-issued assertion; representation: self-contained hoặc opaque + introspection; actor header bị reject | PoC so hot-path dependency/cache, binding, attenuation, replay/revocation, rotation/audit; assertion phải đạt control equivalence | IAM, Security / Security + Architecture |
| `D-04` / `FOR_APPROVAL` | Product-neutral; ưu tiên local/near PDP và existing identity/network | OPA/Cedar/managed PDP; Istio modes/Linkerd/gateway; existing issuer/SPIRE | PoC compatibility, failure, skill, capacity, cost; OPA/Istio chỉ là candidates; boundary change quay lại Board | Security, Platform/SRE / Architecture + platform owner |
| `D-05` / `FOR_APPROVAL` | Route strangler với cap/deadline và `PAUSE/PIVOT/ABORT` | Reject big bang và dual-run vô hạn | Reversible nhưng chậm; transition cần `PASS`; scope/funding/value fail → §7.4 | Product, Delivery / route pause/rollback theo gate; Board duyệt pivot/abort/restart |

Binding interpretation: SPIFFE là standard, SPIRE là implementation và chỉ thêm
khi existing issuer không đạt requirement. RFC 8693 biểu diễn được
subject/actor/audience nhưng không tự enforce profile; IAM-issued assertion phải
đạt cùng caller/actor/audience/action/tenant/expiry/JTI, sender/replay và audit floor
tại `D-03`. Introspection là validation/revocation mechanism, không phải
delegation protocol. PKI/product/data-plane mode cần version-pinned ADR/PoC.
Internal assertion MUST được cryptographically integrity-protect hoặc là opaque
handle do trusted authority validate; minting issuer MUST được authorize cho
actor/audience. Conformance MUST reject forged, tampered, replayed, wrong-issuer
và wrong-audience assertion.

### 1.3 Yêu cầu Board và hợp đồng học tập

Board được đề nghị:

1. Approve `GR-01..08` làm security floor có hiệu lực với scope AP-CBFF;
   conditional-approve `D-01..05`.
2. Authorize G0 trong cap 10 ngày/30 person-days; ngày 10 ra một exit decision.
3. Cho phép chuẩn bị reversible pilot; shadow chỉ sau `SG-2`, canary sau
   `SG-3`. Target-default vẫn cần production approval record tại `SG-4`.

Board không duyệt numeric SLO, rollout ROM, Istio mode, SPIRE/OPA adoption hoặc
production traffic trong vòng này. G0 phải trả lại pilot ROM/capacity trước build.

---

## 2. Hiện trạng, phạm vi và giả thuyết

### 2.1 Fact base

| Chủ đề | Claim hiện có | Mức bằng chứng | Evidence cần có trước promotion |
|---|---|---|---|
| Scope | Ba API identifier do owner đưa vào chương trình | `OWNER_DECLARED` | Repo, deployment, catalog và boundary |
| Problem | Cross-cutting controls có thể lặp hoặc phân kỳ | `HYPOTHESIS` | Code/config diff, change lead time, incident/audit evidence |
| Pilot | `market.order.read` là ứng viên | `OWNER_DECLARED` | Method/path, consumer, handler, downstream, side-effect proof |
| Capability | Chưa biết API là channel BFF, gateway, domain hay workflow | `NOT_MEASURED` | Context map và owner interview |
| Runtime | Traffic, latency/error, payload/fan-out, capacity và cost chưa đo | `NOT_MEASURED` | Telemetry có time window và query |
| Security/data | AuthN/AuthZ, identity, bypass, data/dependency path chưa quan sát | `NOT_MEASURED` | Config, trace, code/data graph và negative tests |
| Ownership | Code/deploy/rollback/on-call chưa được xác nhận | `NOT_MEASURED` | Repo IAM, pipeline, pager và named authority |

Tên service không được dùng để suy diễn domain hoặc team. Claim dùng cho gate phải
là `OBSERVED` hoặc `MEASURED`, có locator và observed-at.

### 2.2 Hồ sơ bằng chứng và measurement contract G0

Mỗi API/route trong cohort có một versioned record trong system of record hiện
hữu:

| Nhóm | Trường tối thiểu |
|---|---|
| Vai trò/value | Capability, consumer/use case; duplicated `control_id`, implementation location, semantic diff và incident/audit gap |
| Change flow | Sample/window; repo/deployment/owner phải chạm; lead time, coordination/rework và operator effort |
| Contract/execution | Method/path, resource/action, data/side effect/consistency, downstream/DB access, sync/async và resilience |
| Security | Issuer/audience, actor/client/caller, current AuthZ, final PEP, bypass và delegation capability |
| Runtime/cost | Traffic, latency/error/payload/fan-out, compute/platform cost trên route/request và capacity owner |
| Measurement | Cohort/comparator, operational definition, query/window/exclusion, threshold/rubric, raw locator, owner, approver và version |

`SG-0` MUST đăng ký as-built baseline và enumerate một tập hữu hạn permitted
`alternative × scope`: secure-in-place bắt buộc, các candidate physical topology và
reduced/gateway scope nếu áp dụng. Value measurement MUST dùng cùng cohort, time
horizon, currency và cost basis. Với mỗi permitted pair `a`, G0 tính tối thiểu:

- `duplication_ratio = duplicated control implementation instances / total control
  implementation instances` trong cohort;
- `divergence_rate = duplicated control groups có semantic decision/audit mismatch /
  duplicated control groups được kiểm tra`;
- `change_amplification = median số repo, deployment và accountable owner phải chạm
  cho một representative cross-cutting control change`;
- `benefit_a = annualized avoided duplicated-change effort + avoided attributable
  incident/audit-remediation loss + avoided baseline runtime/operations cost`, chỉ
  nhận phần so với as-built có evidence locator và approved loaded-cost/risk value;
- `TCO_a = annualized migration amortization + added platform/runtime/on-call cost`
  so với as-built trên cùng horizon;
- `net_value_a = benefit_a - TCO_a`; `value_ratio_a = benefit_a / TCO_a` khi
  `TCO_a > 0`, nếu không thì ghi `N/A`.

Một saving chỉ được ghi ở `benefit_a` hoặc giảm `TCO_a`, không cả hai; model này ghi
saving ở benefit nên `TCO_a < 0` là model error. Mọi ratio yêu cầu denominator `> 0`;
denominator bằng `0` là `N/A`, không phải `0`/vô cực và không tự tạo `PASS`.

`SG-0` MUST pre-register riêng absolute materiality floor theo currency, ratio hurdle
và topology-uplift floor. Một pair chỉ đạt G0 value floor trên conservative case khi
`net_value_a` vượt absolute floor và, nếu `TCO_a > 0`, `value_ratio_a` vượt ratio
hurdle; ratio hurdle tối thiểu là `1.0` (break-even) hoặc cao hơn theo investment/risk
policy. `H-01 PASS` chỉ chứng minh ít nhất một control-convergence scope có business
case đủ để fund reversible pilot so với as-built. Nó không chọn physical topology.
Pilot/`SG-4` mới đóng `H-02`: physical topology `a` chỉ thắng khi
`net_value_a - net_value_secure_in_place` vượt pre-registered topology-uplift floor
và không làm yếu guardrail/SLO. G0 dùng defensible conservative estimate; `SG-3`
MUST khóa bounded-canary measurement plan và `SG-4` MUST refresh cùng model bằng
measured canary evidence.

Product/Program và Architecture sở hữu materiality/hurdle; Security phê duyệt risk
valuation, SRE phê duyệt measurement method. Material security divergence vượt risk
appetite có thể bắt buộc remediation nhưng không tự chứng minh consolidation.
`SG-0` MUST pre-register metric/rubric, alternative × scope, comparator, horizon,
cost basis, threshold, owner/approver, `PAUSE` trigger và mapping sang
`CONTINUE/PAUSE/PIVOT/ABORT` trước khi quan sát G0 result. G0 baseline được dùng để
đặt pilot targets trước pilot. Thay đổi sau khi xem result phải versioned, có
rationale/approver và không áp dụng hồi tố.

### 2.3 Giả thuyết chưa đóng và điều kiện dừng

| ID | Claim phải đóng | Nếu không chứng minh được |
|---|---|---|
| `H-01` | Ít nhất một permitted control-convergence scope vượt G0 value floor so với as-built và đáng fund reversible pilot | Nếu mọi pair fail/không defensible khi hết cap thì hard-abort `HA-02`; `PASS` không chọn topology |
| `H-02` | G0 lập comparison plan/owner/threshold; pilot/`SG-4` chứng minh physical topology có incremental value so với secure-in-place cùng isolation/cadence/cost | Pivot secure-in-place nếu nó đạt floor; không physical-consolidate khi topology uplift không đạt |
| `H-03` | `market.order.read` read-only, mirror-safe và rollbackable | Chọn route khác; không shadow |
| `H-04` | Domain có final PEP; IAM có bounded delegation path | Giữ legacy, tạo domain facade hoặc technology/reduced-scope pivot |
| `H-05` | Có accountable owner/capacity và existing platform đáp ứng security floor | Pause, giảm scope hoặc abort; không mặc định thêm platform |

### 2.4 Scope

Trong scope: ingress-to-BFF và BFF-to-domain trust boundary; route/action contract;
identity, delegation và AuthZ; composition/consistency; resilience, audit,
release, ownership, cost và migration.

Ngoài scope: hợp nhất domain database; viết lại domain service; BFF làm distributed
transaction coordinator; UI; vendor/cloud procurement; multi-region active-active;
production adoption của OPA/Istio/SPIRE trước PoC.

---

## 3. Kiến trúc đích và ranh giới trách nhiệm

### 3.1 Guardrail cấp Board

| ID | Guardrail normative | Chi tiết canonical |
|---|---|---|
| `GR-01` | Một logical product nhưng runtime MUST có nhiều replica/failure domain; request path MUST stateless và không có singleton bắt buộc | §3.2–3.3, §5.1/§5.5 |
| `GR-02` | BFF MUST NOT sở hữu domain data, business invariant, durable workflow/idempotency state hoặc truy cập domain database/repository | §3.3, §4.5 |
| `GR-03` | Mọi request MUST có canonical parse và resolve đúng một stable `route_id/action_id`; unknown/ambiguous/collision bị reject; registry chỉ là signed/versioned configuration | §3.3, §4.6, Phụ lục A |
| `GR-04` | Actor/client/caller MUST tách biệt; internal hop MUST workload-authenticated và expected-flow default-deny; egress MUST declared/SSRF-protected; downstream credential MUST short-lived và audience/action/tenant-bound; raw token bị cấm | §4.1–4.3, Phụ lục A |
| `GR-05` | BFF PEP chỉ early-reject; domain/application boundary MUST final-authorize trên current resource facts; bypass không được tạo unexpected allow | §4.2–4.5 |
| `GR-06` | Image và route/policy/trust config MUST integrity/provenance-verified, versioned, compatible, schema-validated và preload; readiness MUST expose active/LKG version; invalid evidence không được thành allow và rollback MUST có | §4.2, §5.1/§5.4, Phụ lục A |
| `GR-07` | Deadline, retry, fan-out, cache, consistency, audit và resource use MUST bounded/explicit; BFF không giả lập atomic cross-domain snapshot | §4.4–4.6, §5 |
| `GR-08` | Mỗi route migration MUST reversible; không promotion nếu owner, contract, security, reliability, cost và rollback evidence áp dụng chưa `PASS` | §7 |

`GR-01..08` có hiệu lực từ ngày trong Board approval record đến khi superseded.
GR row là Board-level invariant; mapped `MUST/MUST NOT` clauses là binding L2
refinement thuộc approved floor. Detail MAY thu hẹp nhưng không làm yếu GR.
Mapping/conflict mơ hồ làm evidence `NOT_READY` và cần sửa L2. Làm yếu
guardrail cần threat/risk review cùng Board approval.

### 3.2 Kiến trúc logic

```mermaid
flowchart TB
    Consumer["Consumers / Channels"]
    Edge["DNS / CDN / WAF / L7 Gateway"]

    subgraph BFF["Logical BFF product — deployment topology selected by D-01"]
        Guard["Guard + Dispatcher"]
        Registry["Signed Route / Action Snapshot"]
        Identity["Identity Context"]
        CoarsePEP["Coarse PEP + local/near PDP"]
        Modules["Channel Modules"]
        Delegation["Delegation Client"]
        Compose["Bounded Composition"]
        Result["Canonical Response / Error"]
        Audit["Audit / Telemetry"]

        Guard --> Registry
        Guard --> Identity --> CoarsePEP --> Modules
        Modules --> Delegation
        Modules --> Compose --> Result
        CoarsePEP -. decision .-> Audit
        Result -. outcome .-> Audit
    end

    Domain["Domain API(s) — as-built boundary TBD<br/>resource truth + final PEP"]
    Data[("Domain-owned data")]
    Workflow["Application / Workflow Service<br/>durable state + recovery"]
    IAM["IAM / OIDC / STS or approved delegation authority"]
    Control["Signed policy/config distribution"]
    Network["Workload identity / expected-flow enforcement"]
    Obs["Observability / SIEM"]

    Consumer --> Edge --> Guard
    Delegation --> IAM
    Modules -->|"mTLS + delegated actor"| Domain
    Modules -->|"cross-domain command"| Workflow
    Workflow -->|"per-domain credential"| Domain
    Domain --> Data
    Control -. "distribution, not request dependency" .-> CoarsePEP
    Control -.-> Domain
    Network -.-> BFF
    Network -.-> Domain
    Audit --> Obs
    Domain -. outcome .-> Obs
```

Sơ đồ là logical responsibility model; BFF boundary MAY nằm trong một hoặc nhiều
deployable theo D-01. Nó không khẳng định số domain service, database, trust domain
hoặc team. G0 phải thay generic nodes bằng context map có evidence.

### 3.3 Ranh giới trách nhiệm và tách triển khai

| Boundary | Sở hữu | Không được sở hữu |
|---|---|---|
| Edge + Guard | TLS/WAF, sanitization, size/schema, request/deadline ID, coarse abuse | User/resource AuthZ hoặc business semantics |
| BFF kernel | Route snapshot, identity context, coarse PEP và bounded delegation | Standalone registry, raw-header trust hoặc final resource decision |
| Channel module | Consumer contract, bounded composition/representation và domain call | Domain data/invariant, saga hoặc direct DB access |
| Domain API | Data/invariant, final PEP, idempotency và domain outcome | Channel representation |
| Workflow service | Cross-domain durable state, recovery và operation status | UI shaping hoặc domain invariant |
| Control plane | Build/sign/distribute/rollback policy/config snapshot | Synchronous request-path dependency |

Allowed dependency là `consumer → edge → BFF → domain/workflow → domain data`.
Forbidden: BFF → domain DB, distributed write fan-out, undeclared bypass, arbitrary
policy lookup hoặc legacy/target loop.

Chỉ extract module khi measured scale, isolation, cadence, security hoặc ownership
biện minh; extraction giữ action taxonomy, delegation, audit và final PEP.
Một logical product không bắt buộc một deployment. Schema federation không thuộc
target; contract/composition technology là lựa chọn L3 theo evidence và không
thay identity, delegation, final PEP hay domain ownership.

---

## 4. Zero Trust, hợp đồng và luồng trọng yếu

### 4.1 Trust, workload identity và network

Parent floor: `GR-04` và `GR-06`.

Theo NIST SP 800-207/207A, network location không tạo implicit trust; decision
dựa trên subject, workload, resource, action và context; enforcement nằm gần
resource và tạo telemetry có thể kiểm tra.

Mỗi request có `actor`, OAuth `client`, workload `caller`, bounded
`delegation_chain` và tenant/risk evidence có provenance/freshness. Edge/Guard
MUST strip client-supplied actor/caller/role/tenant/policy/SPIFFE headers; BFF tái
tạo context từ credential đã verify. Downstream verify caller và delegation độc
lập.

Workload identity MUST ngắn hạn, tự động rotate và bind workload provenance.
Prod/non-prod không vô tình chia trust root. Một identity namespace chỉ có một
authority model tại một thời điểm; migration/federation cần ADR, collision
analysis, overlap, rollback và compromise drill. SPIFFE có thể là identity
contract; SPIRE chỉ là một implementation candidate.

Trong internal scope đã khai báo, mỗi hop MUST dùng mTLS hoặc tương đương để mã
hóa in transit và xác thực workload; target posture MUST default-deny.
Expected-flow policy MUST bind verified workload identity với declared
service/method/path/port; IP hoặc namespace name đơn lẻ không phải authorization
principal. Egress destination/protocol/host/data class MUST được khai báo.
Client-controlled arbitrary URL bị cấm; redirect, metadata endpoint, loopback và
private-address range phải được chống SSRF.

### 4.2 Delegation, authorization và policy lifecycle

External token MUST NOT được forward nguyên trạng. Delegation profile pin issuer,
subject/actor, client/caller, audience, action/scope, tenant, issued-at, expiry,
chain bound, sender constraint khi cần, cache key và revocation behavior. Cache
không sống lâu hơn minimum token/session/policy/security freshness bound. STS
outage không được mở rộng audience hoặc fallback raw token. Partner/outbound
credential MUST scoped theo destination/audience và không dùng chung toàn BFF.

BFF coarse PEP đánh giá actor/caller/action/tenant/channel/risk để reject sớm.
Domain đọc current resource snapshot, cung cấp facts/version cho final PEP và
enforce trên cùng snapshot. Mutation dùng transactional hoặc optimistic
concurrency check nếu resource có thể đổi sau decision.

Final enforcement vẫn ở resource boundary. Option selection:

| Final-PDP option | Disposition và evidence |
|---|---|
| In-domain code trên transaction snapshot | Baseline; chứng minh same-snapshot/concurrency, testability và domain ownership |
| Relationship-based PDP | Chỉ khi có relationship requirement; chứng minh model owner, resource/version token, freshness/invalidation, latency và outage posture |
| Local evaluator trong domain | Chỉ khi facts co-located; chứng minh deterministic input, snapshot binding, bundle/LKG và failure semantics |
| BFF/mesh-only decision | Reject vì thiếu current resource facts |

Không thêm authorization service chỉ vì industry precedent. Mọi option MUST trả
fact/resource version để enforcement và mutation concurrency check dùng cùng
decision context.

Canonical effect là `ALLOW | DENY | CHALLENGE | INDETERMINATE`, kèm reason,
policy version, fact/version reference, `valid_until`, obligations và decision
ID. `INDETERMINATE` MUST NOT thành `ALLOW`; high-risk action mặc định deny.
Chỉ allowlisted obligation được thực thi.

Policy evaluator không tùy ý gọi network. Bundle/snapshot MUST schema-validated,
tested, signed, staged và rollbackable; invalid/empty/expired/incompatible config
không activate. Replica không nhận protected traffic trước khi route, trust/JWKS
và policy snapshot hợp lệ được preload. Managed KMS là key floor; HSM/dual control
chỉ khi key class, external obligation hoặc threat model yêu cầu.

### 4.3 Luồng đọc đã xác thực từ bên ngoài

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant Edge
    participant BFF as BFF Guard / Dispatcher
    participant Registry as Signed Route Snapshot
    participant Coarse as Identity + Coarse PEP
    participant Module as Channel Module
    participant Delegation as Delegation Client
    participant IAM as STS / Delegation Authority
    participant Domain as Domain API + Final PEP

    Client->>Edge: TLS request
    Edge->>BFF: Forward
    BFF->>BFF: Sanitize + deadline
    BFF->>Registry: Resolve route
    Registry-->>BFF: Route/action profile
    BFF->>Coarse: Verify context + decide
    Coarse-->>BFF: Coarse decision
    BFF->>Module: Execute use case
    Module->>Delegation: Credential for audience/action
    Delegation->>IAM: Exchange/assertion request
    IAM-->>Delegation: Bounded credential
    Module->>Domain: mTLS + delegated actor
    Domain->>Domain: Current facts + final PEP
    Domain-->>Module: Result / typed deny
    Module-->>Client: Response via BFF/Edge
```

Registry chỉ trả config; Delegation Client làm token operation; Module gọi Domain;
Domain làm final PEP. Deny/challenge dừng trước data.

### 4.4 Nhất quán cho đọc tổng hợp

| Profile | Contract |
|---|---|
| `EVENTUAL` | Mỗi source trả local committed version; không hứa latest/same-time |
| `BOUNDED_STALENESS` | Component không cũ hơn approved bound; quá bound fail/degrade rõ |
| `AS_OF` | Chỉ khi mọi domain hỗ trợ cùng logical-time/cursor contract |
| `READ_YOUR_WRITES` | Client đưa opaque commit marker; owner domain verify/wait/fail |
| `ATOMIC_SNAPSHOT` | Chỉ owning read-model/application service được hứa; BFF fan-out bị cấm |

Response MUST công bố profile, generated-at và per-component source/version/
observed-at/stale/degraded. Missing không được đổi thành empty; stale/partial không
được che bằng false success. Numeric staleness nằm trong route contract.

### 4.5 Mutation và cross-domain workflow

Single-domain mutation đi qua final PEP và domain transaction. Idempotency key
bind actor/action/tenant/payload hash; BFF không retry mutation khi domain chưa có
idempotency contract.

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant BFF as BFF Module + Coarse PEP
    participant IAM as Delegation Authority
    participant Workflow as Workflow Service + PEP
    participant Domain as Domain API + Final PEP

    Client->>BFF: Command + idempotency key
    BFF->>BFF: Coarse PEP
    BFF->>IAM: Workflow credential
    IAM-->>BFF: Bounded credential
    BFF->>Workflow: mTLS + credential + command
    Workflow->>Workflow: Authorize + persist state
    loop Step / recovery
        Workflow->>IAM: Per-domain delegation
        IAM-->>Workflow: Domain credential
        Workflow->>Domain: mTLS + credential + step
        Domain->>Domain: Current facts + final PEP
        Domain-->>Workflow: Result / failure / commit marker
    end
    Workflow-->>BFF: Operation status/result
    BFF-->>Client: Response
```

Workflow chọn compensation, forward recovery hoặc bounded manual repair; không
giả định mọi effect reversible. Async worker MUST re-authorize sensitive action
tại execution; queued credential/policy/resource state không mặc định còn hợp lệ.

### 4.6 Hợp đồng chuẩn

| Contract | Semantic floor |
|---|---|
| Request context | Request/attempt/operation ID; route/action/channel/version; deadline; verified actor/client/caller/delegation; tenant/risk; trace |
| Route record | Matcher, owner, action/risk/authn, module/downstream, timeout/retry/fan-out, policy/final PEP, consistency/cache, migration, version/signature |
| AuthZ decision | Decision ID, effect, reason, policy version, evaluated/valid-until, facts/resource version, obligations |
| Error | HTTP status, stable safe code/message, request ID, retryable, optional `Retry-After`, field violations |
| Outcome | `REJECTED_BEFORE_DOWNSTREAM`, `DOMAIN_DENY`, `SUCCESS`, `PARTIAL`, `TIMEOUT`, `CANCELLED`, `ROLLED_BACK` |

Error taxonomy phân biệt authentication, authorization, validation, conflict,
consistency, rate limit, dependency unavailable và timeout. Domain deny/partial/
timeout không được collapse thành generic 500 hoặc false success.

---

## 5. Độ tin cậy, vận hành và chi phí

### 5.1 Hành vi khi lỗi

| Tình huống | Hành vi bắt buộc | Bị cấm |
|---|---|---|
| Policy/config control plane lỗi | Dùng valid LKG trong approved freshness window; hết window theo risk profile | Empty/unsigned config hoặc allow vô hạn |
| STS/delegation lỗi | Dùng cache còn hợp lệ hoặc typed failure | Raw-token fallback/mở rộng audience |
| Domain chậm/lỗi | Deadline, cancellation, bulkhead; partial chỉ theo contract | Retry ở mesh và app cùng lúc; che required failure |
| Shared quota lỗi | Local safety vẫn giữ; optional bounded lease theo risk | Bỏ limiter hoặc fail-open costly action |
| Revocation path lỗi | Short TTL/freshness ceiling giới hạn exposure | Cache allow vô thời hạn |
| Audit pipeline lỗi | Theo audit class; durable-required action dùng approved fail posture | Ghi success khi durability contract chưa đạt |
| Replica/AZ lỗi | Nhiều replica/failure domain, không singleton hot path | Bypass PEP để giữ availability |

### 5.2 Deadline, retry, cô lập và cache

Deadline chỉ giảm qua mỗi hop. Mỗi dependency có đúng một retry owner; retry nằm
trong remaining deadline/budget và chỉ cho transient class. Mutation retry cần
idempotency. Fan-out, payload, concurrency và queue có bound theo route/module/
tenant/downstream; overload shed trước unbounded queue.

Response/decision/credential cache key bind mọi input đổi outcome. TTL là minimum
của token, session, decision, policy, fact và security bound. Raw token, secret,
full role list và PII không cần thiết không vào cache/log/trace.

### 5.3 Giới hạn tốc độ, quota và thu hồi

Edge áp coarse IP/client/body abuse control; mỗi BFF replica MUST có local
concurrency/queue/burst safety. Distributed user/tenant quota chỉ thêm khi
fairness/abuse evidence yêu cầu và phải có bounded failure mode; domain sở hữu
quota cho costly/resource/business operation. 429 theo RFC 6585 §4; khi biết
finite retry time SHOULD gửi `Retry-After` theo RFC 9110 §10.2.3. Nếu expose
`RateLimit` fields, L3 pin phiên bản IETF specification đã review; RFC 9333
không phải nguồn cho HTTP RateLimit fields.

Short-lived credential và hard TTL/freshness ceiling là revocation baseline.
Durable invalidation event chỉ thêm khi approved exposure chặt hơn TTL; event
phải versioned, idempotent/replayable và có observable lag/dead-letter. Mất event
path không được loại bỏ hard ceiling.

### 5.4 Quan sát, audit, readiness và phát hành

Parent floor: `GR-06` và `GR-07`.

Telemetry MUST correlate request, actor pseudonym, caller, action,
resource/version, policy/config version, decision, downstream, consistency và
final outcome. Raw token, full role list và PII không cần thiết MUST NOT xuất hiện
trong baggage, log, metric hoặc trace. Audit-of-record tách khỏi metric/trace và
tuân thủ data class.

Action class `DURABLE_AUDIT_REQUIRED` chỉ acknowledge success sau khi durability
contract đạt. Retention, residency, deletion và audit-failure posture cần
Data/Compliance Authority phê duyệt khi áp dụng.

Release MUST dùng immutable digest, SBOM/signature/provenance, default-off route,
route-scoped canary và rollback drill. Authorized health/readiness cung cấp
image/config/policy/trust version, LKG age và activation failure. Protected
readiness preload trust/JWKS/policy; synchronous first-request fetch bị cấm.
Breaking contract cần explicit version, consumer migration, sunset và N/N-1
compatibility evidence cho rolling deployment.

### 5.5 Hiệu năng, năng lực, tính sẵn sàng và chi phí

L2 không đặt sẵn latency, SLO, RTO/RPO, headroom hoặc unit-cost target. G0 tạo
baseline; route owner duyệt target/measurement plan trước load/shadow. Pilot đo
warm/cold, rotation, degraded control plane, burst và rollout; tách end-to-end
latency/error khỏi BFF, proxy/waypoint, PDP, STS/quota và telemetry overhead.

Capacity model gồm route/policy/xDS growth, fan-out/queue, cache churn, audit
volume và per-replica CPU/memory. Cost model:

`incremental cost = replica count × (BFF + proxy/waypoint + local PDP resources)
                 + shared control-plane/quota/audit/network/storage cost`

Không claim mesh overhead/savings nếu thiếu measured input, window và
cost/request. BIA quyết định DR; restore test gồm route/policy/trust metadata,
audit continuity và reconciliation. BFF pod state không phải backup asset.

---

## 6. Quyền sở hữu và ngân sách độ phức tạp

### 6.1 Quyền quyết định

Đây là logical roles, không phải mandate lập team. Architecture/Program sở hữu
scope và exit recommendation/execution; Board duyệt topology pivot/abort/restart.
Route/Product sở hữu value, priority và cutover; Runtime sở hữu
deploy/rollback/on-call; Domain sở hữu data/invariant/final PEP; IAM/Security sở
hữu identity/delegation/security floor; Platform/SRE sở hữu
network/telemetry/reliability/cost; Data/Compliance sở hữu classification và
external obligations. Stage-gate table là authority map.

Một team có thể giữ nhiều role nếu decision rights và approval separation rõ.
Product không risk-accept compliance. Legacy owner accountable tới `SG-5`;
target owner nhận deploy/rollback/on-call trước canary.

### 6.2 Ngân sách độ phức tạp

Không thêm runtime dependency, control plane, persistent store hoặc ceremony nếu
chưa có requirement/evidence cụ thể. Mỗi thứ được thêm phải có owner, failure
behavior, rollback, observability và cost.

- Reuse existing identity/network/KMS; không mặc định thêm SPIRE, new mesh hoặc
  HSM ceremony.
- Registry là signed snapshot; không tạo registry service, quota/event/gate store
  khi local safety, TTL và system of record hiện hữu đủ.
- Estimation dùng dependency/capacity/three-point range; mô phỏng chỉ khi input
  quality và planning maturity biện minh.

Self-service/template chỉ mở rộng sau pilot. Deployable extraction hoặc thay đổi
team topology chỉ dựa trên measured scale, release cadence, failure isolation,
cognitive load và ownership—not service name.

---

## 7. Migration, stage gates và chiến lược thoát

### 7.1 Phân loại route và vòng đời

Trước `SG-1`, mỗi route trong cohort MUST có đúng một primary class dựa trên
context evidence:

| Primary class | Target place |
|---|---|
| Pure proxy/cross-cutting | Edge hoặc BFF kernel |
| Channel representation | Channel module |
| Single-domain rule | Domain service/API |
| Cross-domain stateful workflow | Application/workflow service |
| Direct data access | Tạo domain API; route cutover bị chặn |
| Obsolete/duplicate | Retire; không port mặc định |

Không phân loại từ tên `agent`, `market` hoặc `broker`.

```mermaid
stateDiagram-v2
    state "TARGET DEFAULT" as TARGET_DEFAULT
    [*] --> CANDIDATE
    CANDIDATE --> DISCOVERED: SG-1 PASS
    DISCOVERED --> SHADOW: SG-2 PASS
    SHADOW --> CANARY: SG-3 PASS
    CANARY --> TARGET_DEFAULT: SG-4 PASS
    TARGET_DEFAULT --> RETIRED: SG-5 PASS
    SHADOW --> DISCOVERED: semantic/security failure
    CANARY --> SHADOW: rollback trigger
    TARGET_DEFAULT --> CANARY: regression
```

Gate áp dụng theo route/cohort; route không thừa hưởng `PASS` của route khác.
Transition record tối thiểu có state, evidence locator, measured-at, owner và
approver.

### 7.2 G0 — tranche học tập và pilot

`SG-0` record phải ratify hoặc thay **aggregate** cap đề xuất: 10 ngày, 30
person-days, ba API identifier và một pilot; checkpoint ngày 5 và exit decision ngày
10. Cap chỉ bao gồm discovery/remediation trong G0; `PAUSE` không dừng clock, thêm
person-days hoặc tạo extension. Discovery/remediation G0 sau cap cần `RESTART` với
decision, funding và `SG-0` mới. `CONTINUE/PIVOT` MAY mở một pilot record có cap,
funding và gates riêng; đó không phải G0 extension. G0 chỉ được inventory/interview,
review code/config/trace, baseline
và bounded feasibility spike bằng tooling hiện hữu—không production, procurement,
wide migration hoặc persistent platform dependency mới.

Output là measured fact/hypothesis verdict, owner/capacity,
final-PEP/delegation feasibility, topology comparator, pilot ROM và
pre-registered pilot targets. Chưa có record thì không bắt đầu; quá hạn không tự
gia hạn. `market.order.read` chỉ thành pilot sau
`SG-1` xác minh mirror safety; shadow/canary theo `SG-2/SG-3`. Toàn
estate chỉ reconcile trước retirement, không trước cohort học đầu tiên.

### 7.3 Sáu cổng giai đoạn

| Gate | Decision và minimum `PASS` evidence | Accountable → approver | Khi fail |
|---|---|---|---|
| `SG-0 G0 authorized` | Board record có finite alternative × scope, aggregate scope/cap/date, hypothesis, funded roles, absolute/ratio/uplift thresholds, `PAUSE` trigger, decision mapping và exit authority | Program Owner → Architecture Board | Không bắt đầu platform build |
| `SG-1 Route discovered` | Context/classification, value, owner, data/security/dependency, mirror safety và rollback rõ | Route Owner → Architecture + affected Domain/Security | Giữ legacy, đóng gap/chọn route khác |
| `SG-2 Shadow ready` | Contract/action, delegation, final PEP, signed config/LKG/readiness và negative tests đạt | Runtime + Domain + Security → Security + Architecture | Không shadow |
| `SG-3 Canary ready` | Shadow không có unexpected allow; approved semantic/latency/error/capacity/cost/revocation/rollback target đạt; bounded-canary plan khóa secure-in-place comparator và H-02 uplift measurement | Route + SRE → Product + Security + Architecture | Rollback/optimize/pivot |
| `SG-4 Target-default ready` | Canary không regression; `H-02 PASS` bằng measured uplift/isolation/cadence/cost evidence; SLO/on-call/DR rõ; production record có locator | Route + SRE → named Production Change Authority trong `SG-0` record, với Product/Security/Architecture endorsement | Trở về shadow/canary |
| `SG-5 Legacy retired` | Không legitimate traffic; credential/DNS/network/deploy path removed; recovery/archive theo class | Legacy + Target → Product + Security + Architecture | Legacy chưa retired |

Security floor không được waive bằng legacy baseline. Target biến thiên phải được
route/risk owner duyệt trước phép đo. Data/Compliance Authority approve thêm khi
liên quan classification, residency, retention, audit hoặc external obligation.
Không gate nào do evidence producer tự phê duyệt một mình.

Review MAY batch trong cùng Board sitting và reuse cùng evidence locator; mỗi pack
MAY là one-page index tới raw evidence. Gate outcome vẫn tách: `SG-4` MUST NOT
`PASS` trước canary observation của `SG-3`. Pre-authorization/delegation
không phải pre-approval của evidence chưa tồn tại.

### 7.4 CONTINUE / PAUSE / PIVOT / ABORT

| State | Trigger | Hành động bắt buộc |
|---|---|---|
| `CONTINUE` | Fact base, owner, value, final PEP/delegation và pilot envelope đạt | Mở đúng stage/cohort kế tiếp |
| `PAUSE` | Trước aggregate cap: evidence/owner/capacity tạm thiếu; security floor, rollback hoặc revocation fail nhưng chưa chạm hard boundary | Dừng/rollback cohort, fail theo risk profile, revoke path/credential khi cần; chỉ remediation/evidence trong G0 cap. Sau cap, additional G0 work cần `RESTART`; pilot chỉ mở bằng `CONTINUE/PIVOT` và record riêng |
| `PIVOT — secure in place` | Zero Trust controls khả thi nhưng central runtime không đạt value/latency/cost/blast-radius envelope | Giữ BFF deploy riêng; dùng shared action, identity, delegation, final PEP và audit |
| `PIVOT — reduced scope` | Chỉ proxy/read phù hợp; mutation/workflow không phù hợp | Centralize route phù hợp; stateful use case ở domain/workflow |
| `PIVOT — technology` | PDP/mesh/STS/identity candidate fail nhưng alternative đạt guardrail | Rollback cohort; ADR + threat/ROM; thử alternative đã review |
| `ABORT consolidation` | Chạm `HA-01` hoặc `HA-02`; không còn permitted pivot đạt security/value floor | Dừng build; rollback controlled legacy/hybrid; revoke pilot route/credential; archive evidence; security remediation tách sang BAU |
| `RESTART` | Owner, capacity, business case và architecture decision mới đã có | Bắt đầu G0 mới; không reuse stale `PASS` |

`PIVOT — secure in place` là successful outcome khi shared controls tạo
measured value và runtime consolidation không đạt approved envelope.
Sunk cost không phải lý do `CONTINUE`.
Route owner MAY invoke immediate `PAUSE`/rollback theo gate mà không chờ Board.
Program đưa recommendation; Board duyệt `CONTINUE/PIVOT/ABORT/RESTART` tại
G0 và mọi thay đổi topology `D-01`. Resumption cần gate approver.

Hai hard boundary dưới đây là normative và không được reframe:

1. **`HA-01 — no compliant owned path`:** khi aggregate G0 cap/deadline đã hết,
   không candidate nào đồng thời có named funded owner và evidence-backed feasible
   path đạt mọi applicable `GR-01..08`, chương trình MUST `ABORT consolidation`.
2. **`HA-02 — no defensible value in permitted set`:** khi aggregate G0 cap/deadline
   đã hết và mọi pre-enumerated `alternative × scope` đều `FAIL` absolute value floor
   hoặc vẫn `NOT_MEASURED` vì không lập được defensible evidence, chương trình MUST
   `ABORT consolidation`. Physical topology fail uplift nhưng secure-in-place
   `PASS` thì là `PIVOT — secure in place`, không phải `HA-02`.

`HA-01/02` MUST NOT bị đổi thành `PAUSE`, `PIVOT` hoặc `CONTINUE` bởi sunk cost,
đổi tên cohort/scope, threshold hậu nghiệm hay replan. `PIVOT`
chỉ hợp lệ khi alternative cụ thể vẫn đạt guardrail và pre-registered value floor.
`ABORT consolidation` không waive security issue: remediation cần thiết tiếp tục
theo BAU/risk process riêng. Khởi động lại cần decision/funding/`SG-0` mới và không
được reuse stale `PASS`.

§2.2 là acceptance criterion của `D-01`; `HA-01/02` là binding stop-loss của
`D-05`, không phải decision source thứ hai.

### 7.5 Đường găng và bằng chứng L3

Critical path:
`context/ownership → final-PEP gap → identity/delegation → signed config/policy
→ shadow → rollback/canary → measured value/cost → target-default → retirement`.

Không tạo 13 artefact/tool bắt buộc. Bốn evidence pack là logical views có thể
link tới system of record hiện hữu:

| Pack | Nội dung | Gate chính |
|---|---|---|
| `EP-01 Current State & Ownership` | Context/dependency/data/identity, owners, pre-registered measurement, baseline/value và pilot ROM | SG-0/SG-1 |
| `EP-02 Contract & Security` | OpenAPI/schema, action, AuthZ/delegation/trust, consistency, threat và conformance | SG-2 |
| `EP-03 Performance & Operations` | SLI, load/fault/rotation/revocation/rollback, capacity/cost, audit/DR/runbook | SG-3/SG-4 |
| `EP-04 Migration & Decisions` | Cohort state, approval, ADR, legacy-FR traceability, exit và retirement | All |

Mỗi evidence item ghi scope, source/query, observed-at, owner, reviewer, freshness,
result và immutable/versioned locator.

---

## 8. Kết luận trình duyệt

Residual risks: central blast/cost, thiếu final PEP/bypass, candidate failure,
platform queue và dual-run. Containment nằm tại `D-01/02/04`, complexity
budget và `SG-5/ABORT`; legacy không waive risk.

**Recommendation:** conditional-approve §1.3; chưa approve production. Thiếu
`SG-0` record hoặc Board không chấp nhận learning cap/exit
authority thì disposition là `PAUSE`, không phải approval ngầm.

---

## Phụ lục A. Mức tuân thủ bắt buộc

- `GR-03` — Parser: spoofed header, normalization, smuggling và route collision.
- `GR-04/GR-05` — AuthZ: cross-tenant/resource, stale/spoofed facts,
  `INDETERMINATE` và bypass.
- `GR-04` — Identity: plaintext, expiry, audience/action/hop mismatch, replay
  và token leak.
- `GR-06` — Config/supply: signature, provenance, version, rollback,
  rotation, readiness và N/N-1.
- `GR-02/GR-04/GR-07/GR-08` — Data/operation: SSRF, PII leak, shadow safety,
  idempotency, consistency và reconciliation.

---

## Phụ lục B. Nguồn quy chuẩn và tham khảo

Reviewed on 01/09/2026. Mutable product documentation MUST được version-pin trong
`EP-02/EP-03` trước production use.

### B.1 Architecture, identity và security

- NIST SP 800-207, *Zero Trust Architecture*:
  https://csrc.nist.gov/pubs/sp/800/207/final
- NIST SP 800-207A:
  https://csrc.nist.gov/pubs/sp/800/207/a/final
- CISA Zero Trust Maturity Model v2.0:
  https://www.cisa.gov/resources-tools/resources/zero-trust-maturity-model
- SPIFFE specifications/overview và SPIRE concepts:
  https://spiffe.io/docs/latest/spiffe-specs/
  https://spiffe.io/docs/latest/spiffe-about/overview/
  https://spiffe.io/docs/latest/spire-about/spire-concepts/
- OAuth Token Exchange, RFC 8693; Mutual TLS, RFC 8705; Security BCP, RFC 9700;
  Token Introspection, RFC 7662:
  https://www.rfc-editor.org/rfc/rfc8693.html
  https://www.rfc-editor.org/rfc/rfc8705.html
  https://www.rfc-editor.org/rfc/rfc9700.html
  https://www.rfc-editor.org/rfc/rfc7662.html

### B.2 Technology candidates

- OPA integration, bundles và Envoy:
  https://www.openpolicyagent.org/docs/integration
  https://www.openpolicyagent.org/docs/management-bundles
  https://www.openpolicyagent.org/docs/envoy
- Cedar authorization model và Amazon Verified Permissions:
  https://docs.cedarpolicy.com/auth/authorization.html
  https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html
- Istio sidecar/ambient/security model:
  https://istio.io/latest/docs/overview/dataplane-modes/
  https://istio.io/latest/docs/ambient/overview/
  https://istio.io/latest/docs/ops/deployment/security-model/
- Linkerd server policy và automatic mTLS:
  https://linkerd.io/docs/features/server-policy/
  https://linkerd.io/docs/features/automatic-mtls/
- OpenTelemetry specification:
  https://opentelemetry.io/docs/specs/otel/

### B.3 HTTP semantics

- 429 Too Many Requests, RFC 6585 §4:
  https://www.rfc-editor.org/rfc/rfc6585.html#section-4
- `Retry-After`, RFC 9110 §10.2.3:
  https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.3
- HTTP `RateLimit` fields phải pin reviewed IETF specification; không cite
  RFC 9333 vì RFC đó không định nghĩa RateLimit fields.

### B.4 Books used for synthesis

- *Zero Trust Networks, 2nd Edition*, Razi Rais et al.: inventory, context-aware
  trust, enforcement và incremental adoption.
- *Istio in Action*, Christian E. Posta and Rinor Maloku: mesh/gateway,
  resilience, observability và service identity.
