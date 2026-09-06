# Giá trị bổ sung của Dashboard APM Detail Java — Bản Full

**Ngày lập:** 2026-09-06
**Đối tượng:** DevOps / SRE / Platform / Application team

## Vì sao cần bổ sung

Bản gốc chỉ quan sát tầng ứng dụng (RED metrics, downstream, DB, JVM cơ bản). Nhiều sự cố hạ tầng **không biểu hiện ngay thành 5xx hoặc latency tăng**: pod chết nhưng pod còn lại vẫn gánh được tải, CPU throttling làm tăng p99 dù CPU trung bình thấp, JVM gần cạn heap, hoặc telemetry gián đoạn khiến panel hiện 0 và bị hiểu nhầm là "không có lỗi".

Bản Full bổ sung **4 nhóm panel (22 panel mới)** để phân biệt ba nhóm nguyên nhân có biểu hiện giống nhau: **ứng dụng lỗi thật — hạ tầng có vấn đề — telemetry bị mất**.

---

## 1. Service Health — Kubernetes

| Panel | Giá trị mang lại |
|---|---|
| **301 — Ready replicas (%)** | Phát hiện mất capacity (rollout kẹt, readiness fail, CrashLoop) ngay cả khi RED vẫn đẹp. 50% nghĩa là desired 4 nhưng chỉ còn 2 replica available. |
| **302 — Metric age** | Xác nhận độ tin cậy của toàn bộ số liệu. Trên 300s hoặc No data = telemetry có vấn đề — không được kết luận service khỏe khi panel hiện 0. |
| **303 — Restarts (cả khoảng)** | Phát hiện service "tự hồi phục" sau crash, vốn bị mất dấu vì counter reset sau restart. |
| **304 — OOMKilled (cả khoảng)** | Bắt được OOMKill ở tầng container — JVM bị kernel giết trực tiếp, không kịp ghi log hay tạo 5xx. |
| **305 — Pod readiness** | Phân biệt pod Running nhưng chưa Ready (không nhận traffic), tránh đánh giá quá cao capacity thực tế. |
| **306 — Request rate theo pod** | Phát hiện hot pod / cold pod do sticky session, stale endpoint hoặc load-balancer lệch — điều TPS tổng che mất hoàn toàn. |
| **307 — Pod restarts theo pod** | Nhận diện restart lặp theo chu kỳ, một pod bất thường hay tất cả pod cùng restart do rollout/node incident. |
| **308 — Termination reason** | Chuyển hướng điều tra ngay từ dashboard: `OOMKilled` → memory/limit; `Error` → application log; `ContainerCannotRun` → image/mount/runtime. |

## 2. Container Resources

| Panel | Giá trị mang lại |
|---|---|
| **310 — CPU usage theo pod** | CPU thực tế theo góc nhìn cgroup (gồm sidecar, native thread) — điều `jvm_cpu_recent_utilization_ratio` không phản ánh đủ. |
| **311 — CPU throttling (%)** | Phân biệt "code chậm" với "process bị cgroup không cho chạy". Throttling tăng cùng p95/p99 là bằng chứng CPU limit đang gây latency — nguyên nhân phổ biến khiến tăng limit cải thiện latency mà không cần sửa code. |
| **312 — Memory working set** | Bắt memory ngoài heap (metaspace, direct buffer, thread stack, agent). Working set tăng nhưng heap ổn định → nguyên nhân nằm ngoài heap. |
| **313 — Memory / limit (%)** | Đặt working set cạnh giới hạn thực: 2 GiB an toàn với limit 8 GiB nhưng nguy hiểm với limit 2.2 GiB. Gần 100% = nguy cơ OOMKill cao khi có burst. |

## 3. JVM Saturation

| Panel | Giá trị mang lại |
|---|---|
| **315 — Heap utilization (%) theo pod** | Trả lời nhanh pod đang dùng bao nhiêu % heap tối đa; tìm pod heap pressure thay vì bị trung bình của pod khỏe che lấp. |
| **316 — Old Gen theo pod** | Phát hiện memory leak sớm: đáy Old Gen sau major GC tăng dần = nghi ngờ object retention, trước khi heap chạm trần và pod OOM. |
| **317 — GC pause p99** | Liên hệ trực tiếp spike HTTP p99 với GC pause. Hai đường tăng cùng lúc → GC là nguyên nhân mạnh; GC phẳng → nhìn sang downstream/DB/throttle. |
| **318 — Allocation rate** | Bắt GC pressure do object churn dù heap thấp. Đọc cùng TPS để phân biệt allocation tăng do traffic hay do thay đổi code. |
| **319 — Thread states** | BLOCKED tăng → lock contention; WAITING tăng → chờ downstream/queue; RUNNABLE tăng cùng CPU → workload CPU-bound. |
| **320 — Executor active/pool/queue** | Phát hiện service hết worker trước khi hết CPU/memory: active chạm pool + queue tăng = executor saturation — bottleneck concurrency, không cần vội scale DB hay tăng CPU. |

## 4. SLO & Error Budget

| Thành phần | Giá trị mang lại |
|---|---|
| **Biến `availability_slo`** | Mô phỏng tác động của nhiều mức cam kết (99% → 99.95%, mặc định 99.9%). |
| **322 — Availability SLI** | Diễn đạt error rate theo ngôn ngữ mục tiêu dịch vụ, so trực tiếp với SLO; giữ No data khi mất telemetry để tránh báo 100% giả. |
| **323 — Latency SLI** | Trả lời "bao nhiêu % request đạt mục tiêu latency đã cam kết" — điều p95/p99 không trả lời trực tiếp. |
| **324 — Error budget remaining (30d)** | Chuyển SLO thành ngân sách hữu hạn: SLO 99.9% cho phép 0.1% lỗi; đã tiêu 0.05% → còn 50% budget. Hỗ trợ quyết định tiếp tục release hay tập trung ổn định. |
| **325 — Burn rate đa cửa sổ (5m/30m/1h/6h)** | Cho biết budget đang bị tiêu nhanh gấp bao nhiêu lần cho phép. Cửa sổ ngắn cao + dài thấp = spike; cả hai cùng cao = incident bền vững. Giảm cả false alert với spike ngắn lẫn phát hiện chậm với incident lớn. |

---

## Quy trình điều tra sự cố với các panel mới

Khi HTTP p95 tăng, dashboard Full cho phép thu hẹp nguyên nhân theo thứ tự:

1. **Metric age** — dữ liệu còn cập nhật không?
2. **Ready replicas / Pod readiness** — có thiếu capacity không?
3. **Request rate theo pod** — có hot pod / load imbalance không?
4. **CPU throttling / Memory-limit** — container có bị giới hạn tài nguyên không?
5. **GC pause / Allocation / Thread / Executor** — JVM có saturation không?
6. **Downstream/DB** (phần APM gốc) — nguyên nhân bên ngoài?
7. **SLI / Burn rate** — mức độ ảnh hưởng tới SLO?

Nhờ chuỗi này, dashboard không chỉ báo "latency đang cao" mà chỉ ra hỏng ở đâu: **telemetry → Kubernetes → container → JVM → dependency → tác động SLO**.

## Kết luận

Giá trị cốt lõi của bản Full là nâng dashboard từ APM thuần túy lên tương quan **Application + Kubernetes + Container + JVM saturation + SLO**. Các bổ sung hữu ích nhất cho trực vận hành: **Metric age, Ready replicas/readiness, Restart/OOMKilled, CPU throttling, Memory/limit và Multi-window burn rate**.
