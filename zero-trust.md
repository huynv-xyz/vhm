> **TÀI LIỆU NỘI BỘ - L2 UNDER REVIEW**
>
> Tài liệu này là Technical Design Document L2 kiêm Developer Implementation Guide cho Zero Trust Access, Workload and Authorization Platform. Các từ MUST, MUST NOT, SHOULD, SHOULD NOT và MAY mang nghĩa chuẩn tắc. Mọi ví dụ sản phẩm chỉ là reference implementation; architecture contract không phụ thuộc một vendor cụ thể.

# L2 - ZT-PLATFORM - Zero Trust Access, Workload and Authorization Platform

| Thuộc tính | Giá trị |
| --- | --- |
| Trạng thái | **UNDER REVIEW** |
| Phiên bản | **v2.1-draft - 01/09/2026** |
| Hệ thống | **zt-platform** - Zero Trust Access, Workload and Authorization Platform |
| Capability hiện có | **ap-authz** - Edge Gateway and Authorization Platform |
| Pilot | **market.order.read@v1** - authenticated, read-only, single-active-tenant |
| Chủ sở hữu tài liệu | Security Platform + Application Platform |
| Chủ sở hữu hệ thống target | Security Platform, Application Platform, IAM, Platform/SRE và Domain Owners theo RACI |
| Technical owner pilot | Lead Architect/Tech Lead kiêm Developer và Platform/SRE implementer |
| Control owners bắt buộc | IAM, Domain Owner, Product/Business, Security/Legal, Privacy/SecOps và SRE |
| Nguồn thiết kế | TDD ap-authz v1.9, bản refactor v2.0-draft, Zero Trust Networks 2nd Edition, Istio in Action, NIST SP 800-207/207A, CISA ZTMM v2, SPIFFE và Istio official documentation |
| Production approval | **CHƯA ĐƯỢC CẤP** |

## 0. Cách đọc và phạm vi quyết định

### 0.1 Đối tượng đọc

| Vai trò | Phần đọc tối thiểu |
| --- | --- |
| Developer pilot | 1-10, 16-17 |
| Domain Owner | 1-6, 9-12, 16-18 |
| Platform/SRE | 1-8, 11-18 |
| Security/IAM | 3-12, 15, 17-20 |
| Approver | 1-5, 14-20 |

Sau vòng đọc đầu tiên, người triển khai phải trả lời được:

1. Edge, device trust, workload trust, PEP, PDP/trust engine và domain chịu trách nhiệm gì?
2. Vì sao user token hoặc workload certificate hợp lệ vẫn có thể bị từ chối?
3. Actor, device, caller và application provenance khác nhau thế nào?
4. Resource fact nào phải lấy server-side và tại sao?
5. Khi nào ALLOW vẫn không được trả dữ liệu hoặc commit mutation?
6. Istio policy bảo vệ phần nào và không chứng minh được phần nào?
7. Revocation, cache, rollback và long-running session tương tác ra sao?
8. Evidence nào cho phép một action chuyển từ shadow sang production enforcement?

### 0.2 Trục trạng thái

Không dùng một nhãn để biểu diễn cả architecture decision và production readiness.

| Trục | Giá trị |
| --- | --- |
| Architecture decision | PROPOSED, ACCEPTED, SUPERSEDED, REJECTED |
| Readiness | POC_REQUIRED, EXTERNAL_APPROVAL, IMPLEMENTATION_READY, PRODUCTION_READY |
| Artefact delivery | NOT_STARTED, WAITING_INPUT, IN_PROGRESS, COMPLETE |

Tài liệu đang UNDER REVIEW nên mọi ADR trong tài liệu là PROPOSED cho đến khi có decision record được đúng authority phê duyệt.

### 0.3 Nguyên tắc sử dụng nguồn

- Sách và framework cung cấp mental model, threat pattern và implementation pattern; chúng không thay thế quyết định theo context doanh nghiệp.
- Istio là reference implementation cho workload traffic, mTLS, L4/L7 policy, telemetry và external authorization integration. Istio MUST NOT được coi là toàn bộ Zero Trust Architecture.
- Product version, feature status và deployment mode của Istio phải được khóa trong L3 ADR sau benchmark và security review.
- Quyết định business risk, audit durability, retention, privacy và production SLO phải do control owner tương ứng phê duyệt.

# 1. Executive Summary

## 1.1 Vấn đề

Các BFF và domain service hiện có xu hướng tự parse credential, map role/scope, kiểm tra tenant, tin identity header, tự quyết định service caller và ghi audit theo các schema khác nhau. Network location, namespace, IP hoặc việc request đã đi qua gateway đôi khi bị dùng như bằng chứng tin cậy.

Hệ quả:

- policy bị sao chép và drift;
- user identity, device posture, workload identity và application provenance bị trộn lẫn;
- một workload bị compromise có thể lateral movement qua các service cùng network;
- route, handler, consumer, job, response path hoặc egress path có thể bypass enforcement;
- resource ownership/tenant được kiểm tra từ dữ liệu client hoặc từ snapshot không còn nhất quán;
- cache, LKG và rollback có thể khôi phục quyền đã revoke;
- không nối được policy decision với response hoặc side effect cuối cùng;
- không có evidence định lượng để chuyển từ legacy sang target;
- service mesh dễ bị hiểu nhầm là security boundary đầy đủ trong khi vẫn có đường bypass, plaintext, egress hoặc application-path gap.

## 1.2 Target architecture

    User or machine client
      -> Device and session context
      -> Edge Gateway PEP
      -> Workload-authenticated transport
      -> Service Mesh L4/L7 guardrail
      -> Domain Service PEP
      -> Local or near-workload PDP + Trust Engine
      -> Domain transaction and response authorization
      -> Local durable audit relay

    Control plane
      -> identity, inventory and trust authorities
      -> policy review, test, build, sign and distribution
      -> application provenance and admission
      -> revocation, fleet status and reconciliation

Target tách bốn loại bằng chứng:

- **Actor evidence:** user, machine client, job hoặc service chịu ý nghĩa nghiệp vụ.
- **Device evidence:** thiết bị hoặc execution environment mà actor đang sử dụng.
- **Caller evidence:** workload thực hiện hop hiện tại.
- **Application evidence:** code/artifact/runtime instance nào đang chạy workload.

Authorization sử dụng công thức:

**Actor + Device + Caller + Application + Action + Resource + Context/Risk -> Decision + Obligations + Validity**

## 1.3 Quyết định đề nghị phê duyệt

| ID | Quyết định | Kết quả đề nghị |
| --- | --- | --- |
| DR-01 | Không tin network location; xác minh tại mỗi trust boundary | Đề nghị chấp thuận làm nguyên tắc L2 |
| DR-02 | Tách actor, device, caller và application provenance | Đề nghị chấp thuận làm canonical trust model |
| DR-03 | Edge mỏng, domain giữ resource truth/invariant/response | Đề nghị chấp thuận làm boundary |
| DR-04 | Centralize policy lifecycle, distribute evaluation/enforcement | Đề nghị chấp thuận làm target |
| DR-05 | SPIFFE-compatible workload identity và short-lived X.509-SVID | Đề nghị chấp thuận ở mức contract; authority cần conformance |
| DR-06 | Istio-compatible service mesh là reference carrier, không là application authorization authority duy nhất | Đề nghị chấp thuận làm guardrail |
| DR-07 | Mandatory platform guardrail + domain policy composition | Đề nghị chấp thuận ở mức contract |
| DR-08 | Signed immutable policy/application artifact và admission provenance | Đề nghị chấp thuận làm supply-chain guardrail |
| DR-09 | Device posture và contextual risk là registered facts, không là opaque header | Đề nghị chấp thuận; ngoài pilot phase đầu |
| DR-10 | Migration theo inventory -> observe -> shadow -> canary -> enforce -> retire | Đề nghị chấp thuận làm rollout method |
| DR-11 | Emergency revoke phải thắng positive cache, LKG và rollback | Đề nghị chấp thuận làm invariant; mechanism cần PoC |
| DR-12 | Pilot market.order.read chứng minh vertical slice trước khi mở rộng | Đề nghị chấp thuận làm implementation baseline |

## 1.4 Chưa đề nghị production approval

- Product và topology cuối cùng cho PDP/trust engine.
- Istio sidecar hay ambient mode, waypoint topology và multi-cluster model.
- SPIRE deployment; chỉ kích hoạt nếu identity platform hiện tại không đạt conformance.
- Continuous risk scoring hoặc machine-learning decisioning.
- Cross-tenant/deferred grant production.
- Deny-only security floor.
- Production audit mode, retention, quota và degraded window theo action.
- Numeric workload, capacity, cost, multi-region, RTO/RPO và SLO baseline.
- Device/UEM/EDR authority và device-attestation profile.
- Application admission/provenance platform.

## 1.5 Critical blockers

| Blocker | Điều kiện đóng |
| --- | --- |
| ZR-001 Identity/delegation chưa chứng minh | IAM and Delegation Profile + sender-binding/replay negative E2E |
| ZR-002 Workload identity authority chưa chứng minh | SPIFFE conformance + rotation/bundle/compromise drill |
| ZR-003 PEP/mesh bypass chưa loại bỏ | Route/handler/consumer/job/response/egress inventory + default-deny evidence |
| ZR-004 Revocation/cache/LKG chưa chứng minh | Emergency Revocation ADR + convergence/partition/rollback drill |
| ZR-005 Signer và supply-chain recovery chưa chứng minh | KMS governance + provenance + compromised-key exercise |
| ZR-006 Audit availability risk chưa được nhận | Signed action-level durability matrix + quota/full/noisy-tenant test |
| ZR-007 Resource snapshot consistency chưa chứng minh | Same-snapshot or version-revalidation E2E |
| ZR-008 Production ownership chưa có | Named system owner, on-call, incident authority and escalation |

# 2. Business Outcomes, Scope and Constraints

## 2.1 Business outcomes

- Mọi business action có canonical identity, resource, policy, enforcement và evidence.
- Một credential hoặc workload compromise có blast radius bị giới hạn theo least privilege.
- Không còn quyền implicit vì ở trong cluster, namespace, VPN hoặc private network.
- Thay đổi policy và mesh config có review, test, provenance, rollout và rollback an toàn.
- Domain team không phải tự phát minh security contract nhưng vẫn giữ business invariant.
- Security control có thể đo, audit và chứng minh theo action.
- Migration không yêu cầu big-bang và không xóa BFF còn business/channel value.

## 2.2 Capability scope

| Pillar/capability | Phạm vi L2 | Pilot |
| --- | --- | --- |
| User identity | Issuer, assurance, token class, session/delegation | Bắt buộc |
| Device trust | Device ID, management state, posture, attestation, freshness | Contract only |
| Workload identity | Attestation, SPIFFE ID, SVID, trust bundle, federation | Bắt buộc |
| Application trust | Source/build/distribution/runtime provenance, SBOM/admission | Minimum provenance |
| Edge access | TLS, hygiene, rate/size control, AuthN, coarse PEP | Bắt buộc |
| Service mesh | Strict mTLS, caller policy, L4/L7 traffic guardrail, telemetry | Bắt buộc reference PoC |
| Application authorization | Service PEP, PDP, facts, obligations, response filtering | Bắt buộc |
| Network/egress | Default-deny ingress and egress, expected-flow inventory | Pilot flow only |
| Data protection | Classification, minimization, encryption, resource/tenant truth | Pilot resource |
| Policy lifecycle | Review, test, sign, distribute, inventory, rollback | Bắt buộc |
| Audit/visibility | Decision/outcome correlation, metrics, traces, reconciliation | Bắt buộc |
| Automation/governance | Drift detection, gates, exceptions, RACI | Bắt buộc |

## 2.3 Out of scope

- Thay IAM/IdP, UEM/EDR hoặc SIEM trong một lần.
- Xây full entitlement-management UI/workflow.
- Đưa domain database hoặc full relationship graph vào Edge/PDP.
- Dùng trust score như bằng chứng duy nhất để ALLOW.
- Chọn mesh, PDP, risk engine hoặc workload CA production trước PoC.
- Xây active/active multi-region hoàn chỉnh trong L2.
- Bảo vệ volumetric DDoS chỉ bằng service mesh.
- Coi encryption hoặc mTLS là thay thế authorization.
- Coi Kubernetes namespace/service account label là root of trust nếu chưa attestation.

## 2.4 Assumptions and constraints

| ID | Giả định/ràng buộc | Trạng thái | Failure consequence |
| --- | --- | --- | --- |
| AC-01 | External route trong scope chỉ qua managed Edge | MUST | Direct ingress bị deny |
| AC-02 | Internal hop dùng authenticated workload identity | MUST | Plaintext/unknown principal bị deny |
| AC-03 | Domain là authority của resource/tenant/relationship | MUST | Không có fact authority thì không ALLOW |
| AC-04 | Policy evaluation deterministic, không gọi network tùy ý | MUST | Dynamic fact do PEP/provider cung cấp |
| AC-05 | Clocks được đồng bộ và giám sát | MUST | Credential/fact/cache/replay không an toàn |
| AC-06 | IAM hỗ trợ exact audience, key rotation và sender-bound delegation hoặc equivalent | WAITING_INPUT | OBO flow không production |
| AC-07 | Workload platform hỗ trợ stable identity, attestation và rotation | WAITING_INPUT | Mở SPIRE ADR |
| AC-08 | Remote audit sink không là sync dependency của mọi request | MUST | Dùng local relay/outbox |
| AC-09 | Device facts có authority/freshness khi policy dùng | WAITING_INPUT | Missing/stale device fact deny hoặc step-up |
| AC-10 | Istio policy không chứng minh handler/response/egress coverage | MUST | Cần mesh + application + network evidence |
| AC-11 | Workload/cost/multi-region chưa có baseline | WAITING_INPUT | Không production approve |

## 2.5 Success measures

| Measure | L2 target |
| --- | --- |
| Unknown execution path | 0 path được ALLOW |
| Cross-tenant allow không grant | 0 |
| Plaintext hoặc unknown workload caller | 0 sau strict gate |
| Privilege expansion trong migration | 0 unexplained legacy DENY -> new ALLOW |
| Decision/outcome reconciliation | Theo signed action contract; pilot target 100% trừ documented duplicates/retries |
| Policy/application artifact không provenance | 0 production activation |
| Critical revocation | Đạt approved convergence objective |
| Action availability/latency | Đạt class-specific SLO |

# 3. Zero Trust Principles and Canonical Model

## 3.1 Invariants

| ID | Invariant |
| --- | --- |
| ZT-01 | Network luôn được giả định hostile; locality không tạo trust. |
| ZT-02 | Mọi subject, device, workload, application và flow có identity/provenance phù hợp với risk. |
| ZT-03 | Authentication không đồng nghĩa authorization. |
| ZT-04 | Access được quyết định theo least privilege và resource/action cụ thể. |
| ZT-05 | Policy dùng facts có authority, provenance, freshness và bounded cardinality. |
| ZT-06 | Unknown identity, action, schema, fact source, policy layer hoặc obligation -> deny/indeterminate. |
| ZT-07 | Explicit deny thắng allow; một mandatory layer không được layer khác override. |
| ZT-08 | Resource truth, invariant, transaction và response authorization thuộc domain. |
| ZT-09 | Encryption và mutual authentication là bắt buộc cho sensitive flow; không thay thế PEP. |
| ZT-10 | Ingress và egress đều được inventory, kiểm soát và quan sát. |
| ZT-11 | Trust là có thời hạn; quyết định, credential, fact và session phải reevaluate/revoke được. |
| ZT-12 | Policy/config/application artifact immutable, signed, versioned và inventory được. |
| ZT-13 | Control plane được cô lập, least privilege và không là request hot-path dependency. |
| ZT-14 | Enforcement phải có evidence; policy tồn tại không chứng minh request đã bị chặn. |
| ZT-15 | Migration dùng telemetry và cohort; không dựa trên big-bang. |

## 3.2 Canonical decision input

| Thành phần | Ý nghĩa | Authority |
| --- | --- | --- |
| Actor | User, machine client, job hoặc service chịu ý nghĩa action | IAM/STS/approved system identity |
| Device | Thiết bị/execution environment gắn với actor | UEM/EDR/device attestation authority |
| Caller | Workload trực tiếp thực hiện hop | Verified mTLS/SPIFFE identity |
| Application | Artifact/runtime provenance của caller | Build, registry and admission authorities |
| Action | Động từ nghiệp vụ canonical có version | Domain + API Governance |
| Resource | Type, ID/token, tenant và version | Domain authority |
| Context | Environment, channel, time, network class, session | Trusted ingress/provider |
| Risk facts | Posture, anomaly, threat intelligence, historical signal | Registered provider/trust engine |

Không dùng một opaque trust score làm input duy nhất. Nếu có score, decision record phải giữ model/version, input classes, observed time, expiry và reason category đủ để audit mà không lộ dữ liệu nhạy cảm.

## 3.3 Decision output

| Thành phần | Yêu cầu |
| --- | --- |
| Effect | ALLOW, DENY hoặc INDETERMINATE |
| Reason | Stable reason code; không trả raw trace cho client |
| Policy state | Composition version + platform/domain digests |
| Revocation state | Generation/digest đã áp dụng |
| Obligations | Typed, versioned, capability-checked |
| Validity | valid_until bắt buộc cho cacheable decision |
| Evidence | decision_id, evaluated_at, freshness/cache markers |

PEP chỉ tiếp tục khi:

1. Effect là ALLOW.
2. Mọi mandatory policy layer hợp lệ.
3. Decision chưa hết valid_until.
4. Revocation state đạt freshness budget.
5. Mọi obligation đã được hiểu và có thể thi hành trước first byte hoặc side effect.
6. Resource snapshot được sử dụng vẫn là snapshot đã authorize.

## 3.4 Mapping với NIST logical components

| NIST logical component | Target component |
| --- | --- |
| Policy Engine | PDP + Trust Engine composition |
| Policy Administrator | Policy Rollout/Revocation Controller |
| Policy Enforcement Point | Edge PEP, mesh proxy, Service PEP, egress PEP, response layer |
| Data sources | IAM, device inventory, workload authority, application provenance, domain facts, threat/risk provider |

PEP có thể được tách thành client-side và resource-side enforcement. Resource-side PEP luôn là authority cuối cho resource-level authorization.

## 3.5 Functional requirements

| ID | Requirement | Acceptance evidence |
| --- | --- | --- |
| FR-01 | External credential validation pins issuer, audience, class, algorithm and time | Wrong-value negative E2E |
| FR-02 | Edge strips/canonicalizes untrusted headers and paths | Header/path bypass corpus |
| FR-03 | Actor, device, caller and application evidence remain separate | Contract tests |
| FR-04 | Workload caller uses attested SPIFFE-compatible identity and short-lived SVID | Identity conformance |
| FR-05 | Delegation is exact-audience, caller-bound, short-lived and replay-controlled | IAM profile + replay tests |
| FR-06 | Route/action vocabulary is registered and unknown maps to deny | Route inventory test |
| FR-07 | Resource/tenant/relationship facts are server-resolved | IDOR/cross-tenant tests |
| FR-08 | Read facts and response data use same snapshot/version | Concurrent-change E2E |
| FR-09 | Mutation rechecks invariant/version in authoritative transaction | TOCTOU mutation tests |
| FR-10 | Mandatory policy layers compose with deny-overrides and missing-layer fail close | Composition property tests |
| FR-11 | Decision contains policy/revocation state and valid_until | Contract/cache tests |
| FR-12 | Unknown/conflicting/failed obligation emits no data/side effect | Capability/leak tests |
| FR-13 | Every route, handler, consumer, job, response and egress path has PEP evidence | 100% coverage report |
| FR-14 | Workload traffic migrates to strict mTLS and least-privilege caller policy | Plaintext/wrong-caller tests |
| FR-15 | Protected egress is registry-controlled and cannot bypass gateway | Egress bypass tests |
| FR-16 | Policy/config/application artifacts are immutable, signed and provenance-verifiable | Corrupt/unsigned tests |
| FR-17 | Emergency revoke overrides cache/LKG/rollback/restart/active session | End-to-end revoke drill |
| FR-18 | Decision and final outcome are correlated and reconciled | Audit report |
| FR-19 | Async/deferred work reauthorizes or validates an approved grant | Consumer/grant tests |
| FR-20 | Device/application trust facts have authority, version and freshness when enabled | Provider/admission tests |
| FR-21 | Control-plane outage does not create silent bypass | Failure/chaos tests |
| FR-22 | Migration uses four-way shadow comparison and measurable cohort gates | Gate evidence |
| FR-23 | Privacy-sensitive identity/risk data is minimized and not used as metric cardinality | Privacy review |
| FR-24 | Break-glass is strong-authenticated, scoped, time-bound, alerted and reviewed | Exercise report |

# 4. Current State and Gap Assessment

## 4.1 Existing implementation

Workspace hiện có scaffold:

- authz-contract: Java records cho actor, caller, resource, facts và decision;
- authz-pep-spring: vị trí dành cho reusable Service PEP;
- market-pilot-service: vị trí dành cho pilot domain;
- policies: vị trí cho Rego policy/test;
- deployment/local: vị trí cho runtime local.

Chưa có policy implementation, PDP runtime, service integration, mesh manifest, audit relay hoặc production evidence.

## 4.2 Contract gaps phải sửa trước integration

| Gap | Ảnh hưởng | Required change |
| --- | --- | --- |
| Decision chưa có valid_until | Positive cache có thể sống quá credential/fact/grant | Bổ sung validity và derived TTL |
| Caller chưa mang delegation mode/audience/ID/expiry | Không chứng minh actor-caller binding | Mở rộng caller/delegation contract |
| Actor chưa có credential expiry/entitlement version | Không bound freshness | Bổ sung metadata |
| Không có device/application evidence | Không support device/app trust policy | Thêm optional typed blocks; missing behavior theo action |
| Correlation chỉ có transactionId | Retry/async/business operation khó phân biệt | Thêm attempt_id, business_operation_id, parent_transaction_id |
| Resource fact xuất hiện ở hai vị trí | Canonical fingerprint mơ hồ | Chọn một canonical fact namespace |
| Action chỉ là string | Semantics/version không tách | Dùng action ID + version |
| Decision chỉ có một policy revision | Không chứng minh composition | Bổ sung platform/domain/model digests |
| Không có enforcement outcome schema | Không reconcile final effect | Tạo outcome contract |

## 4.3 Architectural gap so với full Zero Trust

TDD ap-authz v1.9 bao phủ tốt Edge, workload identity, PEP/PDP, audit và migration. Bản này bổ sung:

- device identity/posture và context-aware agent;
- application source/build/distribution/runtime provenance;
- bookended network and egress control;
- explicit trust engine boundary;
- continuous validity và session invalidation;
- service mesh limitations, sidecar/ambient choice và network-policy defense in depth;
- same-snapshot read authorization;
- measurable canary gates;
- end-to-end requirement/risk/test/evidence traceability.

# 5. Threat Model

## 5.1 Protected assets

- User, device, workload và application identities.
- Delegation/grant artifacts.
- Domain data và business side effects.
- Policy, risk model, revocation state và signing keys.
- Workload CA/trust bundles and attestation data.
- Audit evidence and privacy-sensitive metadata.
- Build pipeline, registry, SBOM/provenance and admission control.
- Mesh/control-plane config and rollout authority.

## 5.2 Adversaries

| Adversary | Capability giả định |
| --- | --- |
| External opportunistic | Scan, brute force, malformed traffic, stolen bearer token |
| Targeted attacker | Phishing, replay, confused deputy, supply-chain attack |
| Compromised workload | Valid in-mesh identity, arbitrary outbound request, header spoof |
| Compromised device | Valid user session nhưng posture/runtime không đáng tin |
| Insider | Hợp lệ trong một tenant/domain, cố lateral/cross-tenant access |
| Privileged insider | Có policy/config/deployment access |
| Dependency compromise | IAM, registry, build, CA, fact provider hoặc audit bị lỗi/chiếm quyền |
| Availability attacker | Flood Edge/PDP/fact/audit/control plane hoặc tạo expensive policy input |

State-level compromise toàn bộ hardware/root authorities nằm ngoài khả năng bảo đảm tuyệt đối, nhưng thiết kế phải giảm blast radius, phát hiện và hỗ trợ recovery.

## 5.3 Trust boundaries

| Boundary | Không được tin | Control bắt buộc |
| --- | --- | --- |
| Internet -> Edge | Header, path normalization, credential, device claim | TLS, hygiene, AuthN, size/rate, device/session verification |
| Edge -> workload | Actor header, caller location | mTLS caller identity + delegated artifact validation |
| Workload -> workload | Namespace/IP/service DNS | Strict mutual identity + caller/destination policy |
| Proxy -> application | Proxy metadata nếu app có đường bypass | Loopback/bind policy, network policy, application PEP |
| PEP -> PDP | Caller-supplied identity, unbounded input | Authenticated channel, schema, size/depth/deadline |
| PEP -> fact authority | Client value, stale cache | Registered provider, provenance, freshness, version |
| Runtime -> egress | Declared destination alone | Egress allowlist + gateway + network enforcement |
| Control plane -> data plane | Unsigned/stale config | Signature, capability, atomic activation, fleet status |
| Build -> runtime | Mutable tag, unsigned artifact | Digest, provenance, SBOM, admission and runtime identity |
| Audit producer -> sink | PII, forged outcome, queue flood | Schema, integrity, quota, durability and reconciliation |

## 5.4 Threat/control matrix

| Threat | Example | Required controls |
| --- | --- | --- |
| Forged actor | Spoofed identity header | Strip; verify credential/delegation at Service PEP |
| Forged caller | Fake namespace/IP/header | Verified SPIFFE identity; strict mTLS |
| Confused deputy | Token/grant dùng sai callee | Exact audience, immediate caller binding, delegation depth |
| Replay | Reuse delegated or one-time grant | Sender binding, nonce/jti, TTL, replay store where required |
| IDOR/cross-tenant | Client đổi order/tenant | Server-side resource resolution + explicit grant |
| TOCTOU | Owner/tenant đổi sau ALLOW | Same snapshot or compare-and-revalidate version |
| PEP bypass | Direct pod/handler/consumer path | Default-deny inventory + mesh/network/application coverage |
| Path confusion | Proxy/backend parse path khác | Canonicalization contract + normalization tests |
| Stale permission | Cache/LKG/session sống lâu | valid_until, freshness budget, revoke and terminate |
| Policy tampering | Registry/build/signer compromise | Review, SoD, provenance, KMS, immutable digest |
| Workload CA compromise | Attacker mint SVID | Short-lived cert, root separation, bundle revoke/recovery |
| Supply-chain compromise | Malicious build/dependency | Signed source/build, SBOM, scanning, admission, runtime monitor |
| Egress exfiltration | Compromised pod bypass sidecar | Egress gateway + NetworkPolicy/firewall + DNS policy |
| Mesh config ignored | Invalid selector/target | Static validation, analyze, fleet status, positive/negative test |
| Audit availability attack | Tenant fills spool | Quota isolation, reserve, high-water, bounded degraded mode |
| Control-plane DoS | xDS/policy churn | Isolation, batching, rate control, LKG, not on request path |
| Data leakage by obligation | Mask/filter unsupported | Capability handshake; no first byte before enforcement |

# 6. Architecture

## 6.1 Context architecture

    +--------------------------- CONTROL PLANE ----------------------------+
    | IAM/STS  Device/UEM  Workload CA  Build/Registry  Policy Repository |
    |    \         |           |             |               /            |
    |     \--------+-----------+-------------+--------------/             |
    |              Trust/Policy/Revocation Controllers                    |
    |              Sign, distribute, inventory, reconcile                 |
    +-------------------------------|--------------------------------------+
                                    |
    +---------------------------- DATA PLANE ------------------------------+
    | Client -> Edge PEP -> mTLS/Mesh -> Service PEP -> PDP/Trust Engine  |
    |                                  |                 |                 |
    |                                  +-> Domain facts  |                 |
    |                                  +-> Domain transaction/response     |
    |                                  +-> Egress PEP                       |
    |                                  +-> Local audit relay                |
    +---------------------------------------------------------------------+

Control plane outage không được trực tiếp làm mọi request fail trong freshness budget. Data plane không được tự author/publish policy.

## 6.2 Component responsibilities

| Component | Phải làm | Không được làm |
| --- | --- | --- |
| Edge Gateway | External AuthN, path/header hygiene, device/session context, rate/size, route/action map, coarse PEP | Đọc domain DB hoặc quyết định resource ownership |
| Device Trust Adapter | Xác minh device ID, managed state, posture, attestation và freshness | Tin client-provided device header |
| IAM/STS | Actor identity, assurance, delegation, entitlement/revoke | Quyết định domain resource truth |
| Workload Identity Authority | Node/workload attestation, SPIFFE ID, SVID/bundle lifecycle | Cấp actor entitlement |
| Service Mesh | mTLS, caller/destination L4 policy, selected L7 guardrail, telemetry | Thay thế Service PEP/business authorization |
| Service PEP | Verify delegation, resolve resource/facts, call PDP, enforce validity/obligations, emit outcome | Tin client facts hoặc silent bypass |
| PDP | Deterministic policy composition | Gọi network/domain DB tùy ý khi evaluate |
| Trust Engine | Chuẩn hóa risk/device/session signals có provenance/version | Tự tạo business permission hoặc opaque allow |
| Domain Service | Resource truth, invariant, transaction, response and same-snapshot behavior | Tin ALLOW như lệnh bắt buộc commit |
| Egress PEP | Destination allowlist, TLS/origin policy, exfiltration guard, telemetry | Chỉ dựa vào service-mesh registry mode |
| Policy Control Plane | Review, test, build, sign, distribute, inventory, rollback | Nằm trên request hot path |
| Application Trust Plane | Source/build provenance, SBOM, registry/admission, runtime inventory | Cấp business authorization |
| Audit Platform | Durable delivery, retention, access, integrity, reconciliation | Tham gia business decision |

## 6.3 Enforcement layers and composition

| Layer | Mục đích | Failure posture |
| --- | --- | --- |
| Device/session gate | Giảm rủi ro trước Edge | Missing/stale theo action: deny, step-up hoặc restricted |
| Edge PEP | External boundary và coarse route guardrail | Default deny |
| Workload/mesh PEP | Mutual identity và expected flow | Strict deny sau migration gate |
| Service PEP | Resource-level authorization | Fail close |
| Domain invariant | Business correctness trong transaction | Không commit |
| Response PEP | Field/row/stream filtering | Không phát first byte nếu chưa enforce |
| Egress PEP | Outbound destination/data flow | Default deny cho protected workload |

ALLOW của layer ngoài không bỏ qua layer trong. DENY ở bất kỳ mandatory layer nào là final DENY. Missing, stale, incompatible hoặc unavailable mandatory layer là INDETERMINATE và PEP fail close, trừ exception C2 đã phê duyệt.

## 6.4 Domain-driven boundaries

| Bounded context | Sở hữu | Không sở hữu |
| --- | --- | --- |
| IAM | Actor credential, assurance, delegation, entitlement lifecycle | Device/workload/resource truth |
| Device Trust | Device registration, posture, attestation | User entitlement/business resource |
| Workload Identity | Workload attestation, SPIFFE ID, SVID/bundle | Actor/business permission |
| Authorization | Canonical contract, policy lifecycle, shared guardrail, decision | Full domain model/transaction |
| Application Trust | Source/build/runtime provenance and admission | Business action permission |
| Domain | Resource, tenant relationship, invariant, transaction, response | Shared signing/distribution |
| Audit/SecOps | Evidence schema, retention, investigation | Business decision |

Anti-Corruption Layer tại domain map dữ liệu nội bộ sang minimal canonical facts. Policy input không được trở thành bản sao central của toàn bộ domain model.

## 6.5 PEP coverage model

| Execution path | PEP/evidence |
| --- | --- |
| External HTTP/gRPC | Edge route registry + default-deny test |
| Domain handler | Handler/action registry + framework conformance |
| East-west call | Mesh identity/destination policy + flow inventory |
| Async consumer | Producer/schema/action registry + consumer PEP |
| Scheduled/admin job | Dedicated principal/purpose/action/audit |
| Response/stream | Typed obligation capability + leak/partial-response test |
| External egress | Egress registry/gateway/network policy + bypass test |
| Non-mesh/legacy | Explicit exception, compensating proxy and expiry |

100% coverage nghĩa là inventory và runtime evidence cùng chứng minh. Network policy hoặc mesh policy riêng lẻ không đủ.

## 6.6 Integration catalogue

| ID | Integration | Source -> Destination | Mode | Contract/data |
| --- | --- | --- | --- | --- |
| INT-01 | External AuthN | Edge -> IAM/IdP | HTTPS/cache | Token metadata/JWKS |
| INT-02 | Device posture | Edge/Trust Engine -> Device Authority | Bounded sync/cache/event | Device facts/freshness |
| INT-03 | Delegated call | Caller -> STS -> Callee | HTTPS + workload AuthN | Actor/caller/audience/scope/TTL |
| INT-04 | Workload identity | Workload -> Identity Authority | Stream/local API | SVID/bundle/rotation |
| INT-05 | Mesh authentication/policy | Mesh control -> data plane | Async config | Trust/authz/traffic config |
| INT-06 | Edge coarse authorization | Edge PEP -> PDP | Local/near sync | Contract subset |
| INT-07 | Resource authorization | Service PEP -> PDP | Local/near sync | Full canonical request/decision |
| INT-08 | Fact lookup | Service PEP -> Domain/provider | Bounded sync | Typed value/source/version/freshness |
| INT-09 | Policy distribution | Control plane -> registry/cache/PDP | Async | Signed immutable manifest/artifact |
| INT-10 | Fleet status | Proxy/PDP -> controller | Async | Desired/received/active/capability |
| INT-11 | Application provenance | Build/registry/admission -> runtime/trust | Async/admission | Digest/SBOM/provenance/status |
| INT-12 | Revocation | IAM/Device/App/SecOps -> PEP/PDP | Ordered async | Scope/generation/digest/TTL |
| INT-13 | Audit | PEP/domain -> local relay -> collector | Durable async | Decision/outcome/checkpoint |
| INT-14 | Protected egress | Workload -> egress PEP/gateway -> external | Sync | Destination/purpose/TLS/audit |

Mỗi integration có L3 schema, authentication, authorization, timeout, retry, rate, size, privacy, versioning and error contract.

## 6.7 Technology capability and justification

| Capability | L2 selection | Rationale/status |
| --- | --- | --- |
| Edge | Managed Envoy-compatible gateway or equivalent | Required capabilities; product inventory pending |
| Workload identity | SPIFFE-compatible X.509-SVID | Portable logical identity; authority conformance pending |
| Workload authority | Existing platform issuer if conformant; SPIRE fallback | Avoid dual authority; conditional ADR |
| Service mesh | Istio-compatible reference PoC | mTLS/policy/telemetry/ext-authz; mode/version pending |
| PDP engine | OPA/Cedar or equivalent candidates | Determinism/testability; benchmark required |
| PDP topology | Embedded, sidecar, node-local, waypoint-near or regional | Latency/isolation/operations benchmark |
| Policy/artifact store | Version control + immutable OCI/object registry | Traceability and digest promotion |
| Signing | KMS/HSM-backed purpose-specific keys | Integrity and recovery governance |
| Application provenance | Signed build attestation + SBOM + admission | Supply-chain trust; L3 platform choice |
| Audit | Transactional outbox/local durable relay + collector | Remote sink outside hot path |
| Telemetry | OpenTelemetry-compatible + mesh telemetry | End-to-end correlation |

Programming language, framework and code layout are not L2 decisions. Current Java/Spring/Rego scaffold is a pilot implementation choice, not an enterprise constraint.

# 7. Identity, Device, Workload and Application Trust

## 7.1 Human and machine actor identity

Actor credential profile MUST định nghĩa:

| Field/control | Yêu cầu |
| --- | --- |
| Issuer and token class | Exact allowlist; không nhận token class khác |
| Audience | Exact callee/resource audience |
| Signature algorithm | Pin approved algorithms; reject algorithm downgrade |
| Lifetime | issued_at, not_before, expires_at và maximum age theo action |
| Subject/client | Stable non-reassignable identifier |
| Assurance | MFA/authentication context có authority |
| Tenant | Active tenant là actor context, không thay resource tenant |
| Entitlement | Version/freshness/revocation reference |
| Session | Session ID/token ID để revoke và correlate |
| Sender constraint | mTLS confirmation, DPoP hoặc approved equivalent khi risk yêu cầu |

Edge xác thực external credential nhưng Service PEP vẫn phải xác thực delegated artifact dành cho chính callee. Unsigned actor/role/scope header không có authority.

## 7.2 Device identity and posture

Device trust tách khỏi actor identity. Một user hợp lệ trên thiết bị không đạt posture không mặc nhiên có cùng quyền như trên managed device.

Device evidence tối thiểu:

| Nhóm | Facts |
| --- | --- |
| Identity | device_id/token, issuer, ownership class |
| Management | managed/enrolled state, UEM authority, last check-in |
| Integrity | secure boot/attestation state, hardware-backed key nếu có |
| Hygiene | OS/version, patch age, EDR state, disk encryption, jailbreak/root signal |
| Context | device class, network/location risk category, observed_at, expires_at |
| Lifecycle | registration version, revoke/quarantine state |

Quy tắc:

- Device claim từ client chỉ là hint cho đến khi authority xác minh.
- Raw hardware identifiers không truyền rộng; dùng tokenized device reference.
- Posture fact có short validity và provider provenance.
- Device missing/stale không tự động DENY cho mọi public action; action pack quyết định deny, step-up hoặc restricted obligation.
- High-risk action MUST yêu cầu managed device hoặc approved compensating control.
- Device quarantine/revoke phải có objective và propagate tới active session/decision cache.

## 7.3 Context-aware access tuple

Một request có thể gắn Actor + Device thành access agent, nhưng hai identity vẫn được lưu riêng để tránh conflation.

Ví dụ policy:

- user thuộc Market tenant và device managed -> read full permitted fields;
- user hợp lệ nhưng device unmanaged -> chỉ metadata đã mask;
- privileged mutation -> MFA gần đây + managed/attested device + approved caller;
- service system action -> không có user/device giả; dùng system actor và application/workload provenance.

Context/risk chỉ giảm hoặc điều kiện hóa quyền đã được business policy cấp. Risk score thấp không tạo permission mới.

## 7.4 Workload identity profile

### 7.4.1 SPIFFE contract

| Control | Baseline |
| --- | --- |
| Trust domain | Tách theo environment và security administrative boundary |
| SPIFFE ID | Stable logical workload identity; không chứa pod/node/actor/tenant |
| SVID | Short-lived X.509-SVID mặc định |
| JWT-SVID | Chỉ khi X.509 mTLS không khả thi; exact audience + replay threat model |
| Attestation | Node và workload phải được authority xác minh trước issuance |
| Key | Không phân phối như static application secret; rotate tự động |
| Bundle | Versioned rotation/overlap/revoke; bundle gắn đúng trust domain |
| Federation | Explicit flow inventory; không transitive trust mặc định |

Ví dụ naming logic:

- spiffe://prod.vhm/platform/edge-gateway
- spiffe://prod.vhm/market/order-api
- spiffe://prod.vhm/security/local-pdp

Tên thật được khóa trong Workload Identity Profile. Không encode role, tenant hoặc instance ID vào SPIFFE ID.

### 7.4.2 Authority adoption gate

Platform issuer hiện tại được dùng nếu chứng minh:

1. Stable SPIFFE-compatible identity và URI SAN semantics.
2. Node/workload attestation chống identity spoof.
3. Short-lived X.509 identity và rotation không restart.
4. Trust-bundle rotation, overlap, rollback và compromise recovery.
5. Multi-AZ issuance/rotation SLO.
6. Federation pin đúng trust domain/destination.
7. Private key isolation và no-static-secret behavior.

Chỉ cần một mandatory control không đạt thì mở SPIRE ADR trong identity scope bị ảnh hưởng. Không chạy hai issuer production song song cho cùng workload; migration chỉ cho bounded trust-bundle overlap.

## 7.5 Application and software supply-chain trust

Workload identity chứng minh instance là workload nào; application provenance chứng minh code/artifact nào được phép chạy dưới identity đó.

| Stage | Required evidence |
| --- | --- |
| Source | Protected repository, branch rule, review, signed change/tag theo policy |
| Dependencies | Lock/version, vulnerability/license scan, SBOM |
| Build | Isolated runner, pinned toolchain, reproducible/hermetic target khi khả thi |
| Artifact | Immutable digest, signature, build provenance |
| Distribution | Approved registry, TLS, digest verification, promotion không rebuild |
| Admission | Environment/purpose policy, signer/provenance/SBOM verification |
| Runtime | Image digest, workload identity mapping, least privilege, drift/runtime monitoring |

Mutable tag không là production identity. Build artifact được promote giữa environment bằng immutable digest; không rebuild cùng version cho mỗi environment.

Application evidence đưa vào authorization chỉ khi action thực sự cần. Ví dụ C0 có thể yêu cầu caller đang chạy approved digest/provenance cohort; standard read có thể chỉ dùng admission status.

## 7.6 Delegation

### 7.6.1 Modes

| Mode | Semantics |
| --- | --- |
| on_behalf_of | Caller hành động thay actor; STS bind actor, immediate caller, exact audience, scope, TTL |
| system | Dedicated system actor/purpose; không gắn user giả |
| approved_deferred_grant | Side effect tương lai dựa trên signed grant có scope/expiry/revoke |

### 7.6.2 Hop rules

- Workload mTLS chỉ xác minh immediate caller, không xác minh actor.
- Callee Service PEP MUST validate delegated artifact: issuer, signature, audience, authorized party/caller binding, not-before, expiry, token/grant ID và replay rule.
- Transitive delegation mặc định bị cấm. Nếu A -> B -> C, STS phải mint artifact mới cho audience C và giữ original actor/purpose/delegation-chain reference.
- Maximum delegation depth và allowed exchangers được khai báo trong IAM profile.
- Raw upstream user token không được chuyển xuyên nhiều service nếu exact audience không đúng.

### 7.6.3 Explicit grant

Grant tối thiểu gồm:

| Nhóm | Fields |
| --- | --- |
| Identity | grant_id, issuer, schema/version, signature/key ID |
| Subject | actor, immediate caller constraints, exact callee/audience |
| Permission | action/version, resource type/ID/tenant scope, purpose |
| Time | issued_at, not_before, expires_at |
| Usage | single-use hoặc multi-use, nonce/jti, idempotency/business operation binding |
| Governance | reason, approver, ticket/evidence |
| Revocation | generation/version and authority |

Broad admin role không được suy ra cross-tenant grant.

## 7.7 Trust Engine

Trust Engine chuẩn hóa contextual signals; không thay PDP và không ALLOW trực tiếp.

Provider registry cho mỗi signal:

- owner và authenticated provider identity;
- schema/type/range;
- source of truth;
- observed_at, expires_at và maximum staleness;
- model/rule version nếu derived;
- data classification/residency;
- failure semantics;
- explanation category và bias/quality review nếu dùng ML.

Opaque trust score không được dùng cho C0. Nếu derived score dùng cho C1/C2, policy phải có guardrail, fallback, drift monitoring và human-review path.

## 7.8 Continuous access and invalidation

- valid_until bound mọi cacheable ALLOW.
- Long-running stream/session phải có maximum continuous authorization interval hoặc revocation subscription.
- Device quarantine, actor disable, grant revoke, application quarantine và workload identity revoke đều có mapped invalidation behavior.
- Session termination không chỉ xóa Edge cookie; downstream cached decisions và active stream phải dừng trong approved objective.
- Reauthentication/step-up tạo attempt mới nhưng giữ business operation correlation.

# 8. Network and Service Mesh Architecture

## 8.1 Expected-flow model

Mọi protected flow có registry row:

| Field | Nội dung |
| --- | --- |
| Source | Workload/device principal hoặc class |
| Destination | Workload/service/resource |
| Direction | Ingress, east-west, egress |
| Protocol | HTTP/gRPC/TCP/event |
| Port/SNI/host | Canonical destination |
| Action class | C0/C1/C2 |
| Authentication | Credential/SVID requirement |
| Authorization | Mesh policy + Service PEP action |
| Encryption | TLS/mTLS profile |
| Owner/expiry | Named owner and review date |

Discovery từ telemetry chỉ là input; observed traffic không tự động trở thành allow policy.

## 8.2 Bookended enforcement

- Destination ingress MUST default deny và chỉ allow expected caller/action.
- Protected workload egress MUST default deny theo destination registry.
- NetworkPolicy/firewall MUST ngăn workload bypass proxy/egress gateway.
- DNS resolution không là authorization; destination identity được xác minh ở authenticated connection.
- East-west traffic được encrypt/authenticate end-to-end giữa enforcement boundaries; TLS termination trung gian phải được threat-model.
- Direct pod IP, hostNetwork, privileged pod, raw socket và alternate port là bypass cases bắt buộc test.

## 8.3 Istio reference profile

Istio được dùng trong PoC để chứng minh carrier capability:

| Capability | Reference mechanism | Guardrail |
| --- | --- | --- |
| Workload mTLS | PeerAuthentication/SDS-issued certificate hoặc equivalent | Chuyển PERMISSIVE -> STRICT theo migration gate |
| Caller authorization | AuthorizationPolicy principal/service identity | Default deny; không authorize chỉ theo namespace |
| External JWT | RequestAuthentication tại approved boundary | Validate issuer/audience; Service delegation vẫn bắt buộc |
| Domain authorization hook | CUSTOM/ext_authz đến local/near PEP/PDP adapter | Bounded headers/input, timeout, fail close |
| Traffic rollout | Route/cohort/mirroring | Không dùng routing thay authorization |
| Resilience | Timeout, retry, circuit breaker, outlier detection | Không retry unsafe mutation nếu thiếu idempotency |
| Telemetry | Access log, metrics, trace metadata | Correlate với application outcome |
| Egress | Egress gateway + network enforcement | Registry-only mode không là security boundary |

## 8.4 Sidecar versus ambient

Deployment mode là ADR-007, chưa baseline production.

| Tiêu chí | Sidecar | Ambient |
| --- | --- | --- |
| L4 identity/mTLS | Per-workload proxy | Node ztunnel |
| L7 policy | Sidecar Envoy | Waypoint bắt buộc |
| Resource overhead | Per pod | Shared L4 + selected waypoint |
| Isolation/blast radius | Per workload proxy | Shared components cần threat review |
| Extensibility | EnvoyFilter/Wasm tùy support | Waypoint capability/targetRefs; EnvoyFilter không mặc định tương thích |
| Migration | Injection/restart lifecycle | Enrollment/waypoint routing lifecycle |

Nếu ambient được chọn:

- L7 action/path/header policy và CUSTOM external authorization MUST attach đúng waypoint/Service bằng supported target reference.
- ztunnel chỉ thi hành L4; không được ghi L7 coverage nếu không có waypoint.
- Policy không tương thích phải fail safe và là migration blocker.
- Waypoint bypass, service enrollment, direct pod traffic và policy attachment được test.

## 8.5 Strict mTLS migration

1. Inventory mesh/non-mesh callers.
2. Enable telemetry và phát hiện plaintext.
3. PERMISSIVE chỉ trong bounded migration scope.
4. Migrate caller identity và negative tests.
5. Apply namespace/workload default-deny authorization.
6. Canary STRICT theo workload.
7. Chặn plaintext bằng network evidence.
8. Xóa exception và identity/port cũ.

PERMISSIVE không được coi là production-ready state cho protected action nếu không có exception owner, expiry và compensating control.

## 8.6 Mesh policy safety

- Default-deny trước allowlist.
- Dùng positive matching cho ALLOW.
- RequestAuthentication chỉ xác minh credential khi có mặt; policy bắt buộc phải yêu cầu authenticated principal/claim cho protected route.
- Request path/method/authority canonicalization giữa proxy và backend phải thống nhất.
- HTTP-only condition trên TCP policy có thể có missing-attribute semantics khác dự kiến; protocol/port phải được scope và test để không tạo unintended allow/deny.
- Config phải qua schema validation, static analysis, dry-run/shadow và positive/negative test.
- Desired config và active config ở từng proxy được quan sát riêng.
- Server-first TCP protocol có thể phát byte trước authorization; sensitive protocol như vậy cần alternate enforcement hoặc bị loại khỏi mesh authorization scope.
- AuthorizationPolicy inbound không tự bảo vệ outbound; egress dùng control riêng.

## 8.7 External authorization channel

| Control | Yêu cầu |
| --- | --- |
| Placement | Local sidecar, waypoint-near hoặc node-local ưu tiên sau benchmark |
| Protocol | Authenticated gRPC/HTTP ext-authz-compatible contract |
| Input | Header/body allowlist, size/depth limit, normalized path |
| Deadline | Nhỏ hơn action deadline; timeout -> INDETERMINATE/fail close |
| Output | Effect, reason, decision ID, policy/revocation state, validity, obligations |
| Failure | Không silent bypass/failure-mode-allow cho C0/C1 |
| Availability | Bulkhead, bounded concurrency, circuit breaker |
| Privacy | Không gửi raw token/full body nếu không cần |

Mesh proxy chỉ enforce được obligations thuộc capability của proxy. Resource/row/field/business obligations vẫn do Service PEP/domain thi hành.

## 8.8 Egress

Protected workload chỉ được gọi external destination khi registry có:

- owner, purpose và data classification;
- canonical host/service identity và TLS requirement;
- destination/port/protocol;
- caller/action allowlist;
- request data minimization/DLP rule nếu cần;
- timeout/retry/idempotency;
- audit and incident owner;
- expiry/review.

Egress gateway không tự ngăn sidecar bypass. NetworkPolicy, firewall, route/NAT policy hoặc equivalent MUST buộc traffic đi qua approved egress boundary.

# 9. Authorization and Evidence Contracts

## 9.1 Request contract v1

| Block | Mandatory fields |
| --- | --- |
| Envelope | schema_version, authorization_transaction_id, attempt_id, request_time, deadline |
| Actor | type, issuer, subject, active_tenant_id, assurance, entitlement_version, credential_expires_at, session/token ID |
| Device | device token/ID, issuer, class, posture facts, observed_at, expires_at; optional only by action contract |
| Caller | verified SPIFFE ID, trust domain, delegation mode, exact audience, delegation/grant ID, expires_at |
| Application | artifact digest, provenance/admission status, runtime cohort, observed_at |
| Action | canonical action ID and version |
| Resource | type, ID/token, tenant, version/snapshot |
| Facts | name, typed value, source, source version, observed_at, expires_at |
| Context | environment, channel, network class, risk/model version, parent transaction |

Rules:

- PEP tạo request từ verified evidence; client không gửi canonical contract trực tiếp đến PDP.
- Contract không mang raw token, full request body hoặc full domain record.
- Một fact chỉ tồn tại một lần trong canonical namespace.
- Missing mandatory block/fact -> INDETERMINATE; behavior theo action failure posture.
- Canonical serialization/fingerprint algorithm được version hóa.

## 9.2 Decision contract v1

| Field | Semantics |
| --- | --- |
| schema_version | Contract version |
| decision_id | Unique evaluation ID |
| effect | ALLOW, DENY, INDETERMINATE |
| reason_codes | Stable ordered reason identifiers |
| policy_state | composition version + platform/domain digests |
| trust_state | risk model/rule version and signal freshness summary |
| revocation_state | generation/digest |
| obligations | Typed, versioned parameters |
| evaluated_at | Evaluation time |
| valid_until | Hard upper bound for reuse |
| cache_metadata | Hit/miss, input fingerprint version |

valid_until được tính không muộn hơn minimum của:

- actor credential/session expiry;
- delegation/grant expiry;
- device/application/fact expiry;
- entitlement freshness;
- policy LKG maximum age;
- revocation freshness budget;
- action-specific maximum decision TTL.

PEP MUST kiểm tra valid_until bằng monotonic elapsed-time mapping để wall-clock rollback không kéo dài ALLOW.

## 9.3 Policy composition

| Layer state | Final result |
| --- | --- |
| Bất kỳ mandatory layer DENY | DENY |
| Mandatory layer missing/incompatible/unavailable/indeterminate | INDETERMINATE |
| Tất cả mandatory layer ALLOW | Candidate ALLOW |
| Candidate ALLOW nhưng obligation unsupported/conflict/fails | No response/side effect |
| Domain invariant fails | NOT_EXECUTED/domain error |

Mandatory layers cho pilot:

1. Platform identity/schema/default-deny guardrail.
2. Workload caller/destination guardrail.
3. Market domain policy.
4. Revocation state.

Device and application trust layer chỉ mandatory khi Action Pack bật requirement.

## 9.4 Obligation contract

| Obligation | Enforcer | Precondition |
| --- | --- | --- |
| mask_fields | Response PEP | Capability/version known before serialization |
| filter_rows | Domain query/response PEP | Áp dụng trước pagination/count leak |
| limit_result | Domain/response | Bound trước materialization nếu cần |
| require_step_up | Edge/session | Không thực hiện protected action trước reauth |
| require_fresh_fact | Service PEP | Refresh từ registered authority |
| constrain_mutation | Domain transaction | Recheck invariant in transaction |
| force_reauth_interval | Session/stream PEP | Timer/revocation-aware termination |

Unknown, conflicting hoặc failed obligation là fail close. Streaming response không gửi header/body đầu tiên trước capability and policy preflight; long stream có reauthorization rule.

## 9.5 Resource consistency contract

Đối với read:

- resource facts và response data MUST đến từ cùng snapshot/version; hoặc
- PEP MUST compare-and-revalidate version ngay trước serialization/first byte.

Đối với mutation:

- read/lock resource version trong transaction;
- evaluate trên transaction facts;
- recheck version/invariant;
- commit mutation + audit intent atomically.

Nếu version đổi: không response nhạy cảm/side effect; trả RESOURCE_CONFLICT hoặc retry an toàn theo action.

## 9.6 Error contract

| Error | Client behavior | SLO classification |
| --- | --- | --- |
| AUTHENTICATION_FAILED | 401/equivalent | Client/security reject |
| AUTHORIZATION_DENIED | 403 hoặc concealed 404 | Business/security deny |
| INPUT_INVALID | Controlled 400 | Client unless trusted mapper defect |
| POLICY_UNAVAILABLE | 503; fail close | Platform error |
| REVOCATION_STALE | 503/deny by action | Platform/security error |
| FACT_UNAVAILABLE | 503 hoặc deny by action | Dependency error |
| DEVICE_POSTURE_REQUIRED | Step-up/remediation response | Business/security control |
| OBLIGATION_UNSUPPORTED | No data/side effect | Platform error |
| RESOURCE_CONFLICT | 409/equivalent | Domain conflict |
| RATE_LIMITED | 429/equivalent | Theo quota owner |

Không map platform unavailable thành policy deny trong internal telemetry.

## 9.7 Correlation identifiers

| ID | Lifetime/authority |
| --- | --- |
| authorization_transaction_id | Một logical ingress execution; trusted ingress tạo/ghi đè |
| attempt_id | Mỗi technical attempt/retry |
| business_operation_id | Một business command xuyên retry/async |
| parent_transaction_id | Hop/async causal parent |
| decision_id | Mỗi PDP evaluation |
| enforcement_outcome_id | Final response/side effect result |

Trusted ingress bao gồm Edge, internal RPC boundary, consumer và scheduled-job launcher. Nếu message/client cung cấp ID, boundary giữ nó như external_correlation_reference, không dùng làm trusted transaction ID.

## 9.8 Enforcement outcome contract

| Field | Nội dung |
| --- | --- |
| Outcome ID | Unique |
| Transaction/decision IDs | Correlation |
| Action/resource token | Minimal approved identifier |
| Outcome | COMMITTED, RETURNED, FILTERED, NOT_EXECUTED, ERROR, TERMINATED |
| Applied obligations | Type/version/result |
| Resource version | Authorized and final version |
| Policy/revocation state | Digests/generation |
| Timestamp | Start/final |
| Retry/idempotency | attempt and business operation |

# 10. Detailed Flows

## 10.1 External authenticated read

    1. Client -> Edge: request + external credential + optional device proof.
    2. Edge: strip untrusted identity/delegation headers; canonicalize path.
    3. Edge: validate actor credential and device/session evidence.
    4. Edge: create trusted transaction/attempt IDs and map action.
    5. Edge -> STS if needed: obtain sender-bound delegated artifact for Market.
    6. Edge -> Mesh: establish authenticated workload connection.
    7. Mesh -> Market Service: verified immediate caller + untrusted-until-verified delegation artifact.
    8. Service PEP: validate delegation exact audience/binding/TTL/replay.
    9. Service/Domain: load order data and policy facts from same snapshot/version.
   10. Service PEP -> PDP: canonical request.
   11. PDP -> PEP: decision, validity, digests and obligations.
   12. PEP: verify validity/capability, apply obligations before first byte.
   13. Service -> Client: filtered response or controlled error.
   14. Service -> Audit Relay: decision and final outcome.

Workload Trust xác minh caller, không xác minh actor.

## 10.2 Multi-hop on-behalf-of

    Edge(A) -> Market(B) -> Broker(C)

- B không forward artifact audience B sang C.
- B dùng own workload identity để exchange actor authority thành artifact audience C.
- C xác minh immediate caller B, actor, original purpose/chain reference và scope.
- Delegation depth vượt profile hoặc B không phải allowed exchanger -> deny.

## 10.3 Device-conditional access

    1. Edge xác minh actor.
    2. Device adapter resolve posture từ tokenized device reference.
    3. PDP nhận device facts có observed_at/expires_at.
    4. Policy có thể ALLOW, DENY hoặc ALLOW với mask/step-up obligation.
    5. Device quarantine event invalidates session/decision trong objective.

## 10.4 High-risk mutation

    begin authoritative transaction
      -> lock/read resource version N
      -> resolve actor/device/caller/application and transactional facts
      -> evaluate; positive cache disabled by default
      -> verify every obligation and invariant
      -> persist mutation + audit intent atomically as version N+1
      -> commit
    else
      -> rollback
      -> emit NOT_EXECUTED

External side effect:

    transaction commits durable command + audit intent
      -> idempotent dispatcher calls provider
      -> provider outcome updates operation and audit

Không gọi provider trước khi durable intent tồn tại.

## 10.5 Async event and deferred command

- Committed event là historical fact; consumer xác minh producer identity, schema, integrity và replay/idempotency.
- Event không mang lại user ALLOW cũ.
- Deferred command authorize lại theo current state hoặc dùng approved grant còn hiệu lực.
- Consumer tạo trusted transaction/attempt ID mới, giữ parent/business operation ID.
- Inbox/outbox và side effect idempotency là bắt buộc.

## 10.6 Long-running session/stream

- Initial ALLOW có valid_until.
- PEP theo dõi revocation/device/application/session events hoặc re-evaluate theo maximum interval.
- Khi state stale/revoked, dừng stream tại bounded point và ghi TERMINATED outcome.
- Không tiếp tục gửi cached sensitive data sau validity.

## 10.7 Protected egress

    Workload -> local/mesh egress PEP -> approved egress gateway -> external service

1. Caller/workload identity verified.
2. Destination registry row and purpose checked.
3. Network policy prevents direct internet path.
4. Gateway verifies external TLS identity and applies DLP/rate/audit controls.
5. Response passes size/content controls.

# 11. Data, Cache, Revocation and Audit Architecture

## 11.1 Logical data ownership

| Data | Authority | Consumer | Freshness/version |
| --- | --- | --- | --- |
| Actor/session/entitlement | IAM/STS | Edge/PEP/PDP | Credential + entitlement/revoke version |
| Device posture | UEM/EDR/attestation | Edge/Trust Engine/PDP | observed_at/expires_at |
| Workload identity | Workload authority | Mesh/PEP | SVID + bundle generation |
| Application provenance | Build/registry/admission | Platform/PEP/PDP | Artifact digest + attestation |
| Action vocabulary | API Governance + Domain | Edge/PEP/policy | Immutable version |
| Resource facts | Domain | Service PEP/PDP | Resource version/snapshot |
| Risk signals | Registered provider/Trust Engine | PDP | Model/rule/source version |
| Policy | Security Platform + Domain | PDP | Signed digests |
| Revocation | IAM/SecOps/grant/app authorities | PEP/PDP | Ordered generation |
| Decision/outcome | PDP/PEP/domain | Audit | IDs + digests + timestamps |

## 11.2 Provider registry

Mỗi provider khai báo owner, authenticated principal, allowed attributes, schema/type, source of truth, freshness, timeout/error semantics, classification, residency, volume and SLO.

PEP không cho client override server-side tenant/state. PDP không tự gọi provider trong evaluation.

## 11.3 Cache rules

| Cache | Key | Rule |
| --- | --- | --- |
| JWKS/trust bundle | issuer/key ID/trust domain | Proactive refresh; unknown key fail close |
| Policy | immutable digests/composition version | Signature verify; atomic activation; bounded LKG |
| Attribute | subject/resource/provider/version | TTL theo provider/action |
| Decision | canonical fingerprint + all policy/trust/revocation state | Chỉ deterministic result; expire at valid_until |

Decision fingerprint bao phủ actor/session, device, caller/delegation, application digest/status, action/version, resource ID/tenant/version, all relevant facts/source versions, policy/model/revocation state và contract version.

High-risk mutation mặc định không cache ALLOW. Không chứng minh được complete fingerprint hoặc validity thì không dùng decision cache.

## 11.4 LKG and freshness

Action Pack khai báo:

- maximum policy LKG age;
- maximum revocation state age;
- fact/device/application freshness;
- behavior khi vượt budget;
- recovery and alert owner.

LKG là artifact đã verify, không phải quyền giữ ALLOW vô thời hạn. Restore/rollback không activate policy hoặc revocation generation cũ hơn minimum safe state.

## 11.5 Emergency revocation

Production invariant:

> Authority revoke phải chặn quyền trong objective được duyệt và không bị positive cache, LKG, restart, restore hoặc rollback khôi phục.

Candidate mechanisms:

- token/session introspection or invalidation;
- entitlement/device/application version invalidation;
- push invalidation to PEP/PDP;
- emergency signed deny bundle;
- deny-only security floor.

PoC bắt buộc đo authority commit -> 100% healthy PEP enforcement, gồm ordering, signing, distribution, compatibility, cache bypass, active stream termination, restart/restore, partition và clock skew.

Nếu PEP không hiểu revocation schema mới, stable envelope phải cho phép nhận biết scope. Không nhận biết được scope -> fail close toàn action class được envelope đánh dấu; không áp dụng phần hiểu được theo cách có thể mở quyền.

## 11.6 Revocation state safety

- Generation monotonic và persisted ngoài ephemeral runtime.
- Rollback controller có minimum accepted generation.
- TTL expiry của deny entry không làm generation rollback.
- Multi-region writer/order model phải có ADR; không split-brain.
- Signing key compromise có freeze, trust revoke, alternate signer and recovery ceremony.
- PEP fleet báo desired/received/active generation riêng.

## 11.7 Privacy and data minimization

| Data | Rule |
| --- | --- |
| Raw credential/private key | Không log/persist ngoài authority requirement |
| Actor/device/resource ID | Tokenize/hash theo policy; không metric label |
| Posture/risk facts | Minimal allowlist; không gửi full EDR record |
| Policy explain | Redact; authorized support only |
| Decision/outcome | Encrypt, integrity, access control, retention/legal hold |
| Trace/log | Không full request/response hoặc secret |

## 11.8 Audit durability modes

| Mode | Khi local audit không durable/full/corrupt |
| --- | --- |
| REQUIRED_DURABLE | Không side effect visible; 503/incident |
| DEGRADED_BOUNDED | Continue trong signed window/quota; hết budget dùng approved failure action |
| BEST_EFFORT | Continue + alert/reconcile; chỉ public/low-risk |

Mỗi production action có signed row: owner, data class, mode, degraded window, maximum event size, peak EPS, reserved quota, drain rate, retention, failure action và incident owner.

## 11.9 Quota and reconciliation

- Logical quota theo domain/tenant/action class.
- Critical action có reserved capacity.
- High-water backpressure trước full.
- Degraded window dùng persistent/monotonic elapsed time, không reset khi restart.
- Reconciliation so trusted attempt/enforcement counters với collector checkpoint.
- Duplicate audit do retry được deduplicate bằng outcome/attempt/business-operation semantics.

Capacity:

**required bytes = peak EPS x maximum event bytes x outage seconds x safety factor**

## 11.10 Transactional audit intent

- Domain mutation: business state và audit intent commit atomically trong same authoritative transaction.
- External side effect: durable command/audit intent trước dispatcher.
- Read: local enqueue behavior theo action mode; final response outcome được emit sau obligation enforcement.

## 11.11 Data protection profile

Zero Trust access control không thay thế data governance. Mỗi protected resource type có:

| Control | Requirement |
| --- | --- |
| Classification | Public, internal, confidential, restricted hoặc enterprise equivalent |
| Ownership | Named Domain/Data Owner and steward |
| Access semantics | Action/row/field/purpose rules and explicit tenant model |
| Encryption | In transit and at rest with approved key authority |
| Minimization | Chỉ facts/fields cần cho action/decision |
| Exfiltration | Egress destination/purpose/DLP rule |
| Retention/deletion | Business/legal schedule and legal hold behavior |
| Backup/restore | Access control and key availability preserved; expired data not resurrected |
| Non-production | Synthetic/anonymized by default |

Encryption key access tách khỏi data access khi risk yêu cầu. Backup, analytics replica, search index, cache và export path phải nằm trong resource/flow inventory; không chỉ bảo vệ primary API/database.

# 12. Policy, Configuration and Application Supply Chain

## 12.1 Policy lifecycle

    change request
      -> schema/static validation
      -> positive/negative/property tests
      -> threat and impact replay
      -> domain + security approval
      -> immutable build
      -> provenance + KMS/HSM signature
      -> registry
      -> canary distribution
      -> fleet compatibility/health/parity gate
      -> promote or safe rollback

Production artifact manifest:

- artifact digest and media type;
- platform/domain layer IDs;
- contract/action vocabulary requirements;
- engine/runtime capability requirements;
- revocation envelope version;
- source revision, build identity and provenance;
- signer key ID/purpose/environment;
- created_at and optional expiry;
- test/evidence digest;
- previous/rollback-safe compatibility.

## 12.2 Separation of duties

- High-risk author không tự approve/promote.
- Signer service chỉ sign approved immutable build; không author policy.
- Break-glass tách requester/approver khi có thể, có TTL và retrospective.
- Single technical owner được build/test pilot non-production nhưng mọi production promotion cần independent control-owner approval.

## 12.3 Policy safety properties

- Default deny.
- Deny monotonicity: thêm deny không mở quyền.
- Privilege monotonicity: migration DENY -> ALLOW bị phát hiện.
- Tenant non-interference: thay tenant B facts không làm actor tenant A có quyền.
- Unknown fact/obligation/capability không bị bỏ qua.
- Policy evaluation bounded CPU/memory/input depth.
- Same input/state cho deterministic result.

## 12.4 Mesh and infrastructure configuration

Mesh/network configuration theo cùng lifecycle:

- schema validation and static analysis;
- policy attachment/selector/target verification;
- generated proxy config inspection;
- rejected-distribution metric;
- canary and rollback;
- active fleet status;
- positive and negative traffic test;
- expiry for temporary PERMISSIVE/bypass exception.

## 12.5 Application artifact lifecycle

    reviewed source
      -> isolated build
      -> tests/scans/SBOM
      -> signed immutable artifact + provenance
      -> approved registry
      -> environment promotion by digest
      -> admission verification
      -> workload identity assignment
      -> runtime monitoring/quarantine

Compromised artifact response:

1. Stop promotion and new admission.
2. Mark digest/application generation revoked.
3. Drain/terminate affected instances.
4. Invalidate decisions/actions requiring approved application state.
5. Restore known-safe digest without lowering revocation floor.
6. Reconcile traffic/audit and rotate exposed credentials.

## 12.6 Key management

| Key/purpose | Control |
| --- | --- |
| IAM signing | HSM/KMS, issuer pin, overlap and emergency revoke |
| Workload CA | Environment/trust-domain isolation, short-lived leaves, root recovery |
| Policy/revocation signing | Separate purpose/environment key, SoD, offline/recovery path |
| Artifact signing | Build identity/provenance binding, registry verification |
| Audit encryption/integrity | Separate access, retention-compatible rotation |

Không reuse cùng authoritative key cho nhiều trust domain hoặc purpose nếu làm yếu isolation.

# 13. Deployment and Infrastructure Topology

## 13.1 Environments

| Environment | Data | Trust | Availability | Purpose |
| --- | --- | --- | --- | --- |
| Development | Synthetic | Local/mock authority được cô lập | Best effort | Contract/policy development |
| Integration | Synthetic | Representative identity/mesh | Defined window | Cross-component integration |
| Staging/OAT | Synthetic/anonymized | Production-like trust domain riêng | Multi-AZ where possible | Load, chaos, security, DR |
| Production | Approved data | Strict trust and signed artifacts | Action SLO | Real traffic |

Không share private keys, trust roots, signer purpose hoặc actor credentials giữa non-production và production.

## 13.2 Reference production topology

    Region
      Traffic Manager
        -> Edge replicas in AZ-A/AZ-B
        -> Domain workloads in AZ-A/AZ-B
             - workload identity agent
             - mesh L4/L7 enforcement
             - Service PEP
             - local/near PDP
             - audit relay
        -> Regional policy cache
        -> Regional audit collector
        -> Egress gateway pool on dedicated nodes

    Control planes
      IAM/STS                 multi-AZ external authority
      Device Trust           multi-AZ external authority
      Workload Identity      multi-AZ, trust-domain scoped
      Istio/mesh control     multi-AZ, not request hot path
      Authorization control multi-AZ, signed distribution
      Build/registry         isolated supply-chain services
      Revocation authority  ordered durable state

Data plane phải tiếp tục bằng verified LKG trong approved freshness budget khi control plane outage. Vượt budget thì action fail theo Action Pack; không tiếp tục vô hạn.

## 13.3 Network flow matrix

| Source | Destination | Data/protocol | Controls |
| --- | --- | --- | --- |
| Client | Edge | TLS + external credential/device proof | WAF/rate/size/hygiene/AuthN |
| Edge | IAM/STS | Validation/exchange | Authenticated client, exact audience, timeout |
| Edge | Domain | mTLS + delegation | Caller/destination allowlist |
| Domain proxy | Service PEP/app | Local request/context | No bypass/bind restriction |
| Service PEP | PDP | Canonical auth request | Local/authenticated, size/deadline |
| Service PEP | Domain authority | Minimal facts | Workload identity, least privilege |
| Workload | Egress gateway | Approved outbound request | Mesh identity + NetworkPolicy |
| Egress gateway | External | TLS | Destination pin/policy/audit |
| Control plane | Registry/cache | Signed artifacts | KMS signature, immutable digest |
| Registry/cache | PDP/mesh fleet | Artifact/config | Verify/atomic activate |
| PDP/proxy | Fleet status | Desired/received/active state | Authenticated telemetry |
| PEP/domain | Audit relay | Decision/outcome | Encryption, quota, backpressure |

## 13.4 Network and runtime hardening

- Default-deny Kubernetes NetworkPolicy hoặc equivalent cho protected namespace.
- Separate nodes/pools and service accounts cho ingress/egress/control-plane critical components.
- No public IP cho domain workload nếu không có approved exception.
- Pod/workload chạy non-root, readonly filesystem, seccomp/AppArmor/SELinux theo platform capability.
- HostNetwork, privileged, hostPath, CAP_NET_ADMIN/RAW và debug container bị admission control.
- Proxy admin interface không public; access least privilege.
- Control-plane admin/API tách khỏi data plane và user traffic.
- KMS/HSM/private CA access tách environment/purpose.
- Backup/restore không hạ revocation generation hoặc activate untrusted artifact.

## 13.5 Multi-cluster and multi-region

Chỉ enable sau ADR xác định:

- trust-domain/federation boundary;
- CA hierarchy and bundle distribution;
- service discovery and destination identity semantics;
- promotion authority and policy consistency;
- revocation ordering and partition behavior;
- data residency/audit routing;
- failover and split-brain behavior;
- cross-cluster egress/ingress enforcement;
- capacity under region/AZ loss.

Shared trust root giúp interoperability nhưng tăng blast radius. Mặc định tách administrative/security domain và federation explicit.

## 13.6 Deployment strategy

| Component | Strategy | Rollback rule |
| --- | --- | --- |
| Edge | Canary/rolling | Previous compatible config |
| Mesh control/data plane | Revision/canary by workload | No auth gap; compatibility matrix |
| mTLS mode | Observe -> permissive exception -> strict | Rollback không mở plaintext ngoài bounded exception |
| Service PEP | Rolling/canary by action | Legacy path giữ until gate |
| PDP runtime | N/N-1 rolling | Only compatible policy artifact |
| Policy | Shadow -> canary -> promote | Respect minimum revocation generation |
| Device/application trust | Observe -> advisory -> enforce | Action-specific |
| Audit relay | Rolling with drain | Preserve checkpoint/spool |
| Egress | Shadow/observe -> deny | Emergency exception has owner/TTL |

## 13.7 Configuration compatibility

- Runtime, contract, policy, mesh config and vocabulary support N/N-1 during rolling upgrade.
- Manifest declares minimum/maximum supported capability.
- Incompatible artifact không activate.
- Desired, received, validated and active versions được quan sát riêng.
- Dual-read/dual-evaluate dùng cho breaking contract migration.

# 14. Non-Functional Requirements, Capacity and Reliability

## 14.1 Action classes

| Class | Action | Candidate availability | Failure posture |
| --- | --- | ---: | --- |
| C0 | Privileged, cross-tenant, financial, critical mutation | >= 99.99% | Fail close; durable audit and fresh security state |
| C1 | Authenticated business read/standard mutation | >= 99.95% | Fail close AuthZ; bounded audit degradation only if approved |
| C2 | Public/non-sensitive read, health/discovery | >= 99.9% | Fail open only with registry, classification, owner and expiry |

## 14.2 Canonical SLI definitions

**Eligible valid attempt** là request/command:

- đến đúng registered ingress/action;
- có syntactically valid transport/request;
- không bị client cancellation trước platform processing;
- được deduplicate theo attempt/business-operation rule;
- có expected outcome class được Action Pack định nghĩa.

Availability numerator:

- platform đưa ra correct controlled outcome trong action deadline;
- policy DENY và domain conflict là successful controlled outcomes nếu platform xử lý đúng;
- platform timeout, stale mandatory state, unsupported obligation, dropped attempt hoặc wrong enforcement là failure.

Không được loại platform error khỏi denominator bằng cách map thành policy deny.

Latency đo từ trusted ingress accepted đến final response/outcome; tách queue/client network nếu action contract quy định.

## 14.3 Candidate NFR

| Metric | Candidate target | Gate |
| --- | ---: | --- |
| End-to-end availability | Theo action class | Product/Domain/SRE approval |
| Local PDP evaluation | P95 <= 5 ms; P99 <= 10 ms | Representative policy/input |
| Edge AuthN/coarse AuthZ overhead | P95 <= 15 ms | OAT |
| Near/remote PDP nếu dùng | P95 <= 30 ms in-region | Separate ADR/error budget |
| Standard policy propagation | P95 <= 2 minutes; 100% compatible active | Fleet evidence |
| Emergency revoke | <= 30 seconds candidate | Security/Business PoC |
| Audit delivery | >= 99.99% within 5 minutes | Action mode |
| Workload SVID rotation | No user-visible interruption | Identity conformance |
| Capacity | 2x approved peak; one-AZ loss holds SLO | Load/soak |
| Data isolation | 0 cross-tenant allow without grant | Mandatory |

Numeric targets là candidate L2, không phải production promise cho đến khi có workload model và approver.

## 14.4 Deadline budget

Action Pack phân bổ end-to-end latency cho:

- Edge queue/AuthN/device lookup;
- STS/delegation;
- workload/mesh hop;
- resource/fact lookup;
- PDP evaluation;
- invariant/transaction;
- obligation/serialization;
- local audit durability.

Mỗi component timeout nhỏ hơn remaining deadline. Retry không reset overall deadline.

## 14.5 Retry and idempotency

- Read MAY retry trong remaining deadline nếu error retriable.
- Mutation chỉ retry khi idempotency/business operation contract chứng minh safe.
- Authorization call MAY retry một lần trong bounded budget nếu evaluation deterministic và không tạo side effect.
- Retry storm được chặn bằng circuit breaker/bulkhead/backoff/jitter.
- Mesh retry config không được áp dụng rộng cho unsafe method.

## 14.6 Rate and quota model

| Boundary | Quota key | Behavior |
| --- | --- | --- |
| External Edge | client/tenant/route/action | 429/throttle |
| Edge -> Domain | caller/action/tenant | backpressure |
| PEP -> PDP | workload/action + bounded concurrency | short queue then fail close |
| PEP -> fact provider | provider/action bulkhead | circuit/cache if permitted |
| Workload -> egress | caller/destination/purpose | deny/throttle |
| Producer -> audit | domain/tenant/action class | action durability mode |
| Control-plane change | author/domain/environment | queue/rate/approval |

Quota không được dùng raw personal identifier làm metric label.

## 14.7 Scaling

| Component | Signal | Constraint |
| --- | --- | --- |
| Edge | concurrency, queue, latency, CPU | Multi-AZ, connection drain |
| Mesh/waypoint | connection/request load, CPU, config size | Security-boundary aware |
| PEP/PDP | evaluation concurrency, latency, memory | Scale together or capability routing |
| Fact provider | QPS/cache miss/latency | Batch, bounded query, bulkhead |
| Policy distribution | fleet lag/download/activation | Regional cache, jitter/backoff |
| Audit | bytes, oldest age, drain rate | Persistent capacity, fair quota |
| Control plane | config/change/build volume | Not request path; churn control |

## 14.8 Reliability matrix

| Dependency | Failure behavior | Recovery |
| --- | --- | --- |
| IAM/JWKS | Unknown/stale beyond overlap -> AuthN fail | Refresh, alert, IAM recovery |
| Device provider | Fresh cache if action allows; else deny/step-up/503 | Provider recovery |
| Workload CA/bundle | Existing valid SVID within budget; new/expired fail | Multi-AZ CA, bundle runbook |
| Mesh control plane | Existing proxy config/LKG | Restore/reconcile active state |
| PDP | Fail close; controlled unavailable | Restart/rollback compatible runtime |
| Fact provider | Fresh cache or fail by action | Circuit/provider recovery |
| Policy registry | Verified LKG within age | Regional cache/restore |
| Revocation | C0/C1 deny/not-ready when stale | Emergency path |
| Audit | Action durability mode | Drain/reconcile/replay |
| Egress gateway | Fail closed for protected egress | Multi-AZ reroute |

## 14.9 RTO/RPO

| Capability | Candidate RPO | Candidate RTO |
| --- | ---: | ---: |
| Edge/mesh/PDP runtime | No request state restore | Auto failover within action SLO |
| Policy/control metadata | <= 15 minutes | <= 60 minutes |
| Revocation/signing capability | No loss of key/generation state | <= 15 minutes emergency operation |
| Workload identity issuance | No loss of authoritative registration/root state | Within identity SLO |
| Audit | Per retention/compliance contract | Per SecOps/business need |

Compromise recovery khác outage recovery và có separate runbook.

## 14.10 Capacity inputs

| ID | Required input | Owner |
| --- | --- | --- |
| CAP-01 | Average/peak/burst/concurrency by action | Product + SRE |
| CAP-02 | Policy/rule/bundle/input size | Authorization + Domains |
| CAP-03 | Fact/device lookup distribution and cache hit | Providers + SRE |
| CAP-04 | Audit EPS/event size/outage objective | SecOps + Product |
| CAP-05 | Shadow dual-evaluation overhead | Platform + SRE |
| CAP-06 | Mesh proxy/waypoint connection and config scale | Platform/SRE |
| CAP-07 | Multi-AZ/region traffic and residency | Platform + Security |
| CAP-08 | Egress destinations/volume/DLP overhead | Domains + Security |

## 14.11 Cost model

Cost estimate gồm:

- Edge, mesh proxy/ztunnel/waypoint, PEP/PDP compute;
- identity/CA/KMS/HSM;
- policy build/registry/distribution;
- device/risk provider queries;
- audit ingest/storage/retention/query;
- egress gateway/network and external traffic;
- shadow/canary dual processing;
- multi-AZ/region redundancy;
- operational staffing/on-call.

FinOps estimate phải dùng cost per million actions, peak/N-1 capacity và retained GB-month; không chỉ monthly compute average.

# 15. Observability, Audit and Operational Readiness

## 15.1 Observability principles

- Một action có canonical transaction, attempts, decisions và final outcome.
- Mesh telemetry mô tả traffic/proxy outcome; application outcome là authority cho business result.
- Explicit deny, input error, dependency unavailable, stale state, obligation failure và business conflict được phân biệt.
- Raw actor/device/tenant/resource ID không là metric label.
- Security telemetry không bị sampling mất mandatory event; distributed trace có controlled sampling.

## 15.2 Required metrics

| Metric | Labels được phép |
| --- | --- |
| Action attempt/availability/latency | environment, action class/version, outcome |
| AuthN reject | issuer/token class/reason |
| Device posture | provider/posture class/freshness result |
| Workload mTLS | trust domain/principal class/result |
| Authorization decision | action class, effect/reason, digest cohort |
| PDP evaluation | topology, cache state, effect, latency |
| Policy/config fleet | desired/received/active cohort, age |
| Revocation | generation, region, convergence |
| Fact lookup | provider/attribute class/freshness |
| Obligation | type/version/returned/applied/failed |
| Egress | destination class/policy outcome |
| Audit | quota class, bytes, oldest age, drain/reconcile |
| Migration | legacy/new four-way comparison, cohort |
| Supply chain | admission/provenance/quarantine result |

## 15.3 Required alerts

| Alert | Severity/owner |
| --- | --- |
| Cross-tenant allow without registered grant | Critical - SecOps/Domain |
| Legacy DENY -> new ALLOW | Critical - Security/Domain |
| Revocation convergence breach | Critical - Security/SRE |
| Plaintext/unknown workload principal | Critical/High - Platform |
| Route/handler/egress coverage drift | High - Platform/Domain |
| Policy/config signature/schema/activation drift | High - Security Platform |
| Device/application revoked but active | Critical/High - respective owner |
| Action SLO fast burn | Critical - SRE/System owner |
| Audit high-water/full/reconcile gap | Critical/High - SecOps/SRE |
| Break-glass use | Critical - SecOps/Approver |
| Egress bypass attempt | Critical - Security/Platform |
| Workload certificate/bundle expiry risk | High/Critical - Platform |

## 15.4 SLO and error budget

- Rolling 30-day budget theo action.
- Explicit policy deny/business conflict đúng semantics không tiêu budget.
- Platform error, timeout, stale mandatory state, wrong enforcement và obligation failure tiêu budget.
- Security event không chờ budget cạn.
- Rollout tự pause khi burn tăng hoặc critical signal xuất hiện.

Candidate burn alerts:

| Level | Condition | Action |
| --- | --- | --- |
| Fast | >= 14.4x over both 1h and 5m windows | Page, incident, pause rollout |
| Sustained | >= 6x over both 6h and 30m | Page/on-call, pause |
| Slow | >= 1x over both 3d and 6h | Ticket/deadline/review |

SRE Service Level Policy có thể điều chỉnh nhưng không làm yếu security-event alert.

## 15.5 Log and trace governance

- Field allowlist, classification, owner, retention, access and sampling.
- Không raw token, private key, full body/response hoặc unredacted facts.
- Authorization explain view redact PII và policy internals.
- Proxy admin/access logs có access control và không expose secret/header.
- Restore không làm dữ liệu hết retention tái xuất hiện.
- Trace baggage không mang raw identity/credential.

## 15.6 Required runbooks

1. IAM/JWKS outage and emergency actor/client revoke.
2. Device provider outage/quarantine and stale posture.
3. Workload CA/SVID/bundle rotation/federation failure.
4. Plaintext or unknown caller detection.
5. Mesh config reject, proxy drift and policy attachment failure.
6. PDP crash/latency/cold-start/capability mismatch.
7. Fact provider outage/cache storm.
8. Corrupt/compromised policy or artifact signer.
9. Application digest quarantine and workload drain.
10. Revocation lag, partition and rollback safety.
11. Audit full/corrupt/noisy tenant/reconciliation.
12. Egress bypass/destination compromise.
13. AZ/region loss, restore and post-restore validation.
14. Break-glass activate/revoke/retrospective.

## 15.7 Production readiness checklist

| Area | Evidence |
| --- | --- |
| Ownership | Named owner/on-call/escalation |
| Identity | IAM/delegation negative E2E |
| Device | Required action posture contract or explicit out-of-scope decision |
| Workload | SPIFFE conformance/rotation/failure drill |
| Application | Provenance/admission/runtime inventory |
| Coverage | Route/handler/consumer/job/response/egress report |
| Mesh/network | Strict mTLS, default deny, bypass tests |
| Policy | Test/sign/provenance/rollback/revoke |
| Data | Same-snapshot/resource-version evidence |
| Capacity | Workload model, 2x peak, one-AZ |
| Reliability | Chaos/restore/revocation/audit-full |
| Observability | Dashboard/alerts/synthetics |
| Privacy/audit | Signed fields/retention/durability |
| Migration | Shadow/canary/rollback/decommission |

# 16. Migration and Pilot

## 16.1 Migration state machine

    INVENTORY
      -> OBSERVE
      -> SHADOW
      -> CANARY_ENFORCE
      -> FULL_ENFORCE
      -> RETIRE_LEGACY

Rollback:

- CANARY_ENFORCE -> SHADOW khi SLO/security/parity gate fail.
- FULL_ENFORCE rollback chỉ về verified safe path không vi phạm revocation floor.
- RETIRE_LEGACY chỉ sau zero-traffic window, credential/network/secret revoke và owner sign-off.

## 16.2 Four-way comparison

| Legacy | New | Classification |
| --- | --- | --- |
| ALLOW | ALLOW | Match; vẫn cần negative corpus |
| DENY | DENY | Match |
| ALLOW | DENY | Security improvement hoặc regression; owner review |
| DENY | ALLOW | Privilege expansion; rollout blocker |

Parity không chứng minh cả hai policy đúng. Negative/property/security corpus vẫn bắt buộc.

## 16.3 Measurable cohort gates

Mỗi Action Pack MUST định nghĩa trước:

- cohort key và tránh tenant bias;
- minimum requests và minimum wall-clock duration;
- representative low/high traffic windows;
- maximum unexplained mismatch: zero cho DENY -> ALLOW;
- platform error/latency/error-budget threshold;
- audit loss/duplicate/reconciliation tolerance;
- policy/config fleet convergence threshold;
- automatic pause/rollback signal;
- approver and evidence location.

Không dùng cụm từ đủ traffic đại diện hoặc observation window nếu không có số/tiêu chí trong Action Pack.

## 16.4 Pilot market.order.read

| Item | Baseline |
| --- | --- |
| Actor | Authenticated Market user/agent |
| Device | Observe only phase đầu; không dùng để mở quyền |
| Caller | Edge and Market workload with verified SPIFFE-compatible identity |
| Application | Approved pilot artifact digest/admission state |
| Action | market.order.read@v1 |
| Resource | Canonical order ID and version |
| Facts | Tenant, owner/relationship, state, version, provenance |
| Decision | Same-tenant/resource policy; missing fact deny |
| Obligation | Field/row filtering with capability confirmation |
| Audit | Draft DEGRADED_BOUNDED 15 minutes; external approval required |
| Mesh | Strict mTLS for pilot path after caller inventory |
| Egress | No unapproved external egress in request path |
| Rollout | Shadow -> 1% -> 5% -> 25% -> 50% -> 100% |

## 16.5 Pilot success

- 100% route, handler, caller, response and relevant egress path coverage.
- Wrong actor/caller/audience, expired SVID/delegation và spoofed headers bị chặn.
- Resource facts and response data use same snapshot/version.
- No unexplained legacy DENY -> new ALLOW.
- P95/P99 and platform errors within approved pilot gate.
- Decision valid_until and cache expiry tests pass.
- Decision/outcome reconcile within signed tolerance.
- Policy/runtime rollback does not revive revoked access.
- Strict mTLS rotation and one-AZ/failure tests meet criteria.

## 16.6 Legacy and exception retirement

Retire BFF/security path khi:

- zero traffic/dependency window đạt;
- rollback window hết theo approval;
- legacy service account, token, secret, port and network path revoked;
- policy/config exception removed;
- dashboards/alerts/on-call ownership moved;
- historical audit and incident records retained correctly.

# 17. Testing and Quality Strategy

## 17.1 Test layers

| Layer | Scope | Gate |
| --- | --- | --- |
| Contract | Schema/version/canonicalization/validity/error/obligation | CI |
| Policy unit | Positive/negative/default deny/composition | CI |
| Property | Tenant isolation, deny monotonicity, cache fingerprint, privilege monotonicity | CI |
| Supply chain | Signature/provenance/SBOM/admission | CI/Staging |
| Integration | IAM/device/workload/mesh/PDP/fact/audit/egress | Staging |
| Identity conformance | SPIFFE ID, attestation, SVID/bundle rotation/federation deny | OAT |
| Coverage | Route/handler/consumer/job/response/egress | Staging/OAT |
| Migration | Four-way parity/cohort/rollback | Each cutover |
| Performance | 2x peak, worst bundle/input, cold start, one-AZ | OAT |
| Chaos/DR | Authority/control/data plane outages, corrupt/stale state | OAT/DR |
| Privacy | Minimize/mask/access/retention/restore | Privacy/OAT |

## 17.2 Critical security cases

1. Wrong issuer/audience/token class/algorithm/key/expiry/not-before deny.
2. Spoofed actor/device/caller/delegation/application header deny.
3. Actor valid but caller invalid and reverse both deny.
4. Delegation wrong callee/caller/depth/expired/replayed deny.
5. Unknown route/action/handler/consumer/direct port no bypass.
6. Plaintext call denied after strict gate.
7. Cross-tenant without grant and grant wrong audience/scope/revoke deny.
8. Resource changes between authorization and response produce no leak.
9. Cache entry expires at earliest authority validity.
10. Emergency revoke bypasses cache/LKG and terminates active stream within objective.
11. Unknown/conflicting obligation returns no data/side effect.
12. Proxy/backend path normalization mismatch corpus produces no bypass.
13. Mesh policy missing/wrong selector/target is detected before rollout.
14. Sidecar/waypoint/egress bypass fails under network policy.
15. Server-first protocol leaks no protected first byte.
16. Corrupt/unsigned/incompatible policy/application artifact not activated/admitted.
17. Compromised signer recovery does not trust compromised digest.
18. Audit full/corrupt/noisy tenant follows signed mode/quota.

## 17.3 Resilience and chaos cases

- IAM/JWKS unavailable/rotating.
- Device provider timeout/stale/quarantine event.
- SVID rotation, CA failover, bundle overlap/expiry.
- Mesh control plane unavailable; proxy config drift.
- PDP crash/slow/cold; policy registry down.
- Fact provider slow/unavailable; cache storm.
- Revocation distribution out-of-order/partition/restart/restore.
- Audit relay disk full/corrupt/drain slow.
- Egress gateway loss or external TLS identity change.
- AZ/region loss and post-failover convergence.

## 17.4 Configuration quality gates

- Schema/lint/static analysis.
- Rendered/generated proxy policy inspection.
- istioctl analyze or equivalent.
- Rejected config/control-plane metrics are zero.
- Positive and negative test executed against active proxies.
- Desired equals active for required fleet percentage.
- Temporary exception has expiry.

## 17.5 Definition of Ready

Action chỉ vào shadow khi có:

1. Named Product/Domain Owner and class.
2. Canonical action/resource/tenant/snapshot semantics.
3. Actor/device/caller/application requirements.
4. Delegation/caller allowlist and expected-flow inventory.
5. Legacy behavior and positive/negative corpus.
6. Obligation capability and partial-response leak plan.
7. Audit durability row/quota/failure approval.
8. Workload/latency/SLO/cohort/rollback thresholds.
9. Route/handler/consumer/response/egress inventory.
10. Policy and runtime compatibility matrix.

## 17.6 Definition of Done

Action production-enforced khi:

1. Contract/policy/integration/security/coverage/load/failure tests pass.
2. Shadow parity has zero unexplained privilege expansion.
3. Canary 100% passes predefined statistical/operational gates.
4. Strict workload trust and no-bypass evidence pass.
5. Same-snapshot/resource-version behavior pass.
6. Decision/outcome audit reconcile meets signed contract.
7. Revocation/rollback/signer recovery drills pass for action risk.
8. Runbook/dashboard/alert/on-call exercised.
9. Domain, Security, SRE and control owners sign evidence.

# 18. Governance, ADR, Risks and Open Issues

## 18.1 Governance gates

| Transition | Required condition |
| --- | --- |
| DRAFT -> UNDER REVIEW | Scope, model, boundaries, requirements, risks, diagrams and owners |
| UNDER REVIEW -> ACCEPTED L2 | Critical invariants accepted; blockers closed or conditionally accepted by correct owner |
| ACCEPTED -> POC BASELINE | Contract/vocabulary/pilot/benchmark plan approved |
| POC -> IMPLEMENTATION BASELINE | L3 artefacts, threat model and PoC evidence complete |
| IMPLEMENTATION -> PRODUCTION | Action DoD, readiness and control-owner approvals complete |

## 18.2 ADR register

| ADR | Decision | Architecture | Readiness |
| --- | --- | --- | --- |
| ADR-001 | Thin Edge; domain owns resource/invariant | PROPOSED | IMPLEMENTATION_READY pilot |
| ADR-002 | Actor/device/caller/application separated | PROPOSED | Contract update required |
| ADR-003 | Distributed local/near PDP | PROPOSED | POC_REQUIRED |
| ADR-004 | Mandatory platform + domain policy composition | PROPOSED | POC_REQUIRED |
| ADR-005 | SPIFFE-compatible X.509 workload identity | PROPOSED | EXTERNAL_APPROVAL/conformance |
| ADR-006 | Istio-compatible mesh as carrier/guardrail | PROPOSED | POC_REQUIRED |
| ADR-007 | Sidecar versus ambient/waypoint topology | PROPOSED | POC_REQUIRED |
| ADR-008 | Exact-audience sender-bound delegation | PROPOSED | EXTERNAL_APPROVAL IAM |
| ADR-009 | Device posture as typed contextual fact | PROPOSED | Outside pilot enforcement |
| ADR-010 | Signed policy/application provenance | PROPOSED | POC_REQUIRED |
| ADR-011 | Action-level audit durability | PROPOSED | EXTERNAL_APPROVAL |
| ADR-012 | Emergency revocation mechanism | PROPOSED | POC_REQUIRED |
| ADR-013 | Bookended egress + network enforcement | PROPOSED | POC_REQUIRED |
| ADR-014 | Action SLO/error-budget/cohort gate | PROPOSED | EXTERNAL_APPROVAL |
| ADR-015 | Multi-cluster/trust federation | PROPOSED | Deferred |

## 18.3 RACI

| Capability | Accountable | Responsible | Consulted |
| --- | --- | --- | --- |
| IAM/delegation | Identity/Security | IAM | Platform/Domains |
| Device trust | Endpoint Security | UEM/EDR Platform | IAM/Security/Product |
| Edge | Application Platform | Gateway/SRE | Security/Domains |
| Workload identity | Platform/SRE | Trust Platform | Security |
| Mesh/network/egress | Platform/SRE | Mesh/Network Team | Security/Domains |
| PEP/PDP/control plane | Security Platform | Authorization Team | SRE/Domains |
| Application provenance | AppSec/Platform | Build/Runtime Platform | Domain/Security |
| Domain policy/facts/invariant | Domain Owner | Domain Team | Security Platform |
| Audit mode/action | Product/Business | Domain + SRE | Security/Legal/SecOps |
| Audit/privacy | SecOps/Privacy | Audit/Data Platform | Product/Legal |
| SLO/on-call | System Owner + Domain | Service Team/SRE | Security |

## 18.4 Pilot staffing

Một Lead Architect/Tech Lead MAY kiêm technical delivery, developer và Platform/SRE implementer trong pilot. Người đó MUST NOT tự phê duyệt thay IAM, Product/Business, Security/Legal, Privacy/SecOps hoặc independent production promoter. Production promotion và break-glass luôn cần separation of duties.

## 18.5 Risk register

| Risk | Severity | Mitigation/closure |
| --- | --- | --- |
| Actor/caller confused deputy | Critical | Delegation profile + negative E2E |
| PEP/mesh/egress bypass | Critical | Full inventory + network/application conformance |
| Stale cache/revocation | Critical | valid_until + revoke/rollback drill |
| Signer/CA compromise | Critical | Key isolation + recovery exercise |
| Cross-tenant/resource TOCTOU | Critical | Same-snapshot/version tests |
| Device/app provenance unavailable | High | Action-specific fail posture and staged adoption |
| Mesh config ignored/misattached | High | Analyze/fleet/active tests |
| Egress exfiltration | Critical | Gateway + network enforcement + bypass test |
| Audit causes outage/loss | Critical | Signed mode/quota/full test |
| Engine/topology latency/cost | High | Representative benchmark |
| Control-plane blast radius | High | Isolation, SoD, rate, regional cache |
| Team bypass due complexity | High | Paved-road PEP/contract/conformance |
| Multi-region trust/order drift | High | Deferred ADR and drills |

Risk acceptance records scope/action, owner, compensating control, approver, evidence, expiry and review date. Critical risk không được Lead Architect tự nhận thay control owner.

## 18.6 Open issues

| ID | Decision/input | Owner | Gate |
| --- | --- | --- | --- |
| OI-01 | IAM token/delegation/sender binding | IAM/Security | S2S PoC |
| OI-02 | Device authority/posture schema | Endpoint Security | Device-aware action |
| OI-03 | Workload issuer/SPIRE adoption | Platform/Security | Strict mTLS |
| OI-04 | PDP engine/topology | Authorization/SRE | Production selection |
| OI-05 | Istio mode/version/waypoint topology | Platform/SRE | Mesh baseline |
| OI-06 | Cross-tenant/deferred grant authority | Security/Domains | Grant enablement |
| OI-07 | Emergency revocation | IAM/Security/SRE | C0/high-risk |
| OI-08 | Application provenance/admission | AppSec/Platform | Production artifact trust |
| OI-09 | Audit fields/retention/residency/mode | SecOps/Privacy/Product | Real data/enforce |
| OI-10 | Workload/SLO/capacity/cost | Product/SRE/FinOps | OAT |
| OI-11 | Multi-region/federation | Platform/Security | DR/multi-region |
| OI-12 | Production owner/on-call | System Owners | Go-live |

## 18.7 Exception governance

Exception record MUST có:

- exact scope: environment/workload/action/flow;
- reason and unavailable control;
- compensating controls;
- risk owner and approver;
- start/end time and automatic expiry;
- telemetry/alert;
- removal test and ticket.

Không có permanent PERMISSIVE, fail-open hoặc direct-egress exception không expiry.

# 19. Delivery Plan and L3 Artefacts

## 19.1 Delivery phases

| Phase | Deliverable | Exit |
| --- | --- | --- |
| P0 Inventory | Action/flow/identity/resource/audit/bypass map | 100% pilot path owned |
| P1 Contract | Identity/delegation, auth contract, audit row, SLO/cohort | Control owners approve |
| P2 Platform PoC | Workload trust, mesh, PDP, policy, audit, telemetry | Benchmark and failure behavior pass |
| P3 Pilot shadow | Four-way parity, coverage, same-snapshot evidence | No privilege expansion |
| P4 Canary | 1/5/25/50/100 evidence | Each predefined gate pass |
| P5 Expand | C1 then selected C0 Action Packs | Action DoD |
| P6 Retire | Legacy access/config/identity revoke | Zero dependency/ownership moved |
| P7 Broaden Zero Trust | Device/app trust and protected egress adoption | Pillar-specific gates |

## 19.2 Initial backlog

1. Inventory end-to-end market.order.read including direct ports and egress.
2. Update authorization contract with validity, delegation and outcome semantics.
3. Define IAM and Delegation Profile.
4. Run workload identity/SPIFFE conformance.
5. Select Istio reference mode for local PoC and document limitations.
6. Implement platform/domain policy composition and negative corpus.
7. Prove same-snapshot read/response behavior.
8. Build signed policy lifecycle and fleet status.
9. Create local durable audit relay/reconciliation.
10. Create measurable shadow/canary Action Pack.
11. Run strict mTLS, bypass, rollback and revocation drills.
12. Readiness review before enforcement.

## 19.3 L3 artefact register

| Artefact | Gate |
| --- | --- |
| IAM and Delegation Profile v1 | Before S2S PoC |
| Device Trust and Posture Profile | Before device-aware policy |
| Workload Identity/SPIFFE Flow Profile | Before strict workload enforcement |
| Application Provenance and Admission Standard | Before production artifact trust |
| Authorization Contract and Vocabulary v1 | Before PEP/PDP integration |
| Policy Composition/Lifecycle/Test/Signing Standard | Before publish |
| PDP/Trust Engine Benchmark ADR | Before production selection |
| Istio Data Plane/Mode/Security Profile | Before mesh baseline |
| Expected Flow and Egress Registry | Before strict network enforcement |
| PEP Coverage and Conformance Specification | Before action enforce |
| Emergency Revocation PoC and ADR | Before high-risk action |
| Decision/Outcome Audit and Retention Contract | Before real data |
| Action-level Audit Durability Matrix | Before action enforce |
| SRE Service Level/Error Budget/Capacity Matrix | Before OAT |
| BFF/Mesh Migration and Rollback Plan | Before cutover |
| Pilot Action Pack market.order.read | Before shadow |
| Dashboard/Alert/On-call/DR Pack | Before production |

# 20. Traceability

## 20.1 Requirement to evidence matrix

| Requirement | Risk | Evidence/test | Gate |
| --- | --- | --- | --- |
| Actor/caller separated | Confused deputy | Wrong actor/caller/audience E2E | S2S |
| Device fact authoritative | Forged posture | Provider signature/freshness tests | Device-aware action |
| Workload identity | Spoof/lateral movement | SPIFFE/mTLS rotation and wrong-principal tests | Strict |
| App provenance | Supply-chain compromise | Signature/SBOM/admission/quarantine tests | Production artifact |
| Default-deny mesh | Unregistered flow | Plaintext/direct-port/unknown-caller tests | Enforce |
| Service PEP | Resource bypass | Handler/consumer/response coverage | Enforce |
| Same resource snapshot | TOCTOU/IDOR | Concurrent owner/tenant/version test | Pilot |
| Decision validity | Stale ALLOW | Earliest-expiry/cache property tests | Pilot |
| Revocation floor | Revived access | Cache/LKG/rollback/restart/stream drill | High-risk |
| Egress control | Exfiltration | Sidecar/gateway bypass tests | Protected egress |
| Signed policy/config | Tampering | Corrupt/signature/provenance test | Publish |
| Durable audit | Loss/outage | Full/noisy tenant/reconcile test | Action enforce |
| Measured rollout | Unsafe migration | Predefined cohort gate report | Canary |

## 20.2 Review evidence package

Mỗi architecture/action review package chứa:

- immutable document/action-pack revision;
- decision/ADR records;
- threat model and risk acceptance;
- contract/policy/config digests;
- test/benchmark/chaos reports;
- fleet/coverage/parity dashboard snapshot;
- approval identities/timestamps;
- exception list and expiries;
- rollback/recovery evidence.

# Appendix A - Glossary

| Term | Definition |
| --- | --- |
| Actor | Principal chịu ý nghĩa nghiệp vụ của action |
| Device | Endpoint/execution environment gắn với actor |
| Caller | Workload thực hiện hop hiện tại |
| Application provenance | Bằng chứng source/build/artifact/runtime |
| Agent/context-aware subject | Quan hệ Actor + Device, không làm mất identity riêng |
| PEP | Component tạo trusted input và enforce decision |
| PDP | Deterministic policy evaluator |
| Trust Engine | Chuẩn hóa/derive contextual risk signals |
| Policy Administrator | Component thực thi rollout/revoke decision tới PEP |
| Obligation | Enforcement bắt buộc kèm candidate ALLOW |
| LKG | Last-known-good verified state có maximum age |
| SPIFFE ID | URI identity của workload trong trust domain |
| SVID | Cryptographic document chứng minh SPIFFE identity |
| Trust domain | Identity namespace/root authority boundary |
| Explicit grant | Signed cross-tenant/deferred permission có scope/validity/revoke |
| Bookended enforcement | Ingress và egress đều được kiểm soát |
| Same-snapshot authorization | Facts và data/transaction dùng cùng resource version |
| Valid attempt | Canonical SLI-eligible request/command |

# Appendix B - Source Mapping

## B.1 Zero Trust Networks, 2nd Edition

| Book topic | Design adoption |
| --- | --- |
| Chapters 1-2: fundamentals, trust delegation, threat model | ZT invariants, authority and adversary model |
| Chapters 3-4: context-aware agents, enforcement, policy/trust engines, data stores | Actor/device tuple, PEP/PDP/Trust Engine boundaries |
| Chapter 5: device identity, attestation, inventory, posture | Device Trust Profile and freshness |
| Chapter 6: user identity and MFA | Actor/assurance/session contract |
| Chapter 7: source/build/distribution/runtime trust | Application provenance, SBOM, admission, runtime quarantine |
| Chapter 8: traffic authentication/encryption/filtering | mTLS, expected flows, bookended ingress/egress |
| Chapter 9: incremental realization and case studies | Inventory, shadow/canary, initial server-to-server pilot |
| Chapter 10: adversarial view | Threat matrix, invalidation, control-plane and DDoS limits |
| Chapter 11: NIST/CISA/DoD frameworks | Component/pillar mapping and governance |
| Chapter 12: adoption challenges | Iterative delivery, ownership and exception governance |

## B.2 Istio in Action

| Book topic | Design adoption |
| --- | --- |
| Chapters 1-4 | Control/data plane, Edge/mesh boundaries |
| Chapter 5 | Canary, traffic shifting and mirroring |
| Chapter 6 | Timeout, retry, circuit breaker and locality resilience |
| Chapters 7-8 | Metrics, traces, Kiali-style topology and troubleshooting |
| Chapter 9 | SPIFFE identity, mTLS, default deny, JWT and external authorization |
| Chapters 10-11 | Active config verification, troubleshooting and control-plane performance |
| Chapters 12-13 | Multi-cluster, trust and VM workload considerations |
| Chapter 14 | Request-path extension trade-offs |
| Appendix C | SPIFFE/SVID foundations |

Book examples reflect their publication period. Production implementation MUST use supported current APIs and official security guidance.

# Appendix C - References

- NIST SP 800-207 - Zero Trust Architecture: https://csrc.nist.gov/pubs/sp/800/207/final
- NIST SP 800-207A - Cloud-Native Multi-Cloud Access Control Model: https://csrc.nist.gov/pubs/sp/800/207/a/final
- CISA Zero Trust Maturity Model v2: https://www.cisa.gov/resources-tools/resources/zero-trust-maturity-model
- OAuth 2.0 Token Exchange, RFC 8693: https://www.rfc-editor.org/rfc/rfc8693.html
- OAuth 2.0 Mutual-TLS, RFC 8705: https://www.rfc-editor.org/rfc/rfc8705.html
- OAuth 2.0 Security BCP, RFC 9700: https://www.rfc-editor.org/rfc/rfc9700.html
- SPIFFE concepts and specifications: https://spiffe.io/docs/latest/spiffe-specs/
- Istio Security Best Practices: https://istio.io/latest/docs/ops/best-practices/security/
- Istio External Authorization: https://istio.io/latest/docs/tasks/security/authorization/authz-custom/
- Istio Egress Gateway guidance: https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/
- Zero Trust Networks, 2nd Edition - Razi Rais, Christina Morillo, Evan Gilman and Doug Barth, O'Reilly, 2024.
- Istio in Action - Christian E. Posta and Rinor Maloku, Manning, 2022.

# Appendix D - Final Review Checklist

- [ ] Scope distinguishes full Zero Trust from ap-authz pilot.
- [ ] Actor, device, caller and application evidence are separate.
- [ ] Workload Trust only claims verified caller.
- [ ] Request/decision/outcome contracts include validity and correlation.
- [ ] Resource facts and returned/mutated data share snapshot/version.
- [ ] Mandatory policy composition and default-deny semantics are approved.
- [ ] Istio mode/version and limitations have L3 ADR.
- [ ] Strict mTLS, NetworkPolicy and egress bypass evidence exist.
- [ ] Device/application trust scope has explicit enforcement phase.
- [ ] Policy/config/application supply chain has signature/provenance.
- [ ] Revocation beats cache/LKG/rollback/restart/active stream.
- [ ] Audit durability, privacy, quota and incident owner are signed.
- [ ] SLI denominator and cohort gates are measurable.
- [ ] Capacity, cost, RTO/RPO and on-call have owners.
- [ ] Critical risks are closed or accepted by correct authority with expiry.

# Appendix E - Assessment

Tài liệu đủ điều kiện đưa vào L2 review và làm baseline tạo L3 artefacts. Nó không tự cấp production approval. Implementation bắt đầu bằng market.order.read, nhưng architecture giữ chỗ rõ cho device trust, application provenance, bookended traffic, continuous invalidation và enterprise governance để tránh biến một authorization service thành toàn bộ Zero Trust Architecture.
