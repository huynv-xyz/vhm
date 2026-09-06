# Báo cáo so sánh APM Detail Java: bản gốc và bản Full

Ngày lập: 2026-09-06  
Đối tượng: DevOps / SRE / Platform / Application team

## 1. Phạm vi so sánh

| Nội dung | Bản gốc | Bản Full |
|---|---|---|
| File | `apm-detail-original.json` | `apm-detail-java-full.json` |
| Title | `APM Detail — Java` | `VHM - Test` |
| Dashboard UID | `apm-detail-java` | `vhm-test-java-full` |
| Tổng số object panel/row | 38 | 64 |
| Số row | 5 | 9 |
| Số panel dữ liệu | 33 | 55 |

Bản Full giữ toàn bộ panel ID của bản gốc và bổ sung 26 object mới, gồm 4 row và 22 panel dữ liệu. Tuy nhiên, bản Full không chỉ thêm panel: một số query, biến, mô tả, field configuration và interaction của panel cũ cũng đã thay đổi.

## 2. Tóm tắt điều hành

Bản gốc tập trung vào APM ở tầng ứng dụng: RED metrics, downstream, DB, JVM cơ bản, trace và log. Bản Full mở rộng sang bốn lớp vận hành còn thiếu:

1. Kubernetes service health: replica, readiness, restart, OOM và độ mới của telemetry.
2. Container resources: CPU, throttling, working set và memory limit.
3. JVM saturation: heap utilization, Old Gen, GC pause, allocation, thread state và executor queue.
4. SLO/error budget: availability, latency SLI, ngân sách lỗi 30 ngày và burn rate.

Giá trị chính của phần bổ sung là giúp phân biệt ba nhóm nguyên nhân có biểu hiện APM giống nhau:

- Ứng dụng lỗi thật: 5xx/latency tăng.
- Pod hoặc container có vấn đề: thiếu replica, restart, OOM, CPU throttling.
- Telemetry bị mất: metric age tăng hoặc No data.

## 3. Phân tích chi tiết từng thành phần được bổ sung

### 3.1. Vì sao RED metrics của bản gốc là chưa đủ?

RED trả lời tốt ba câu hỏi ở tầng request: có bao nhiêu request, bao nhiêu lỗi và request mất bao lâu. Nhưng nhiều sự cố hạ tầng không biểu hiện ngay thành 5xx:

- Một pod chết nhưng các pod còn lại tiếp tục nhận traffic: TPS tổng vẫn bình thường, error rate vẫn bằng 0, nhưng hệ thống đã mất redundancy.
- Pod bị CPU throttle: CPU trung bình có thể không cao nhưng request bị dừng ngắn liên tục, làm p95/p99 tăng.
- JVM gần cạn heap hoặc executor hết thread: request chưa lỗi ngay nhưng service đang ở trạng thái saturation.
- Agent hoặc Prometheus scrape bị gián đoạn: panel có thể hiện No data hoặc 0 và bị hiểu nhầm là hệ thống không có lỗi.
- Availability hiện tại nhìn tốt nhưng tốc độ phát sinh lỗi đang tiêu error budget quá nhanh.

Vì vậy bản Full thêm các lớp Kubernetes, container, JVM saturation và SLO để trả lời cả “service đang hỏng vì đâu” và “service còn bao nhiêu dư địa trước khi hỏng”.

### 3.2. Nhóm Service Health — Kubernetes

#### Panel 301 — Ready replicas (%)

**Vấn đề cần giải quyết:** Bản gốc chỉ nhìn request đi vào các pod còn sống. Nếu Deployment mong muốn 4 replica nhưng chỉ còn 2 replica available, RED vẫn có thể đẹp vì 2 pod còn lại đang gánh được tải hiện tại. Tuy nhiên hệ thống đã mất một nửa capacity và có nguy cơ sập khi traffic tăng.

**Cách tính:**

```promql
100 * available_replicas / desired_replicas
```

Dashboard dùng `kube_deployment_status_replicas_available` làm tử số và `kube_deployment_spec_replicas` làm mẫu số. `clamp_min(..., 1)` tránh chia cho 0 khi desired replica được scale về 0.

**Cách đọc:**

- 100%: số replica available đúng bằng cấu hình mong muốn.
- 50%: ví dụ desired là 4 nhưng chỉ có 2 replica available.
- No data: selector không tìm thấy Deployment hoặc kube-state-metrics không có dữ liệu; không được diễn giải thành service khỏe.

**Giúp ích khi vận hành:** Đây là tín hiệu nhanh để phát hiện rollout kẹt, readiness probe fail, thiếu tài nguyên để schedule hoặc pod CrashLoop. Khi latency tăng nhưng traffic không đổi, ready replica giảm là bằng chứng tải đang bị dồn lên ít pod hơn.

**Hành động tiếp theo:** kiểm tra rollout status, pod events, readiness probe, quota và khả năng schedule của node.

#### Panel 302 — Metric age

**Vấn đề cần giải quyết:** Một panel lỗi hiện 0 có thể mang hai nghĩa hoàn toàn khác nhau: thật sự không có lỗi, hoặc telemetry đã ngừng cập nhật. Nếu không có tín hiệu freshness, người trực vận hành dễ kết luận sai rằng service đang khỏe.

**Cách tính:**

```promql
time() - max(timestamp(http_server_request_duration_seconds_count{...}))
```

Prometheus vẫn scrape counter dù giá trị counter không đổi, nên panel đo tuổi sample chứ không phụ thuộc việc service có traffic mới hay không.

**Cách đọc:**

- Gần 0–60 giây: scrape đang cập nhật bình thường, tùy scrape interval.
- Trên 120 giây: cần kiểm tra scrape delay hoặc exporter/agent.
- Trên 300 giây hoặc No data: có khả năng target mất, selector sai, agent chết hoặc Prometheus không scrape được.

**Giúp ích khi vận hành:** Đây là panel xác nhận độ tin cậy của toàn bộ số liệu APM. Trước khi kết luận “0 request” hoặc “0 lỗi”, DevOps có thể nhìn Metric age để chắc chắn dữ liệu còn sống.

**Hành động tiếp theo:** mở Prometheus Targets, kiểm tra ServiceMonitor/PodMonitor, endpoint metrics, OTel Java agent và network policy.

#### Panel 303 — Restarts — cả khoảng

**Vấn đề cần giải quyết:** Counter HTTP/JVM của process mới sẽ bắt đầu lại sau restart. Nếu chỉ nhìn đường metric sau khi pod lên lại, sự cố vừa xảy ra có thể biến mất khỏi dashboard.

**Cách tính:** tổng `increase(kube_pod_container_status_restarts_total[$__range])` của các container thuộc service trong đúng khoảng thời gian đang xem. Query chỉ zero-fill khi `kube_pod_info` xác nhận pod đúng selector có tồn tại; nếu không tìm thấy pod thì vẫn nên xem là vấn đề selector/telemetry.

**Cách đọc:**

- 0: các pod được chọn tồn tại và không restart trong khoảng.
- Lớn hơn 0: đã có container restart; cần xem panel chi tiết theo pod.
- No data: không tìm thấy pod/KSM metric, không đồng nghĩa không restart.

**Giúp ích khi vận hành:** Cho phép phát hiện service “tự hồi phục” sau crash. Đây là tín hiệu quan trọng khi người dùng báo lỗi ngắn nhưng biểu đồ hiện tại đã trở lại bình thường.

**Hành động tiếp theo:** đối chiếu thời điểm restart với deploy, logs trước khi process chết, probe failures và node events.

#### Panel 304 — OOMKilled — cả khoảng

**Vấn đề cần giải quyết:** OOMKill xảy ra ở tầng container. JVM có thể không kịp ghi log, không tạo exception và cũng không để lại 5xx rõ ràng vì process bị kernel giết trực tiếp.

**Cách tính:** `max_over_time` trên `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` trong khoảng dashboard, sau đó lấy giá trị lớn nhất. Khi metric từng bằng 1 trong khoảng, stat sẽ báo có OOMKilled.

**Giúp ích khi vận hành:** Rút ngắn điều tra từ “service tự nhiên biến mất” sang đúng hướng memory limit/container OOM. Panel này phải được đọc cùng working set, memory limit và JVM heap.

**Giới hạn:** Kube-state-metrics chỉ phản ánh termination reason mà Kubernetes còn giữ. Đây không phải một event store lâu dài; retention và việc container terminate nhiều lần với reason khác nhau có thể ảnh hưởng khả năng truy vết lịch sử xa.

**Hành động tiếp theo:** kiểm tra `kubectl describe pod`, container limit, exit code 137, heap sizing, direct buffer, metaspace, số thread và native memory.

#### Panel 305 — Pod readiness

**Vấn đề cần giải quyết:** Replica có thể đang Running nhưng chưa Ready, vì vậy không được đưa vào endpoint nhận traffic. Chỉ đếm pod Running sẽ đánh giá quá cao capacity thực tế.

**Cách tính:** lấy `kube_pod_status_ready{condition="true"}` và nhóm theo namespace/pod. Giá trị 1 nghĩa là pod Ready, 0 nghĩa là chưa Ready.

**Giúp ích khi vận hành:** Cho biết pod nào fail readiness và thời điểm mất Ready. Khi request rate theo instance lệch, panel này giúp xác định pod không nhận traffic vì chưa Ready thay vì do load balancer lệch.

**Hành động tiếp theo:** kiểm tra readiness probe response, dependency lúc startup, cấu hình timeout/failureThreshold và application startup time.

#### Panel 306 — Request rate theo pod/instance

**Vấn đề cần giải quyết:** TPS tổng của service che mất phân phối tải. Một pod có thể nhận phần lớn request trong khi pod khác gần như không nhận gì do stale endpoint, session affinity, topology hoặc load-balancer configuration.

**Cách tính:** `rate(http_server_request_duration_seconds_count)` rồi `sum by (instance)`.

**Cách đọc:**

- Các instance có lưu lượng tương đối đồng đều: cân bằng tải hoạt động như kỳ vọng.
- Một instance cao vượt trội: kiểm tra sticky session, topology-aware routing hoặc endpoint distribution.
- Một instance vẫn có JVM metric nhưng request rate bằng 0: pod có thể không nằm trong Service endpoints hoặc readiness có vấn đề.

**Giúp ích khi vận hành:** Phát hiện hot pod và cold pod, vốn bị mất hoàn toàn khi chỉ xem TPS tổng theo endpoint.

**Giới hạn:** Label `instance` có thể là pod IP hoặc scrape endpoint, không nhất thiết là tên pod. Muốn map chắc chắn cần bổ sung label pod/resource attribute nhất quán khi scrape.

#### Panel 307 — Pod restarts

**Vấn đề cần giải quyết:** Stat tổng chỉ báo “đã có restart”; DevOps vẫn cần biết pod/container nào restart và restart vào giai đoạn nào.

**Cách tính:** `increase(...[$rateinterval])` và nhóm theo namespace, pod, container. Mỗi điểm là số lần restart trong cửa sổ trượt `$rateinterval`, không phải một event tức thời.

**Giúp ích khi vận hành:** Nhận diện restart lặp theo chu kỳ, chỉ một pod bất thường hoặc tất cả pod cùng restart do rollout/node incident.

**Hành động tiếp theo:** chọn pod có spike, truy log ở thời điểm ngay trước spike và đối chiếu termination reason.

#### Panel 308 — Termination reason

**Vấn đề cần giải quyết:** Số lần restart không nói nguyên nhân. Restart do OOM khác hoàn toàn restart do application error hoặc container không khởi chạy được.

**Cách tính:** lấy trạng thái gần nhất trong `$rateinterval`, nhóm theo namespace, pod, container và reason; giới hạn vào `OOMKilled`, `Error`, `ContainerCannotRun`.

**Giúp ích khi vận hành:** Chuyển hướng xử lý ngay từ dashboard:

- `OOMKilled`: điều tra memory và limit.
- `Error`: điều tra exit code/application log.
- `ContainerCannotRun`: điều tra image, command, permission, mount hoặc runtime.

**Giới hạn:** CrashLoopBackOff thường là waiting reason, không phải last terminated reason. Muốn bao phủ đầy đủ nên có thêm metric waiting reason hoặc kiểm tra trực tiếp pod status/events.

#### Phạm vi selector Kubernetes

Các panel pod sử dụng:

```text
^${service_name:regex}(-deployment)?-.*$
```

Các panel Deployment sử dụng:

```text
^${service_name:regex}(-deployment)?$
```

Việc neo regex giúp không lấy nhầm pod/service có tên chứa chuỗi của service đang chọn, ví dụ `miniportal-vhm-agent-api-*`. Dashboard hỗ trợ hai quy ước phổ biến: `<service>-<hash>` và `<service>-deployment-<hash>`.

Đây vẫn là mapping theo tên. Nếu `service.name`, Prometheus `job` và Kubernetes workload name không cùng quy ước, cần thêm label chung hoặc một biến workload riêng; không nên nới regex thành substring vì sẽ làm dữ liệu sai scope.

### 3.3. Nhóm Container Resources

#### Panel 310 — Container CPU usage theo pod

**Vấn đề cần giải quyết:** `jvm_cpu_recent_utilization_ratio` chỉ phản ánh JVM và phụ thuộc metric runtime. Nó không thể hiện đầy đủ CPU của sidecar, native thread hoặc toàn bộ container theo góc nhìn cgroup.

**Cách tính:** tốc độ tăng của `container_cpu_usage_seconds_total`, cộng theo namespace/pod. Đơn vị `cores`: giá trị 1 tương đương sử dụng trung bình một CPU core.

**Giúp ích khi vận hành:** So sánh CPU thực tế giữa các pod, phát hiện hot pod, CPU tăng cùng traffic, hoặc CPU tăng dù traffic không tăng do GC/background job.

**Cách sử dụng:** đọc cùng request rate theo instance và GC time. CPU tăng tỷ lệ với TPS thường là tải thật; CPU tăng cùng GC nhưng TPS phẳng thường là memory pressure; một pod CPU cao riêng lẻ thường là phân phối tải hoặc workload lệch.

**Giới hạn:** Panel hiện chưa vẽ CPU request/limit, vì vậy “0.8 core” chưa nói được có gần limit hay không. Cần đối chiếu resource spec hoặc bổ sung panel CPU usage/limit.

#### Panel 311 — CPU throttling theo pod (%)

**Vấn đề cần giải quyết:** Container có thể bị throttle dù CPU usage trung bình nhìn không cao. Các burst ngắn chạm CPU limit bị kernel tạm dừng, làm request latency tăng nhưng biểu đồ CPU trung bình làm mượt tín hiệu này.

**Cách tính:**

```promql
100 * rate(throttled_periods_total) / rate(periods_total)
```

Mẫu số được `clamp_min` để tránh chia cho giá trị quá nhỏ.

**Cách đọc:** Đây là tỷ lệ kỳ CFS bị throttle, không phải phần trăm CPU time bị mất. Giá trị tăng đồng thời với p95/p99 là bằng chứng mạnh rằng CPU limit đang ảnh hưởng latency.

**Giúp ích khi vận hành:** Phân biệt “code xử lý chậm” với “process có việc để chạy nhưng bị cgroup không cho chạy”. Đây là một trong các nguyên nhân phổ biến khiến tăng CPU request/limit cải thiện latency mà không thay đổi code.

**Hành động tiếp theo:** kiểm tra CPU limit/request, HPA, burst pattern, GC CPU và cân nhắc điều chỉnh hoặc bỏ CPU limit theo chính sách platform.

#### Panel 312 — Memory working set theo pod

**Vấn đề cần giải quyết:** Java heap không phải toàn bộ memory của container. Metaspace, code cache, direct buffer, native allocation, thread stack và OTel agent đều nằm ngoài heap nhưng vẫn bị tính vào memory cgroup.

**Cách tính:** cộng `container_memory_working_set_bytes` của các application containers theo pod. Working set gần với phần memory khó reclaim và thường hữu ích hơn total usage để đánh giá nguy cơ OOM.

**Giúp ích khi vận hành:** Nếu working set tăng nhưng heap ổn định, nguyên nhân có thể nằm ngoài heap. Nếu cả heap và working set cùng tăng, cần điều tra object retention/leak trong JVM.

**Hành động tiếp theo:** đối chiếu heap/Old Gen, direct memory, số thread, native memory tracking và memory limit.

#### Panel 313 — Container memory / limit (%)

**Vấn đề cần giải quyết:** Số byte working set chỉ có ý nghĩa khi đặt cạnh giới hạn của container. 2 GiB có thể an toàn với limit 8 GiB nhưng nguy hiểm với limit 2.2 GiB.

**Cách tính:** tổng working set của pod chia tổng memory limit của các container tương ứng, nhân 100.

**Cách đọc:**

- Dưới khoảng 70% và ổn định: còn dư địa tương đối.
- Tiến dần tới 90%: cần điều tra xu hướng và đỉnh sử dụng.
- Gần 100%: nguy cơ OOMKill cao khi có burst hoặc GC/native allocation.

Các mức trên chỉ là hướng dẫn ban đầu; ngưỡng production phải dựa trên workload và headroom mong muốn.

**Giúp ích khi vận hành:** Cho biết trực tiếp pod nào gần giới hạn nhất và hỗ trợ quyết định tăng limit, giảm heap, sửa leak hoặc scale ngang.

**Giới hạn:** Pod/container không đặt limit sẽ không có mẫu số phù hợp; panel có thể No data hoặc không phản ánh container unlimited. Cần kiểm tra metric `kube_pod_container_resource_limits` và label `unit="byte"` trong cluster.

### 3.4. Nhóm JVM Saturation

#### Panel 315 — Heap utilization (%) theo pod

**Vấn đề cần giải quyết:** Bản gốc hiển thị số byte theo pool nhưng không trả lời nhanh pod đang dùng bao nhiêu phần trăm heap tối đa. Khi nhiều pod bị gộp, một pod gần cạn heap còn có thể bị trung bình của pod khác che lấp.

**Cách tính:** tổng heap used chia tổng heap limit theo `instance`, nhân 100; bỏ các limit không dương.

**Giúp ích khi vận hành:** So sánh trực tiếp các pod và tìm pod có heap pressure. Heap utilization cao cùng GC time cao là dấu hiệu heap chật; heap cao nhưng GC không tăng có thể là cache/retained objects ổn định.

**Giới hạn:** Cách exporter biểu diễn memory pool và limit có thể khác theo garbage collector/JDK. Cần đối chiếu tổng limit với `-Xmx` hoặc container-aware JVM setting trước khi đặt alert.

#### Panel 316 — Old Gen theo pod

**Vấn đề cần giải quyết:** Tổng heap dạng răng cưa là hành vi bình thường và không đủ để nhận diện memory leak. Tín hiệu cần quan sát là đáy Old Gen sau các lần major GC có tăng dần hay không.

**Cách tính:** lọc các heap pool có tên chứa `old` hoặc `tenured`, nhóm theo instance và pool.

**Cách đọc:**

- Tăng rồi giảm sau GC: vòng đời object bình thường.
- Đáy sau GC tăng dần qua thời gian: nghi ngờ object retention hoặc memory leak.
- Old Gen tăng cùng allocation rate: workload tạo nhiều object sống lâu hoặc promotion tăng.

**Giúp ích khi vận hành:** Cho phép phát hiện leak sớm trước khi heap chạm trần và pod OOM/restart.

**Giới hạn:** Tên pool phụ thuộc GC/JDK. ZGC, Shenandoah hoặc collector mới có thể không chứa `old/tenured`, khiến panel No data dù heap metric vẫn tồn tại.

#### Panel 317 — GC pause p99 theo pod

**Vấn đề cần giải quyết:** Panel GC `% thời gian` cho biết GC tiêu bao nhiêu tổng thời gian nhưng không cho biết một lần pause tệ nhất dài bao lâu. Một số pause hiếm nhưng dài có thể trực tiếp tạo tail latency.

**Cách tính:** `histogram_quantile(0.99, ...)` trên bucket thời gian GC, giữ nhãn instance và GC name.

**Giúp ích khi vận hành:** Liên hệ spike HTTP p99 với GC pause p99. Nếu hai đường tăng cùng thời điểm, GC là ứng viên nguyên nhân mạnh; nếu HTTP p99 tăng nhưng GC pause phẳng, cần nhìn downstream, DB hoặc CPU throttle.

**Giới hạn:** Chỉ có dữ liệu nếu agent/exporter phát `jvm_gc_duration_seconds_bucket`. Có `_sum` không đồng nghĩa có `_bucket`; No data ở đây có thể là thiếu histogram chứ không phải không có GC.

#### Panel 318 — Allocation rate

**Vấn đề cần giải quyết:** Heap utilization chỉ là ảnh chụp lượng memory đang giữ. Một service có heap thấp nhưng cấp phát object cực nhanh vẫn tạo GC pressure và tiêu CPU lớn.

**Cách tính:** `rate(jvm_gc_memory_allocated_bytes_total)` theo instance, đơn vị byte/giây.

**Giúp ích khi vận hành:** Phát hiện request path tạo quá nhiều object, serialization/deserialization nặng hoặc batch job tạo allocation burst. Đọc cùng TPS để phân biệt allocation tăng do traffic tăng hay allocation/request tăng do thay đổi code.

**Hành động tiếp theo:** dùng JFR/allocation profiler, so sánh trước/sau deploy và tối ưu object churn nếu allocation tăng bất thường.

**Giới hạn:** Đây thường là Micrometer metric, không phải mọi cấu hình OTel Java agent đều có. No data cần được ghi nhận là “metric chưa bật”.

#### Panel 319 — Thread states

**Vấn đề cần giải quyết:** Tổng thread count tăng chỉ cho biết có nhiều thread hơn, không cho biết thread đang làm gì. Nhiều RUNNABLE có ý nghĩa khác nhiều BLOCKED hoặc WAITING.

**Cách tính:** tổng `jvm_threads_states_threads` theo instance và state.

**Giúp ích khi vận hành:**

- BLOCKED tăng: nghi ngờ lock contention/deadlock.
- WAITING/TIMED_WAITING tăng: có thể chờ downstream, queue hoặc timer; cần đặt trong bối cảnh thread pool.
- RUNNABLE tăng cùng CPU: workload CPU-bound hoặc busy loop.

**Hành động tiếp theo:** lấy thread dump/JFR tại đúng thời điểm spike. Metric chỉ chỉ ra trạng thái tổng quát, không thay thế stack trace.

#### Panel 320 — Executor active / pool / queue

**Vấn đề cần giải quyết:** Service có thể hết worker trước khi JVM hết CPU hoặc memory. Khi active thread chạm pool size và queue tăng, công việc mới phải chờ, làm latency tăng rồi timeout.

**Cách tính:** vẽ đồng thời `executor_active_threads`, `executor_pool_size_threads` và `executor_queued_tasks`, giữ tên executor trong legend.

**Cách đọc:**

- Active thấp hơn pool và queue gần 0: còn capacity.
- Active chạm pool, queue tăng: executor saturation.
- Queue tăng trong khi downstream latency tăng: worker có thể đang bị giữ bởi call blocking.

**Giúp ích khi vận hành:** Chỉ ra bottleneck ở application concurrency thay vì vội scale DB hoặc tăng CPU. Panel cũng giúp đánh giá pool-size và queue-size có phù hợp workload hay không.

**Giới hạn:** Executor phải được bind/observe bằng Micrometer. Custom executor không bind sẽ không xuất hiện.

### 3.5. Nhóm SLO & Error Budget

#### Biến `availability_slo`

Bản Full thêm biến availability objective với các lựa chọn 99%, 99.5%, 99.9% và 99.95%; mặc định 99.9%. Với SLO 99.9%, error budget availability là 0.1% request trong chu kỳ.

Việc đưa SLO thành biến giúp DevOps mô phỏng tác động của nhiều mức cam kết. Tuy nhiên SLO production chính thức nên được quản lý bằng configuration/recording rule, không nên phụ thuộc hoàn toàn vào lựa chọn thủ công trên dashboard.

#### Panel 322 — Availability SLI

**Vấn đề cần giải quyết:** Error rate chỉ cho biết phần trăm lỗi; SLI availability diễn đạt cùng dữ liệu theo ngôn ngữ mục tiêu dịch vụ, giúp so trực tiếp với SLO.

**Cách tính:**

```text
100 × (1 - 5xx_requests / total_requests)
```

Nếu có total request nhưng không có series 5xx, tử số lỗi được coi là 0. Nếu mất cả total metric, panel vẫn No data để tránh báo availability 100% giả.

**Giúp ích khi vận hành:** Cho biết availability biến động theo thời gian, đối chiếu với deploy/downstream incident và làm đầu vào cho burn-rate investigation.

**Giới hạn chính sách:** Công thức coi 4xx là request hợp lệ vì thường là lỗi client. Nếu một số 4xx thực chất do backend hoặc hợp đồng nghiệp vụ yêu cầu tính timeout/429 vào lỗi, công thức cần điều chỉnh.

#### Panel 323 — Latency SLI

**Vấn đề cần giải quyết:** p95/p99 trả lời “ngưỡng latency là bao nhiêu”, nhưng không trả lời trực tiếp “bao nhiêu phần trăm request đạt mục tiêu latency đã cam kết”.

**Cách tính:** số request rơi vào histogram bucket `le="$slo_bucket"` chia bucket `+Inf`, nhân 100.

**Ví dụ:** Nếu `$slo_bucket=0.5`, giá trị 98% nghĩa là 98% request hoàn thành dưới hoặc bằng 0.5 giây trong cửa sổ rate.

**Giúp ích khi vận hành:** Cho phép đánh giá compliance của mục tiêu latency và theo dõi tác động của tối ưu/incident bằng một tỷ lệ dễ so với SLO.

**Giới hạn:** `$slo_bucket` phải đúng một boundary thật của histogram. Chọn 0.5 khi metric không có `le="0.5"` sẽ trả No data; histogram bucket cũng phải được aggregate đúng theo `le`.

#### Panel 324 — Error budget remaining — 30d

**Vấn đề cần giải quyết:** Availability hiện tại 99.95% nghe có vẻ tốt nhưng chưa cho biết đã tiêu bao nhiêu phần lỗi được phép. Error budget chuyển SLO thành một ngân sách hữu hạn để quyết định tốc độ release và mức độ ưu tiên reliability work.

**Cách tính khái quát:**

```text
remaining = 100 × (1 - actual_error_ratio / allowed_error_ratio)
allowed_error_ratio = 1 - availability_slo
```

Kết quả được clamp về khoảng 0–100%.

**Ví dụ:** SLO 99.9% cho phép 0.1% lỗi. Nếu trong 30 ngày đã phát sinh 0.05% lỗi thì còn khoảng 50% error budget. Nếu tỷ lệ lỗi đạt hoặc vượt 0.1%, budget remaining về 0%.

**Giúp ích khi vận hành:** Hỗ trợ quyết định có nên tiếp tục release rủi ro hay chuyển trọng tâm sang ổn định hệ thống. Nó cũng giúp diễn đạt reliability bằng một con số có thể trao đổi giữa DevOps, dev và product.

**Giới hạn:** Query cần ít nhất 30 ngày dữ liệu Prometheus liên tục. Nếu retention ngắn hơn, service mới chạy hoặc label/job thay đổi giữa chu kỳ, kết quả không đại diện đủ 30 ngày.

#### Panel 325 — Availability burn rate

**Vấn đề cần giải quyết:** Error budget remaining là chỉ báo chậm và nhìn về quá khứ. Burn rate trả lời budget đang bị tiêu nhanh gấp bao nhiêu lần tốc độ cho phép ngay lúc này.

**Cách tính:** error ratio trong từng cửa sổ chia cho allowed error ratio `(1 - availability_slo)`. Dashboard vẽ bốn cửa sổ 5m, 30m, 1h và 6h.

**Cách đọc:**

- Burn rate = 0: không có 5xx trong cửa sổ.
- Burn rate = 1: đang tiêu budget đúng tốc độ trung bình cho phép.
- Burn rate > 1: đang tiêu nhanh hơn kế hoạch.
- Cửa sổ ngắn tăng nhưng cửa sổ dài thấp: spike ngắn, cần quan sát.
- Cả cửa sổ ngắn và dài cùng cao: incident bền vững, mức độ nghiêm trọng cao hơn.

**Giúp ích khi vận hành:** Multi-window burn rate giảm hai lỗi phổ biến của alert tĩnh: báo động quá nhạy với spike ngắn và phát hiện quá chậm với incident lớn. Đây là cơ sở tốt để xây alert cặp cửa sổ nhanh/chậm.

**Giới hạn:** Panel mới chỉ visualization, chưa có threshold/alert rule hoàn chỉnh. DevOps cần xác định burn-rate threshold và `for` duration theo SLO/window chính thức.

### 3.6. Cách các nhóm bổ sung hỗ trợ điều tra sự cố

Ví dụ khi HTTP p95 tăng:

1. Kiểm tra Metric age để chắc chắn dữ liệu đang cập nhật.
2. Kiểm tra Ready replicas và Pod readiness để loại trừ thiếu capacity.
3. Kiểm tra request rate theo instance để tìm hot pod/load imbalance.
4. Kiểm tra CPU throttling và container memory/limit.
5. Kiểm tra GC pause, allocation rate, thread states và executor queue.
6. Nếu tầng local bình thường, kiểm tra downstream HTTP/RPC/DB của phần APM gốc.
7. Kiểm tra Availability/Latency SLI và burn rate để đánh giá mức ảnh hưởng tới SLO.

Nhờ chuỗi này, dashboard Full không chỉ thông báo “latency đang cao” mà giúp thu hẹp nguyên nhân theo thứ tự telemetry → Kubernetes → container → JVM → dependency → tác động SLO.

## 4. Thay đổi trên các panel có sẵn

### 4.1. Error rate được tính theo đúng khoảng thời gian

Trong bảng RED theo endpoint, bản gốc tính error rate bằng raw cumulative counter. Kết quả phản ánh từ lúc process khởi động, không đúng với time range người dùng đang chọn.

Bản Full đổi tử số và mẫu số sang `increase(...[$__range])`. Nhờ đó error rate phản ánh đúng khoảng dashboard.

### 4.2. Zero-fill có điều kiện

Các tỷ lệ 4xx/5xx và downstream status dùng zero-fill ở tử số dựa trên chính tập total request. Khi có traffic nhưng không có series lỗi, panel hiển thị 0%. Khi mất cả total traffic/telemetry, panel vẫn No data thay vì tạo `vector(0)` vô điều kiện.

Điều này tránh hiểu nhầm “mất metric” thành “0 lỗi”. Riêng Request rate ở bản Full đã bỏ `or vector(0)` để No data lộ rõ.

### 4.3. Response Status selector dùng HTTP semantic conventions

Biến `status_code` chuyển từ spanmetrics `status_code` sang:

```promql
label_values(
  http_server_request_duration_seconds_count{cluster=~"$cluster", job=~"$service_name"},
  http_response_status_code
)
```

Panel `Status code theo thời gian` sử dụng biến này trực tiếp. Giá trị hiển thị là mã HTTP thực tế như 200, 401, 500 thay vì span status `STATUS_CODE_OK/ERROR`.

### 4.4. JVM được tách theo instance

Hai panel hiện có được mở rộng:

- Heap: từ gộp theo memory pool sang `instance + memory pool`.
- GC time: từ gộp theo collector sang `instance + collector`.

Ý nghĩa là có thể phát hiện một pod bất thường thay vì bị giá trị của các pod khỏe che lấp.

### 4.5. TraceQL lọc bằng `span.http.route`

Bản gốc so route với tên span. Bản Full chuyển sang attribute `span.http.route`, phù hợp hơn với selector HTTP route. Giới hạn là trace không có `http.route` hoặc chỉ có route tổng quát `/*` sẽ không cung cấp độ chi tiết endpoint mong muốn.

## 5. Thay đổi biến dashboard

| Biến | Thay đổi | Ý nghĩa |
|---|---|---|
| `cluster` | Lấy từ HTTP server metric thay vì spanmetrics | Service xuất HTTP metric vẫn xuất hiện dù trace/spanmetrics chưa có. |
| `service_name` | Lấy từ label `job`; bỏ `All`; mặc định `vhm-agent-api` | Dashboard trở thành dashboard chi tiết một service, tránh gộp nhầm dữ liệu nhiều ứng dụng. |
| `status_code` | Lấy từ `http_response_status_code` | Đồng nhất với các panel HTTP stable semconv. |
| `availability_slo` | Biến mới, mặc định 99.9% | Dùng cho error budget và burn rate. |
| `predictive_interval`, `predictive_offset` | Loại bỏ | Hai biến này không được panel nào sử dụng. |

Namespace vẫn cho phép `All`. Khi điều tra Kubernetes nên chọn namespace cụ thể để tránh cùng tên service ở nhiều namespace bị gộp chung.

## 6. Những phần không được bổ sung hoặc không thể sửa bằng dashboard

### DB telemetry

Các panel DB connection pool, wait time và DB operation của bản Full sử dụng cùng họ metric và selector `job=~"$service_name"` như bản gốc. Bản Full không tạo thêm DB telemetry.

Nếu chọn riêng `vhm-agent-api` mà DB panel No data thì một trong các điều kiện sau đang xảy ra:

- Service không phát DB semantic-convention metrics.
- Metric được xuất dưới một `job` khác.
- Chỉ connection-pool metrics được bật, chưa có stable DB operation metrics.
- Histogram wait/operation chưa được cấu hình xuất bucket.

Việc bảng gốc hiển thị dữ liệu khi `Service Name = All` chỉ chứng minh cluster có DB metrics của một hoặc nhiều service; không chứng minh số liệu đó thuộc `vhm-agent-api`. Không nên gán DB job của service khác chỉ để làm panel có số.

### Mapping service sang Kubernetes workload

HTTP/JVM metric dùng label `job`, còn kube-state-metrics dùng `deployment`/`pod`. Dashboard đang dùng quy ước tên để nối hai nguồn. Đây là heuristic, không phải join bằng một định danh chuẩn. Giải pháp lâu dài là chuẩn hóa resource attributes/Prometheus labels để có cùng `service.name`, namespace và workload identity.

## 7. Regression và rủi ro cần xử lý trước production

Mặc dù không xóa panel ID của bản gốc, bản Full đã tái tạo nhiều panel dùng chung. Một số trải nghiệm của bản gốc chưa được giữ nguyên hoàn toàn:

- Nhiều mô tả chi tiết của panel gốc bị rút gọn hoặc để trống.
- Field links trong bảng RED và Trace Search có thể không còn đầy đủ như bản gốc; cần kiểm thử click route và Trace ID.
- Transform/column formatting của các bảng RED/downstream đã được đơn giản hóa; cần kiểm tra tên cột, sort và unit sau import.
- Các panel conditional JVM có thể No data trên service chưa bật Micrometer tương ứng.
- Container memory/limit cần xử lý rõ trường hợp không đặt limit.
- SLO/error-budget hiện là dashboard calculation, chưa thay thế recording rules và alert rules được quản lý bằng code.

## 8. Metric dependencies

| Thành phần | Metric/họ metric cần có |
|---|---|
| HTTP RED/SLO | `http_server_request_duration_seconds_{count,sum,bucket}` |
| Kubernetes health | `kube_deployment_*`, `kube_pod_status_ready`, `kube_pod_container_status_*`, `kube_pod_info` |
| Container resources | `container_cpu_*`, `container_memory_working_set_bytes`, `kube_pod_container_resource_limits` |
| JVM cơ bản | `jvm_memory_*`, `jvm_gc_duration_seconds_sum`, `jvm_cpu_recent_utilization_ratio`, `jvm_thread_count` |
| JVM nâng cao | `jvm_gc_duration_seconds_bucket`, `jvm_gc_memory_allocated_bytes_total`, `jvm_threads_states_threads`, `executor_*` |
| DB | `db_client_connections_*` hoặc `db_client_connection_*`, `db_client_operation_duration_seconds_*` |
| Downstream | `traces_service_graph_*`, `http_client_request_duration_seconds_*`, `rpc_client_duration_milliseconds_*` |
| Trace/log | Tempo resource/span attributes và Loki labels `cluster`, `namespace`, `service_name` |

## 9. Khuyến nghị triển khai

1. Import bản Full bằng UID riêng, không overwrite bản gốc trong giai đoạn kiểm thử.
2. Chọn một cluster, namespace và service cụ thể; không dùng ảnh chụp với `Service Name = All` để xác nhận tính đúng của dữ liệu.
3. Chạy từng họ metric trong Explore để lập ma trận service nào có HTTP, JVM, DB, Kubernetes và Micrometer metrics.
4. Kiểm thử selector pod/deployment với workload thật và bổ sung mapping nếu tên workload khác quy ước.
5. Kiểm thử link route, Trace ID, đơn vị, legend và table transformations.
6. Chuyển availability/error-budget PromQL ổn định thành recording rules; sau đó cấu hình multi-window burn-rate alerts.
7. Giữ No data cho trường hợp mất telemetry. Chỉ zero-fill khi total series của đúng service vẫn tồn tại.

## 10. Kết luận

Bản Full bổ sung đáng kể khả năng quan sát vận hành: từ APM thuần túy sang tương quan Application + Kubernetes + Container + JVM saturation + SLO. Các bổ sung hữu ích nhất cho trực vận hành là Metric age, replica/readiness, restart/OOM, CPU throttling, container memory/limit và multi-window burn rate.

Trước khi dùng làm dashboard production chính thức, cần hoàn thiện hai việc: kiểm kê metric thực tế theo từng service và khôi phục/kiểm thử các interaction, description, table formatting của dashboard gốc. Riêng DB No data phải được xử lý ở telemetry hoặc mapping label; không nên lấy dữ liệu của service khác để lấp panel.
