# AI-Native Spring Boot Engineering Handbook
## Docs-First Architecture, Engineering Conventions, and AI Agent Operating Model

**Version:** 1.0  
**Audience:** Backend Engineers, Tech Leads, Architects, AI Coding Agents  
**Primary Stack:** Java, Spring Boot, PostgreSQL, Kafka, Kubernetes  
**Status:** Recommended Engineering Standard

---

# 1. Mục đích tài liệu

Tài liệu này định nghĩa một chuẩn repository Spring Boot được thiết kế để:

1. Developer mới hiểu hệ thống nhanh.
2. AI Agent đọc repository và đưa ra thay đổi đúng hơn.
3. Business knowledge không bị giấu trong code.
4. Architecture không phụ thuộc vào kiến thức truyền miệng.
5. Contract, data, transaction và failure behavior được mô tả rõ ràng.
6. Các convention quan trọng có thể được kiểm tra bằng compiler, test, static analysis hoặc CI.
7. Repository có thể phát triển lâu dài mà không trở thành một "big ball of mud".

Tài liệu này không chỉ trả lời:

> Code phải viết như thế nào?

Nó còn phải trả lời:

> Hệ thống này là gì?  
> Business hoạt động thế nào?  
> Điều gì tuyệt đối không được phá?  
> Data do ai sở hữu?  
> Failure nào có thể xảy ra?  
> Contract nào phải giữ tương thích?  
> AI Agent phải đọc gì trước khi sửa?  
> Làm sao biết thay đổi đã đúng?

---

# 2. First Principles

## 2.1. Vấn đề gốc

Một developer hoặc AI Agent khi nhận task thực chất phải giải quyết chuỗi sau:

```text
Task
  ↓
Hiểu business intent
  ↓
Xác định feature liên quan
  ↓
Xác định business rules / invariants
  ↓
Xác định contract bị ảnh hưởng
  ↓
Xác định data ownership / transaction boundary
  ↓
Xác định failure / concurrency risks
  ↓
Đọc tests
  ↓
Đọc implementation
  ↓
Thay đổi tối thiểu
  ↓
Verify
```

Nếu repository bắt người làm phải đoán ở nhiều bước:

```text
guess business
guess ownership
guess transaction
guess failure semantics
guess compatibility
```

thì xác suất lỗi tăng mạnh.

---

## 2.2. Nguyên tắc cốt lõi

Repository phải tối ưu cho:

```text
1. Explicitness
2. Locality
3. Determinism
4. Verifiability
5. Context Efficiency
6. Evolvability
7. Operability
```

### Explicitness

Knowledge quan trọng phải được viết ra.

### Locality

Những thứ cùng business capability nên nằm gần nhau.

### Determinism

Một task cùng loại phải dẫn đến một vị trí tương đối predictable.

### Verifiability

Rule quan trọng phải có cách kiểm chứng.

### Context Efficiency

AI và developer không nên đọc 100 file chỉ để sửa một rule nhỏ.

### Evolvability

API, schema, event, code và data phải thay đổi được mà không gây breaking change ngoài ý muốn.

### Operability

Production behavior phải observable và recoverable.

---

# 3. Triết lý tổng thể

Repository được nhìn như 5 lớp:

```text
DOCS
 ↓
Explain intent

TESTS
 ↓
Prove behavior

CODE
 ↓
Implement behavior

AUTOMATION
 ↓
Enforce standards

OPERATIONS
 ↓
Run and recover system
```

Câu quan trọng nhất:

> **Do not make the next engineer — human or AI — reverse engineer the business from code.**

---

# 4. Cấu trúc repository tổng thể

```text
project/
│
├── AGENTS.md
├── README.md
├── build.gradle.kts
├── settings.gradle.kts
├── gradlew
├── gradle/
│
├── docs/
│   ├── README.md
│   ├── 00-context/
│   ├── 10-domain/
│   ├── 20-architecture/
│   ├── 30-contracts/
│   ├── 40-data/
│   ├── 50-reliability/
│   ├── 60-security/
│   ├── 70-testing/
│   ├── 80-operations/
│   ├── 90-decisions/
│   ├── engineering/
│   └── ai/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
│
├── scripts/
│   ├── test.sh
│   ├── lint.sh
│   ├── run-local.sh
│   └── verify.sh
│
└── .github/
    └── workflows/
```

---

# 5. Docs-First Architecture

`docs/` không phải thư mục phụ.

Nó là:

```text
Specification
+
Knowledge Base
+
Architecture Map
+
Business Manual
+
Operating Manual
+
AI Context
```

---

# 6. Quy tắc cho documentation

## DOC-001 — One Fact, One Source of Truth

Một fact quan trọng chỉ có một nơi authoritative.

Ví dụ:

```text
Business Rule
→ docs/10-domain/business-rules.md
```

Các file khác chỉ link lại.

---

## DOC-002 — Mỗi loại knowledge có đúng chỗ của nó

Không trộn:

```text
business rule
architecture decision
API contract
operational procedure
coding convention
```

vào cùng một file.

---

## DOC-003 — Searchable identifiers

Các concept quan trọng cần ID:

```text
BR-ORDER-001
INV-ORDER-001
WF-ORDER-001
ADR-021
ERR-ORDER-409-01
EVENT-ORDER-CANCELLED
```

---

## DOC-004 — Metadata

Mỗi tài liệu authoritative nên có:

```yaml
---
title: Order Business Rules
status: active
owner: order-team
last-reviewed: 2026-10-04
review-cycle: 90d
source-of-truth: true
---
```

---

## DOC-005 — Status

```text
draft
active
deprecated
superseded
archived
```

---

# 7. Cấu trúc `docs/`

```text
docs/
│
├── README.md
│
├── 00-context/
│   ├── product.md
│   ├── goals.md
│   ├── glossary.md
│   └── system-context.md
│
├── 10-domain/
│   ├── README.md
│   ├── domain-model.md
│   ├── business-rules.md
│   ├── invariants.md
│   ├── workflows.md
│   ├── state-machines.md
│   ├── permissions.md
│   └── edge-cases.md
│
├── 20-architecture/
│   ├── README.md
│   ├── principles.md
│   ├── module-boundaries.md
│   ├── dependency-rules.md
│   ├── runtime-architecture.md
│   ├── data-flow.md
│   └── deployment-architecture.md
│
├── 30-contracts/
│   ├── README.md
│   ├── api/
│   ├── events/
│   ├── integrations/
│   └── error-codes.md
│
├── 40-data/
│   ├── README.md
│   ├── data-model.md
│   ├── schema.md
│   ├── ownership.md
│   ├── consistency.md
│   ├── transaction-model.md
│   ├── migration-strategy.md
│   └── retention.md
│
├── 50-reliability/
│   ├── README.md
│   ├── failure-model.md
│   ├── timeout-retry.md
│   ├── idempotency.md
│   ├── concurrency.md
│   ├── capacity-model.md
│   ├── slo.md
│   └── observability.md
│
├── 60-security/
│   ├── README.md
│   ├── security-model.md
│   ├── authentication.md
│   ├── authorization.md
│   ├── secrets.md
│   └── sensitive-data.md
│
├── 70-testing/
│   ├── README.md
│   ├── strategy.md
│   ├── test-pyramid.md
│   ├── fixtures.md
│   ├── integration-tests.md
│   └── contract-tests.md
│
├── 80-operations/
│   ├── README.md
│   ├── local-development.md
│   ├── environments.md
│   ├── configuration.md
│   ├── deployment.md
│   ├── rollback.md
│   ├── runbook.md
│   └── incident-playbook.md
│
├── 90-decisions/
│   ├── README.md
│   └── ADR-xxxx-*.md
│
├── engineering/
│   ├── README.md
│   └── CONVENTIONS.md
│
└── ai/
    ├── README.md
    ├── AGENT-OPERATING-MANUAL.md
    ├── READ-ORDER.md
    ├── TASK-PROTOCOL.md
    ├── CHANGE-MATRIX.md
    ├── STOP-CONDITIONS.md
    └── VERIFICATION.md
```

---

# 8. `docs/README.md`

## Vai trò

Entry point duy nhất cho documentation.

## Phải trả lời

```text
Docs được tổ chức thế nào?
Source of truth nằm ở đâu?
Task nào đọc phần nào?
Read order ra sao?
```

## Template

```md
# Documentation

## Purpose

This directory is the source of truth for:
- business meaning
- business rules
- architecture
- contracts
- data semantics
- reliability
- operations
- engineering conventions

## Read Order

1. 00-context/
2. 10-domain/
3. relevant feature README
4. relevant architecture/data/contracts
5. tests
6. code

## Source of Truth

Business rules:
10-domain/business-rules.md

Data ownership:
40-data/ownership.md

API:
30-contracts/api/

Architecture decisions:
90-decisions/

Engineering conventions:
engineering/CONVENTIONS.md
```

---

# 9. `00-context/`

Mục tiêu:

> Hệ thống tồn tại để làm gì?

---

## 9.1. `product.md`

Chứa:

```text
problem
users
business value
primary workflows
```

Không chứa implementation detail.

---

## 9.2. `goals.md`

Chứa:

```text
Goals
Non-Goals
```

Ví dụ:

```text
G-001
Assignment must be deterministic and auditable.

NG-001
System does not manage HR employee lifecycle.
```

Non-goals giúp chống scope creep.

---

## 9.3. `glossary.md`

Định nghĩa ubiquitous language.

Ví dụ:

```md
## Sale

Employee who can receive Leads.

Do not use:
- Seller
- Agent
- Salesman
```

Nếu code và docs dùng nhiều từ cho cùng một concept, glossary là source of truth.

---

## 9.4. `system-context.md`

Mô tả hệ thống trong ecosystem:

```text
users
upstream systems
downstream systems
external dependencies
ownership
protocol
failure impact
```

---

# 10. `10-domain/`

Đây là phần quan trọng nhất.

Mục tiêu:

> Mô tả business truth.

---

## 10.1. `README.md`

Index domain.

---

## 10.2. `domain-model.md`

Mô tả:

```text
business concepts
identity
relationships
responsibilities
ownership
non-responsibilities
```

Không đồng nhất domain model với DB schema.

---

## 10.3. `business-rules.md`

Source of truth của business rules.

Template:

```md
## BR-ASSIGN-001 — Sale must be eligible

### Statement

A Lead MUST NOT be assigned to a Sale
unless the Sale is eligible at assignment time.

### Inputs

- projectId
- saleId
- eligibility

### Failure

SALE_NOT_ELIGIBLE

### Related

- INV-ASSIGN-001
- WF-ASSIGN-001
```

---

## 10.4. `invariants.md`

Invariant là điều không được phép sai trong consistency boundary.

Ví dụ:

```text
INV-ASSIGN-001
At most one ACTIVE Assignment may exist for a Lead.
```

Phải ghi:

```text
scope
enforcement
concurrency behavior
verification
```

---

## 10.5. `workflows.md`

Mô tả end-to-end business flow.

Mỗi workflow có:

```text
trigger
preconditions
steps
business rules
state changes
side effects
failure paths
compensation
outputs
```

---

## 10.6. `state-machines.md`

Mô tả lifecycle.

Ví dụ:

```text
UNASSIGNED
  ↓ assign
ASSIGNED
  ↓ complete
COMPLETED
```

Cần cả diagram và transition table.

---

## 10.7. `permissions.md`

Phân biệt:

```text
Authentication
Authorization
Business Eligibility
```

---

## 10.8. `edge-cases.md`

Lưu knowledge mà senior engineers thường giữ trong đầu:

```text
duplicate input
empty eligible pool
late callback
stale state
same timestamp
concurrent update
```

---

# 11. `20-architecture/`

Trả lời:

> Hệ thống được chia như thế nào để implement domain?

---

## 11.1. `principles.md`

Các principle sống lâu hơn implementation.

Ví dụ:

```text
Business rules stay independent from infrastructure.
Schema evolution supports rolling deployment.
Network failure is expected.
Observability is part of production behavior.
```

---

## 11.2. `module-boundaries.md`

Mỗi module phải ghi:

```text
owns
may depend on
must not depend on
exposes
```

---

## 11.3. `dependency-rules.md`

Baseline:

```text
api → application
application → domain
infrastructure → domain

domain ↛ infrastructure
domain ↛ Spring MVC
domain ↛ JPA implementation
domain ↛ Kafka
```

---

## 11.4. `runtime-architecture.md`

Mô tả runtime topology:

```text
service
database
cache
Kafka
external service
OTel collector
gateway
```

---

## 11.5. `data-flow.md`

Mô tả:

```text
where data enters
where it transforms
where it persists
sync vs async
source of truth
consistency
```

---

## 11.6. `deployment-architecture.md`

Mô tả topology vận hành:

```text
Kubernetes deployment
service
HPA
ConfigMap
Secret
monitoring
```

Không copy toàn bộ YAML.

---

# 12. `30-contracts/`

Nếu một thứ có consumer độc lập thì nó là contract.

---

## 12.1. `api/`

OpenAPI nên là machine-readable source of truth.

Narrative docs chỉ giải thích:

```text
semantics
examples
compatibility
business meaning
```

---

## 12.2. `events/`

Mỗi event phải mô tả:

```text
meaning
producer
consumer
schema
delivery semantics
ordering
idempotency
compatibility
```

---

## 12.3. `integrations/`

Mỗi external dependency phải có:

```text
purpose
owner
protocol
auth
timeouts
retry
rate limit
failure behavior
SLA assumptions
```

---

## 12.4. `error-codes.md`

Error code phải stable và machine-readable.

Ví dụ:

```text
SALE_NOT_ELIGIBLE
LEAD_ALREADY_ASSIGNED
WORKFORCE_UNAVAILABLE
```

---

# 13. `40-data/`

Trả lời:

> Data có meaning gì, ai sở hữu, consistency ra sao?

---

## 13.1. `data-model.md`

Logical business data model.

---

## 13.2. `schema.md`

Physical schema:

```text
table
primary key
columns
constraints
indexes
access patterns
```

---

## 13.3. `ownership.md`

Ví dụ:

| Data | Owner | Writers | Readers |
|---|---|---|---|
| Lead | Lead | Lead | Distribution |
| Eligibility | Eligibility | Eligibility | Distribution |
| Assignment | Assignment | Assignment | Reporting |

---

## 13.4. `consistency.md`

Ví dụ:

```text
Assignment + Lead state
→ strong consistency

Assignment + Notification
→ eventual consistency

Assignment + Analytics
→ eventual consistency
```

---

## 13.5. `transaction-model.md`

Phải ghi:

```text
transaction boundary
aggregate boundary
isolation assumption
locking
external calls excluded/included
outbox
```

---

## 13.6. `migration-strategy.md`

Chuẩn:

```text
EXPAND
 ↓
DEPLOY COMPATIBLE CODE
 ↓
BACKFILL
 ↓
SWITCH
 ↓
CONTRACT
```

---

## 13.7. `retention.md`

Mỗi data type cần:

```text
retention period
archive
deletion
audit
PII requirement
```

---

# 14. `50-reliability/`

Đây là production reality.

---

## 14.1. `failure-model.md`

Với mỗi dependency:

```text
failure mode
detection
impact
handling
recovery
```

---

## 14.2. `timeout-retry.md`

Centralized matrix:

| Dependency | Connect | Read | Retry | Backoff |
|---|---:|---:|---:|---|
| Workforce | 200ms | 500ms | 2 | exponential |
| Kafka | n/a | 5s | bounded | internal |
| Email | 300ms | 2s | async | consumer |

---

## 14.3. `idempotency.md`

Ghi:

```text
operation
idempotency key
scope
storage
TTL
conflict behavior
```

---

## 14.4. `concurrency.md`

Mô tả race conditions.

Ví dụ:

```text
Race:
Two workers assign same Lead.

Protection:
DB unique constraint + transaction.

Not sufficient:
check then insert.
```

---

## 14.5. `capacity-model.md`

Không viết:

```text
high traffic
```

Phải ghi:

```text
normal RPS
peak RPS
message/sec
payload distribution
connections
thread/concurrency assumptions
growth
```

---

## 14.6. `slo.md`

Ví dụ:

```text
Availability 99.9%
p95 < 300ms
p99 < 700ms
error rate < 0.1%
```

---

## 14.7. `observability.md`

Map:

```text
business process
  ↓
metric
log
trace
alert
```

---

# 15. `60-security/`

---

## 15.1. `security-model.md`

High-level trust boundaries.

---

## 15.2. `authentication.md`

Trả lời:

```text
identity đến từ đâu?
token?
s2s identity?
```

---

## 15.3. `authorization.md`

Trả lời:

```text
ai có quyền gì?
resource ownership?
tenant isolation?
```

---

## 15.4. `secrets.md`

Trả lời:

```text
secret source
delivery
rotation
logging restrictions
```

---

## 15.5. `sensitive-data.md`

Phân loại:

```text
PII
financial
credential
internal
public
```

và policy tương ứng.

---

# 16. `70-testing/`

---

## 16.1. `strategy.md`

Mapping:

```text
Domain Rule
→ unit test

DB behavior
→ PostgreSQL integration test

HTTP contract
→ contract/controller test

Cross-service
→ contract test

Critical flow
→ e2e
```

---

## 16.2. `test-pyramid.md`

Định nghĩa loại test project dùng:

```text
unit
slice
integration
contract
e2e
architecture
```

---

## 16.3. `fixtures.md`

Fixtures phải thể hiện business meaning:

```text
activeSale()
inactiveSale()
eligibleSale()
assignedLead()
```

---

## 16.4. `integration-tests.md`

Ghi:

```text
PostgreSQL Testcontainers
Kafka test strategy
mock server
migration tests
```

---

## 16.5. `contract-tests.md`

Ghi:

```text
OpenAPI
event schema compatibility
consumer-driven contracts
```

---

# 17. `80-operations/`

---

## 17.1. `local-development.md`

Phải giúp developer mới chạy project được.

---

## 17.2. `environments.md`

Mô tả:

```text
local
dev
sit
uat
prod
```

với mục đích từng environment.

---

## 17.3. `configuration.md`

Source of truth cho precedence:

```text
application.yml
  ↓
profile config
  ↓
environment variables
  ↓
ConfigMap
  ↓
Secret/Vault
```

---

## 17.4. `deployment.md`

Mô tả:

```text
build
test
image
deploy
health
progressive rollout
verification
```

---

## 17.5. `rollback.md`

Phải xét:

```text
application rollback
DB compatibility
event compatibility
config rollback
```

---

## 17.6. `runbook.md`

Technical recovery procedures.

---

## 17.7. `incident-playbook.md`

Process:

```text
detect
acknowledge
mitigate
communicate
recover
postmortem
```

---

# 18. `90-decisions/`

ADR là memory của kiến trúc.

Code cho biết:

> đang làm gì

ADR cho biết:

> tại sao làm như vậy

---

## ADR Template

```md
# ADR-021 — Transactional Outbox

## Status
Accepted

## Context

...

## Problem

...

## Options

...

## Decision

...

## Consequences

Positive:
...

Negative:
...

## Migration

...

## Related

...
```

---

# 19. `engineering/`

`engineering/` mô tả:

> developer và AI phải viết/chỉnh code thế nào.

---

# 20. `ai/`

Nếu AI là first-class contributor thì cần riêng operating model.

---

## 20.1. `AGENT-OPERATING-MANUAL.md`

Agent workflow:

```text
understand
locate
plan
implement
verify
report
```

---

## 20.2. `READ-ORDER.md`

Default:

```text
Task
 ↓
00-context
 ↓
10-domain
 ↓
feature README
 ↓
rules/invariants
 ↓
architecture/contracts/data/reliability
 ↓
tests
 ↓
code
```

---

## 20.3. `TASK-PROTOCOL.md`

Các phase:

```text
PHASE 1 — UNDERSTAND
PHASE 2 — LOCATE
PHASE 3 — PLAN
PHASE 4 — IMPLEMENT
PHASE 5 — VERIFY
PHASE 6 — REPORT
```

---

## 20.4. `CHANGE-MATRIX.md`

| Change | Must Check |
|---|---|
| API field | OpenAPI, compatibility, tests |
| DB column | migration, rollback, old app |
| Event | producer, consumers, schema |
| Business rule | docs, tests, errors |
| Retry | idempotency, timeout, duplicates |
| Transaction | isolation, locking, outbox |

---

## 20.5. `STOP-CONDITIONS.md`

Agent phải dừng khi:

```text
business rule conflicts
source of truth unclear
destructive migration uncertain
authorization unclear
financial action may duplicate
breaking contract not requested
transaction semantics unclear
```

---

## 20.6. `VERIFICATION.md`

Agent chỉ được claim done khi các verification required đã chạy.

---

# 21. Source of Truth Matrix

| Knowledge | Source of Truth |
|---|---|
| Product purpose | `00-context/product.md` |
| Goals | `00-context/goals.md` |
| Terminology | `00-context/glossary.md` |
| Domain concepts | `10-domain/domain-model.md` |
| Business rules | `10-domain/business-rules.md` |
| Invariants | `10-domain/invariants.md` |
| Workflow | `10-domain/workflows.md` |
| State transitions | `10-domain/state-machines.md` |
| Permissions | `10-domain/permissions.md` |
| Module ownership | `20-architecture/module-boundaries.md` |
| Dependency rules | `20-architecture/dependency-rules.md` |
| HTTP shape | OpenAPI |
| Event contracts | `30-contracts/events/` |
| Error codes | `30-contracts/error-codes.md` |
| Data ownership | `40-data/ownership.md` |
| Consistency | `40-data/consistency.md` |
| Transaction | `40-data/transaction-model.md` |
| Schema evolution | `40-data/migration-strategy.md` |
| Failure model | `50-reliability/failure-model.md` |
| Timeout/retry | `50-reliability/timeout-retry.md` |
| Idempotency | `50-reliability/idempotency.md` |
| Concurrency | `50-reliability/concurrency.md` |
| Capacity | `50-reliability/capacity-model.md` |
| SLO | `50-reliability/slo.md` |
| Authentication | `60-security/authentication.md` |
| Authorization | `60-security/authorization.md` |
| Testing | `70-testing/strategy.md` |
| Config | `80-operations/configuration.md` |
| Architecture rationale | ADR |
| Coding rules | `engineering/CONVENTIONS.md` |
| AI workflow | `ai/` |

---

# 22. Source Code Structure

Không tổ chức business code chủ yếu theo technical layer.

Không khuyến nghị:

```text
controller/
service/
repository/
entity/
dto/
mapper/
```

Khuyến nghị package-by-feature:

```text
com.company.app
├── customer/
├── order/
├── payment/
└── common/
```

---

# 23. Feature Structure

```text
order/
├── README.md
├── api/
├── application/
├── domain/
└── infrastructure/
```

---

# 24. Feature README

Feature README là local semantic index.

Template:

```md
# Order

## Responsibility

Manage order lifecycle.

## Domain

See:
docs/10-domain/domain-model.md#order

## Business Rules

- BR-ORDER-001
- BR-ORDER-002

## Invariants

- INV-ORDER-001

## Entry Points

- CreateOrderUseCase
- CancelOrderUseCase

## Data

Owner:
Order module.

## Events

Published:
- OrderCreated
- OrderCancelled

## External Dependencies

- Customer
- Payment

## Reliability

See:
docs/50-reliability/

## Tests

- OrderDomainTest
- OrderConcurrencyTest
```

---

# 25. Layer Responsibilities

## API

```text
transport input
validation
mapping
authentication integration
call use case
transport output
```

Không chứa business rule.

---

## Application

```text
use case orchestration
transaction boundary
load
coordinate
persist
publish through abstractions
```

---

## Domain

```text
business rules
invariants
state transitions
value objects
domain behavior
```

Domain nên gần Pure Java.

---

## Infrastructure

```text
PostgreSQL
JPA
Kafka
Redis
HTTP client
S3
external systems
```

---

# 26. Dependency Direction

```text
             ┌─────────────┐
HTTP ──────► │     API     │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │ APPLICATION │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │   DOMAIN    │
             └──────▲──────┘
                    │
             ┌──────┴──────┐
             │ INFRA       │
             └─────────────┘
```

---

# 27. Engineering Convention Language

```text
MUST
MUST NOT
SHOULD
SHOULD NOT
MAY
VERIFY
```

`MUST` nghĩa là vi phạm phải sửa trước merge trừ khi có approved ADR/waiver.

---

# 28. Rule ID

```text
ARCH-xxx
JAVA-xxx
CODE-xxx
SPR-xxx
API-xxx
DATA-xxx
TX-xxx
DIST-xxx
MSG-xxx
OBS-xxx
PERF-xxx
ERR-xxx
TEST-xxx
CFG-xxx
SEC-xxx
AI-xxx
```

---

# 29. Architecture Conventions

## ARCH-001 — Package by Feature

MUST tổ chức business theo feature.

---

## ARCH-002 — Feature Semantic Boundary

Feature lớn SHOULD có:

```text
README
api
application
domain
infrastructure
```

Không tạo folder rỗng.

---

## ARCH-003 — Domain Owns Business Invariants

Business invariant không nằm trong Controller/Mapper/JPA/Kafka listener.

---

## ARCH-004 — Dependency Direction

MUST enforce bằng ArchUnit.

---

## ARCH-005 — No Abstraction Without Boundary

Không tạo interface/base class chỉ vì pattern.

---

## ARCH-006 — Composition Over Inheritance

Inheritance chỉ dùng khi quan hệ và extension contract thực sự hợp lý.

---

## ARCH-007 — No Cross-Feature Infrastructure Coupling

Module không truy cập persistence implementation của module khác.

---

# 30. Java Conventions

## JAVA-001 — Constructor Injection

Dependency bắt buộc đi qua constructor.

---

## JAVA-002 — No Hardwired Replaceable Dependency

Không `new` trực tiếp external capability trong business class.

---

## JAVA-003 — Minimize Visibility

Ưu tiên:

```text
private
package-private
protected
public
```

---

## JAVA-004 — Prefer Immutability

Value object SHOULD immutable.

---

## JAVA-005 — Fields Private

Không public mutable field.

---

## JAVA-006 — Model Business Concepts With Types

Không lạm dụng String/int cho concept quan trọng.

---

## JAVA-007 — Enum Over Magic Constants

Không dùng `3` hoặc `"ACTIVE"` rải rác.

---

## JAVA-008 — Validate Early

Fail gần boundary nhất có trách nhiệm.

---

## JAVA-009 — Never Return Null Collection

Trả empty collection.

---

## JAVA-010 — Use Optional Judiciously

Tốt cho return value khi absence là bình thường.

Không dùng mặc định cho DTO field/entity field/parameter.

---

## JAVA-011 — BigDecimal For Exact Financial Values

Money/tax/interest phải chính xác.

---

## JAVA-012 — equals/hashCode Consistency

Nếu override equals thì xem xét hashCode tương ứng.

---

## JAVA-013 — Interfaces Represent Real Types/Boundaries

Không `Service/ServiceImpl` máy móc.

---

## JAVA-014 — Streams Must Be Readable

Không giấu side effects trong stream chain.

---

## JAVA-015 — No Blind Parallel Streams

Chỉ dùng khi measured và thread-safe.

---

## JAVA-016 — Exceptions For Exceptional Conditions

Không dùng exception cho normal control flow.

---

## JAVA-017 — Never Swallow Exceptions

Không catch rồi bỏ.

---

## JAVA-018 — Translate Exceptions At Boundaries

Infrastructure exception không leak trực tiếp ra API.

---

## JAVA-019 — Preserve Failure Atomicity

Validate trước mutation khi có thể.

---

## JAVA-020 — Optimize After Measurement

Correct → Measure → Optimize → Measure again.

---

# 31. Clean Code Conventions

## CODE-001 — Names Reveal Intent

Tên phải nói được:

```text
what
why
business meaning
```

---

## CODE-002 — Searchable Names

Tránh:

```text
Helper
Manager
Processor
Common
Data
Info
```

khi không có semantic meaning cụ thể.

---

## CODE-003 — One Word Per Concept

Ví dụ thống nhất:

```text
getX
findX
loadX
searchX
```

với semantics rõ.

---

## CODE-004 — Class Names Are Nouns / Concepts

---

## CODE-005 — Function Names Are Verbs

---

## CODE-006 — Functions Do One Thing

---

## CODE-007 — One Level Of Abstraction

Không trộn business + SQL + HTTP detail cùng method.

---

## CODE-008 — Avoid Primitive Parameter Explosion

Cân nhắc command/value object.

---

## CODE-009 — Avoid Flag Arguments

Không:

```java
calculate(order, true);
```

---

## CODE-010 — Side Effects Must Be Explicit

`findCustomer()` không được âm thầm gửi email hay save DB.

---

## CODE-011 — Comments Explain WHY

Không lặp lại code.

---

## CODE-012 — No Commented-Out Code

Git giữ history.

---

## CODE-013 — TODO Must Be Actionable

Ví dụ:

```text
TODO(PLAT-1423)
```

---

## CODE-014 — No Duplicate Business Rules

Một business invariant có một owner.

---

## CODE-015 — Boy Scout Rule Without Scope Creep

Cải thiện vùng liên quan, không biến bug fix thành rewrite.

---

# 32. Spring Boot Conventions

## SPR-001 — Constructor Injection Only

---

## SPR-002 — Controller Is Transport Adapter

Controller không chứa business rules.

---

## SPR-003 — Use Case Owns Transaction Boundary

---

## SPR-004 — `@Transactional` Must Be Intentional

Hiểu:

```text
propagation
isolation
readOnly
rollback
```

---

## SPR-005 — Avoid Proxy Self-Invocation Bugs

Không giả định:

```text
this.transactionalMethod()
```

đi qua Spring proxy.

Áp dụng tương tự cho:

```text
@Async
@Cacheable
@Retryable
```

---

## SPR-006 — Domain Independent From Spring

---

## SPR-007 — Typed Configuration Properties

Ưu tiên `@ConfigurationProperties`.

---

## SPR-008 — No Secrets In Source

---

## SPR-009 — Profiles Override, Not Duplicate

---

## SPR-010 — Repository Does Persistence Only

---

## SPR-011 — JPA Entity Is Not Automatically Domain Model

Có thể dùng chung trong project nhỏ nếu trade-off rõ.

---

## SPR-012 — Map At Boundaries

Không để HTTP DTO truyền thẳng tới persistence entity.

---

# 33. API Conventions

## API-001 — API Is A Contract

---

## API-002 — Consistent Resource Naming

---

## API-003 — HTTP Methods Follow Semantics

---

## API-004 — Stable Status Semantics

---

## API-005 — Machine-Readable Error Code

Ví dụ:

```json
{
  "code": "ORDER_ALREADY_CANCELLED",
  "message": "Order has already been cancelled",
  "traceId": "..."
}
```

---

## API-006 — Preserve Backward Compatibility

Breaking changes cần version/migration strategy.

---

## API-007 — Remote Call Is Not Local Call

Assume:

```text
timeout
partial failure
slow response
duplicate
version mismatch
network partition
```

---

## API-008 — Explicit Timeouts

---

## API-009 — Retry Only Safe Operations

Phải trả lời:

```text
idempotent?
retryable failure?
backoff?
max attempts?
retry storm?
```

---

# 34. Data Conventions

## DATA-001 — Schema Is A Contract

---

## DATA-002 — Production Schema Changes Use Migrations

Không Hibernate auto-DDL trong production.

---

## DATA-003 — Expand → Migrate → Contract

---

## DATA-004 — Do Not Reuse Old Field Meaning

---

## DATA-005 — Index From Access Pattern

---

## DATA-006 — Bound Query Results

Pagination/limit/batch.

---

## DATA-007 — Explicit Types For Money/Time/ID

---

## DATA-008 — Time Semantics Explicit

Phân biệt:

```text
Instant
OffsetDateTime
LocalDate
LocalDateTime
```

---

# 35. Transaction Conventions

## TX-001 — Use Case-Level Transaction Boundary

---

## TX-002 — Avoid Long Network Call Inside DB Transaction

---

## TX-003 — Isolation Based On Anomaly

Phân tích:

```text
dirty read
non-repeatable read
lost update
write skew
phantom
```

---

## TX-004 — Concurrent Update Must Be Designed

Chọn rõ:

```text
optimistic lock
pessimistic lock
atomic SQL
serialization
application conflict rule
```

---

## TX-005 — Optimistic Locking Where Appropriate

---

# 36. Distributed System Conventions

## DIST-001 — Assume Partial Failure

---

## DIST-002 — Retry Means Duplicate Is Possible

---

## DIST-003 — Idempotency For Retried Mutations

---

## DIST-004 — Avoid Vague "Exactly Once"

Mô tả guarantee thực:

```text
at-least-once
+
idempotent processing
+
dedup scope
```

---

## DIST-005 — Design For Backpressure

---

## DIST-006 — Prevent Cascading Failure

Cân nhắc:

```text
timeout
bulkhead
concurrency limit
circuit breaker
retry budget
rate limit
```

---

# 37. Messaging / Kafka Conventions

## MSG-001 — Event Is A Fact

Past tense:

```text
OrderCreated
PaymentCompleted
LeadAssigned
```

---

## MSG-002 — Event Schema Is A Contract

---

## MSG-003 — Consumer Tolerates Redelivery

---

## MSG-004 — Important Side Effects Idempotent

---

## MSG-005 — Avoid Unsafe DB + Kafka Dual Write

Cân nhắc:

```text
Transactional Outbox
CDC
single transactional log
```

---

## MSG-006 — Stable Identifiers In Payload

Không serialize Java object nội bộ trực tiếp.

---

# 38. Observability Conventions

## OBS-001 — Production Service Is Observable

Tối thiểu:

```text
logs
metrics
traces
health
```

---

## OBS-002 — Telemetry Answers Operational Questions

---

## OBS-003 — Percentiles For Latency

```text
p50
p95
p99
```

---

## OBS-004 — Propagate Trace Context

---

## OBS-005 — Structured Logging

---

## OBS-006 — Never Log Secrets

---

## OBS-007 — Log Exception Once At Appropriate Boundary

---

## OBS-008 — Monitor Critical Invariants

---

# 39. Performance & Scalability Conventions

## PERF-001 — Define Load Parameters

Không nói "scalable" chung chung.

---

## PERF-002 — Quantitative Performance Requirements

Ví dụ:

```text
500 RPS
p95 < 300ms
p99 < 600ms
error < 0.1%
```

---

## PERF-003 — Average Is Not Enough

---

## PERF-004 — Optimization Requires Evidence

```text
profile
trace
metrics
query plan
benchmark
load test
```

---

## PERF-005 — Cache Requires Consistency Definition

Phải trả lời:

```text
source of truth
TTL
invalidation
acceptable staleness
failure behavior
```

---

# 40. Error Handling Conventions

## ERR-001 — Error Taxonomy

```text
validation
business conflict
not found
authorization
dependency
infrastructure
unexpected bug
```

---

## ERR-002 — Stable Error Codes

---

## ERR-003 — Preserve Cause

---

## ERR-004 — Never Expose Stack Trace To Client

---

## ERR-005 — Retryability Explicit

---

# 41. Testing Conventions

## TEST-001 — Test Behavior

---

## TEST-002 — Arrange / Act / Assert

---

## TEST-003 — One Concept Per Test

---

## TEST-004 — FIRST

```text
Fast
Independent
Repeatable
Self-validating
Timely
```

---

## TEST-005 — Deterministic Tests

Không phụ thuộc ngầm:

```text
system clock
network
timezone
execution order
random seed
```

---

## TEST-006 — Boundary Cases

---

## TEST-007 — Bug Fix Requires Regression Test

---

## TEST-008 — Architecture Tests Are First-Class

---

## TEST-009 — Integration Tests For Infrastructure Semantics

---

## TEST-010 — Testcontainers When Real Semantics Matter

---

# 42. Configuration Conventions

## CFG-001 — Config Has Type And Owner

---

## CFG-002 — Unit Is Explicit

Use:

```text
Duration
DataSize
```

---

## CFG-003 — Safe Defaults Or Fail Fast

---

## CFG-004 — No Hidden Configuration

---

# 43. Security Baseline

## SEC-001 — External Input Is Untrusted

---

## SEC-002 — Authentication ≠ Authorization

---

## SEC-003 — Least Privilege

---

## SEC-004 — Secrets From Secret Provider

---

# 44. AI Agent Conventions

## AI-001 — Read Before Edit

Agent MUST đọc:

```text
AGENTS.md
feature README
relevant docs
rules/invariants
tests
implementation
```

---

## AI-002 — Never Guess Business Rules

Nếu ambiguous:

```text
search source of truth
then stop/report if unresolved
```

---

## AI-003 — Smallest Correct Change

---

## AI-004 — Preserve Contracts By Default

Không tự ý break:

```text
API
DB
event
config
public Java API
```

---

## AI-005 — Search Before Create

Trước khi tạo:

```text
util
exception
DTO
interface
policy
```

phải search concept hiện có.

---

## AI-006 — No Generic Future-Proof Abstraction

---

## AI-007 — Follow Existing Pattern Only If Valid

Không blind-copy bad pattern.

---

## AI-008 — Tests Are Part Of Implementation

---

## AI-009 — Run Verification

```bash
./scripts/verify.sh
```

Nếu không chạy được phải nói rõ.

---

## AI-010 — Expose Important Assumptions

---

## AI-011 — No Fabricated APIs

Phải search signature thực.

---

## AI-012 — No Broad Dependency Upgrade Without Request

---

## AI-013 — Keep Docs Synchronized

---

## AI-014 — Do Not Edit Generated Code

---

## AI-015 — Never Hide Uncertainty With Complexity

---

# 45. `AGENTS.md`

Root `AGENTS.md` nên ngắn.

Nó là router, không phải encyclopedia.

Template:

```md
# AI Agent Instructions

## Read Order

Before changing code:

1. Read `docs/README.md`.
2. Read `docs/engineering/CONVENTIONS.md`.
3. Read affected feature `README.md`.
4. Read relevant business rules/invariants.
5. Read relevant contracts/data/reliability docs.
6. Read tests.
7. Read implementation.

## Mandatory Rules

- Follow package-by-feature.
- Domain must not depend on infrastructure.
- Business rules belong to domain.
- Application use cases own orchestration and transactions.
- Controllers contain no business rules.
- Preserve backward compatibility unless explicitly requested.
- Retried mutations must be idempotent or otherwise safely deduplicated.
- Never edit generated code directly.
- Never invent APIs without searching repository.
- Implement the smallest correct change.

## Verification

Run:

`./scripts/verify.sh`

Do not claim success if verification was not executed successfully.

## Full Standards

See:

`docs/engineering/CONVENTIONS.md`
```

---

# 46. Verification

AI và developer cần một entry point:

```bash
./scripts/verify.sh
```

Ví dụ:

```bash
#!/usr/bin/env bash
set -e

./gradlew clean test
./gradlew check
```

Có thể mở rộng:

```text
Spotless
Checkstyle
ArchUnit
OpenAPI validation
Flyway validation
integration tests
dependency checks
security checks
```

---

# 47. Convention Enforcement Matrix

| Rule Group | Enforcement |
|---|---|
| Architecture | ArchUnit |
| Formatting | Spotless |
| Compile Safety | Java compiler |
| Static Analysis | Error Prone / SpotBugs |
| API Contract | OpenAPI validation |
| DB Schema | Flyway/Liquibase |
| Business Rules | Unit/domain tests |
| Persistence | Integration tests |
| Messaging | Contract + idempotency tests |
| Dependency Direction | ArchUnit |
| AI Workflow | AGENTS.md + CI |
| Documentation | PR review + CI lint where possible |

Nguyên tắc:

> Convention càng quan trọng thì càng không nên chỉ tồn tại dưới dạng prose.

---

# 48. Definition of Done

Một change chỉ DONE khi:

```text
Requirement understood
        ↓
Correct feature identified
        ↓
Business rules checked
        ↓
Invariants checked
        ↓
Contracts checked
        ↓
Data/transaction checked
        ↓
Failure/concurrency checked
        ↓
Implementation complete
        ↓
Tests updated
        ↓
Verification passes
        ↓
Docs synchronized
```

Checklist:

- [ ] Correct feature.
- [ ] No duplicate concept.
- [ ] Domain dependency rules respected.
- [ ] API/event/schema compatibility checked.
- [ ] Input validation correct.
- [ ] Transaction boundary correct.
- [ ] Retry/idempotency considered.
- [ ] Error mapping clear.
- [ ] Tests cover behavior and boundaries.
- [ ] No sensitive logs.
- [ ] `./scripts/verify.sh` passes.
- [ ] Documentation updated.

---

# 49. Pull Request Standard

```md
## Why

Business/technical problem.

## What

Minimal implementation summary.

## Business Rules

- BR-...

## Contracts Changed

- API: yes/no
- Database: yes/no
- Event: yes/no
- Config: yes/no

## Failure / Concurrency

- timeout:
- retry:
- idempotency:
- transaction:
- concurrent update:

## Verification

- [ ] unit tests
- [ ] integration tests
- [ ] architecture tests
- [ ] ./scripts/verify.sh

## Observability

Metrics/logs/traces changed?
```

---

# 50. Code Review Order

Review theo thứ tự:

```text
1. Business correctness
2. Data correctness
3. Security
4. Concurrency / transaction
5. Contract compatibility
6. Architecture
7. Failure handling
8. Tests
9. Observability
10. Naming/readability
11. Performance evidence
12. Formatting
```

---

# 51. Documentation Update Matrix

| Change | Review Docs |
|---|---|
| Business condition | `10-domain/business-rules.md` |
| New concept | `domain-model.md`, `glossary.md` |
| State transition | `state-machines.md` |
| New API | `30-contracts/api/` |
| Event | `30-contracts/events/` |
| DB column | `40-data/schema.md` |
| Ownership | `40-data/ownership.md` |
| Transaction | `transaction-model.md` |
| Retry | `timeout-retry.md`, `idempotency.md` |
| Locking | `concurrency.md` |
| Deployment | `80-operations/deployment.md` |
| Architecture decision | ADR |

---

# 52. Documentation Anti-Patterns

## 52.1. Copy code into docs

Docs giải thích intent, không duplicate implementation.

---

## 52.2. Screenshots for searchable knowledge

Ưu tiên:

```text
Markdown
Mermaid
OpenAPI
SQL schema
machine-readable formats
```

---

## 52.3. Duplicate facts

Một fact một source of truth.

---

## 52.4. Docs without owner

Không owner → stale.

---

# 53. Documentation Maturity Model

## Level 0

README only.

## Level 1

README + API.

## Level 2

Architecture + domain overview.

## Level 3

Business rules + contracts + data ownership + operations.

## Level 4

Invariants + failure model + consistency + idempotency + concurrency + ADR.

## Level 5 — AI Ready

Tất cả ở trên cộng:

```text
AI read order
change matrix
stop conditions
verification protocol
source-of-truth map
rule IDs
```

Mục tiêu:

```text
LEVEL 5
```

---

# 54. Golden Path — Developer

```text
Task
 ↓
docs/README
 ↓
context
 ↓
domain
 ↓
contracts/data/reliability
 ↓
architecture
 ↓
tests
 ↓
code
 ↓
verify
 ↓
update docs
```

---

# 55. Golden Path — AI Agent

```text
USER TASK
    ↓
READ-ORDER
    ↓
IDENTIFY DOMAIN
    ↓
FIND RULE / INVARIANT
    ↓
CHECK CONTRACT
    ↓
CHECK DATA OWNERSHIP
    ↓
CHECK FAILURE / CONCURRENCY
    ↓
READ TESTS
    ↓
READ CODE
    ↓
IMPLEMENT MINIMAL CHANGE
    ↓
CHANGE MATRIX
    ↓
VERIFY
    ↓
REPORT
```

---

# 56. Recommended Read Order By Task

## API Task

```text
10-domain
30-contracts/api
20-architecture
50-reliability
tests
code
```

## Database Task

```text
10-domain
40-data
90-decisions
tests
code
```

## Kafka Task

```text
10-domain
30-contracts/events
50-reliability/idempotency
40-data/transaction-model
tests
code
```

## Business Rule Task

```text
10-domain/business-rules
10-domain/invariants
10-domain/state-machines
tests
domain code
```

---

# 57. Template — Authoritative Document

```md
---
title: ...
status: active
owner: ...
last-reviewed: YYYY-MM-DD
review-cycle: 90d
source-of-truth: true
---

# Purpose

Why this document exists.

# Scope

What this document owns.

# Non-Scope

What this document does not own.

# Rules

Authoritative rules.

# Examples

Concrete examples.

# Edge Cases

Known edge cases.

# Verification

How correctness is checked.

# Related Documents

Links.
```

---

# 58. Template — Feature README

```md
# Feature Name

## Responsibility

...

## Domain

See:
docs/10-domain/domain-model.md#...

## Business Rules

- BR-...

## Invariants

- INV-...

## Entry Points

- ...

## Data

Owner:
...

Tables:
- ...

## Events

Published:
- ...

Consumed:
- ...

## External Dependencies

- ...

## Reliability

See:
docs/50-reliability/

## Tests

- ...
```

---

# 59. Template — Business Rule

```md
## BR-XXX-001 — Rule Name

### Statement

...

### Inputs

...

### Preconditions

...

### Success

...

### Failure

...

### Exceptions

...

### Related

- INV-...
- WF-...
```

---

# 60. Template — Invariant

```md
## INV-XXX-001

### Statement

...

### Scope

Strong / eventual consistency.

### Enforcement

...

### Concurrency

...

### Verification

...
```

---

# 61. Template — Workflow

```md
## WF-XXX-001 — Workflow Name

### Trigger

...

### Preconditions

...

### Steps

1. ...
2. ...
3. ...

### Business Rules

- BR-...

### State Changes

...

### Side Effects

...

### Failure Paths

...

### Compensation

...

### Outputs

...
```

---

# 62. Template — Event Contract

```md
# EventName

## Meaning

...

## Producer

...

## Consumers

...

## Delivery

At least once.

## Ordering

...

## Idempotency Key

...

## Compatibility

...

## Schema

...
```

---

# 63. Template — External Integration

```md
# Integration Name

## Purpose

...

## Owner

...

## Protocol

...

## Authentication

...

## Timeout

...

## Retry

...

## Idempotency

...

## Rate Limit

...

## Failure Modes

...

## SLA Assumptions

...
```

---

# 64. Template — ADR

```md
# ADR-XXX — Decision

## Status

Accepted

## Context

...

## Problem

...

## Options

### Option 1

...

### Option 2

...

## Decision

...

## Consequences

### Positive

...

### Negative

...

## Migration

...

## Related

...
```

---

# 65. Template — Failure Model

```md
## Dependency / Component

### Failure F-001

Condition:
...

Detection:
...

Impact:
...

Handling:
...

Retry:
...

Recovery:
...
```

---

# 66. Template — Concurrency Risk

```md
## Race C-001

### Scenario

...

### Risk

...

### Invariant At Risk

INV-...

### Protection

...

### Why Naive Approach Fails

...

### Verification

...
```

---

# 67. Template — Timeout/Retry Matrix

| Dependency | Connect | Read | Retry | Backoff | Idempotent | Notes |
|---|---:|---:|---:|---|---|---|
| ... | ... | ... | ... | ... | ... | ... |

---

# 68. Template — Capacity Model

```md
# Capacity Model

## Traffic

Normal:
...

Peak:
...

## Messaging

Average:
...

Peak:
...

## Payload

p50:
p95:
p99:
max:

## Database

Connections:
Transactions/sec:
Hot queries:

## Growth

Expected 12-month growth:
...
```

---

# 69. Template — SLO

```md
# Service Level Objectives

## Availability

...

## Latency

p95:
p99:

## Error Rate

...

## Business SLO

...
```

---

# 70. Template — Runbook

```md
# Incident / Alert Name

## Symptoms

...

## First Checks

1. ...
2. ...
3. ...

## Diagnosis

...

## Mitigation

...

## Recovery

...

## Escalation

...

## Follow-Up

...
```

---

# 71. Final Engineering Principles

```text
Explicit > Implicit

Local > Scattered

Specific > Generic

Searchable > Hidden

Typed > Magic String

Executable > Documentation-only

Business Meaning > Technical Ceremony

Backward Compatible > Breaking by Default

Idempotent > Hopeful Retry

Measured > Assumed Performance

Deterministic > Ambiguous

Recoverable > Fragile

Small Correct Change > Broad Rewrite
```

---

# 72. Final Mental Model

```text
WHY
 ↓
00-context

WHAT
 ↓
10-domain

HOW STRUCTURED
 ↓
20-architecture

WHAT OTHERS DEPEND ON
 ↓
30-contracts

WHAT DATA MEANS
 ↓
40-data

HOW IT FAILS
 ↓
50-reliability

WHO CAN DO WHAT
 ↓
60-security

HOW WE PROVE IT
 ↓
70-testing

HOW WE RUN IT
 ↓
80-operations

WHY WE CHOSE IT
 ↓
90-decisions

HOW WE CODE
 ↓
engineering/

HOW AI WORKS
 ↓
ai/

IMPLEMENTATION
 ↓
src/

VERIFICATION
 ↓
scripts/
```

---

# 73. Nguyên tắc cuối cùng

Một repository chuyên nghiệp không phải repository có nhiều folder nhất.

Nó là repository mà một engineer hoặc AI Agent mới có thể trả lời nhanh và chính xác:

```text
Hệ thống này giải quyết vấn đề gì?
Business concepts là gì?
Rule nào bắt buộc?
Invariant nào không được phá?
State chuyển thế nào?
Data do ai sở hữu?
Consistency là gì?
Transaction ở đâu?
Contract nào đang tồn tại?
Failure nào có thể xảy ra?
Retry có an toàn không?
Operation có idempotent không?
Ai được phép làm gì?
Làm sao biết thay đổi đúng?
Làm sao rollback?
Tại sao architecture lại như vậy?
```

Nếu phải reverse engineer những câu trả lời này từ code:

```text
repository chưa thực sự AI-ready.
```

Mục tiêu cuối cùng:

> **Documentation explains intent. Tests prove behavior. Code implements both. Automation prevents regression.**
