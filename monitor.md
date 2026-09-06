# Các nhóm cần bổ sung vào dashboard APM Detail Java


## 1. Service Health — Kubernetes

<table>
<colgroup>
<col style="width:15%"><col style="width:14%"><col style="width:19%"><col style="width:31%"><col style="width:21%">
</colgroup>
<thead><tr><th>Thuộc tính/panel</th><th>Metric/công thức</th><th>Ý nghĩa</th><th>Giá trị giám sát</th><th>Điều kiện/giới hạn</th></tr></thead>
<tbody>
<tr><td>Ready replicas (%)</td><td><code>100 × available / clamp_min(desired, 1)</code></td><td>Tỷ lệ replica available so với số replica mong muốn.</td><td>Cho biết service có đủ replica hay đang thiếu capacity và rollout có bị kẹt không; giúp phát hiện suy giảm trước khi người dùng thấy lỗi và quyết định kiểm tra readiness, scheduling hoặc scale.</td><td>Cần kube-state-metrics. No data nghĩa là không tìm thấy Deployment hoặc thiếu telemetry, không được hiểu là service khỏe. Nếu chủ động scale về 0, giá trị 0% cần được diễn giải riêng.</td></tr>
<tr><td>Metric age</td><td><code>time() - max(timestamp(http_count))</code></td><td>Số giây kể từ sample HTTP metric gần nhất.</td><td>Cho biết telemetry còn cập nhật hay dashboard đang dùng dữ liệu cũ; tránh kết luận nhầm “không có lỗi” khi Prometheus, exporter hoặc OTel agent đã ngừng gửi metric.</td><td>Ngưỡng phụ thuộc scrape interval: khoảng 0–60s thường bình thường; &gt;120s cần kiểm tra; &gt;300s hoặc No data có thể là mất target/agent.</td></tr>
<tr><td>Restarts — cả khoảng</td><td><code>sum(increase(restarts_total[$__range]))</code></td><td>Tổng số lần container restart trong khoảng thời gian đang chọn.</td><td>Cho biết service có vừa crash/restart dù hiện tại đã hồi phục; giúp phát hiện sự cố ngắn đã biến mất khỏi RED metrics và tìm log ngay trước khi process chết.</td><td>Chỉ zero-fill khi <code>kube_pod_info</code> xác nhận pod đúng selector tồn tại. Không tìm thấy pod phải giữ No data.</td></tr>
<tr><td>OOMKilled — cả khoảng</td><td><code>max(max_over_time(last_reason{reason="OOMKilled"}[$__range]))</code></td><td>Xác định container từng bị dừng do vượt memory limit.</td><td>Cho biết restart có liên quan tới memory/OOM; giúp khoanh vùng sang memory limit, heap, direct buffer, metaspace, native memory hoặc thread stack.</td><td>Kube-state-metrics phản ánh last terminated reason Kubernetes còn lưu, không phải event store lâu dài.</td></tr>
<tr><td>Pod readiness</td><td><code>max by (pod) (pod_ready{condition="true"})</code></td><td>Trạng thái Ready của từng pod.</td><td>Cho biết pod nào đủ điều kiện nhận traffic và thời điểm mất/lấy lại readiness; phát hiện pod Running nhưng không phục vụ request.</td><td>1 = Ready, 0 = chưa Ready. Cần kube-state-metrics.</td></tr>
<tr><td>Request rate theo pod/instance</td><td><code>sum by (instance) (rate(http_count[$rateinterval]))</code></td><td>Lưu lượng request của từng instance.</td><td>Cho biết traffic có phân phối đều hay dồn vào một pod; giúp phát hiện hot pod, cold pod, stale endpoint, session affinity hoặc load balancing không đều.</td><td><code>instance</code> có thể là pod IP/scrape endpoint, không chắc là tên pod. Muốn map chính xác cần label pod/resource attribute nhất quán.</td></tr>
<tr><td>Pod restarts</td><td><code>sum by (pod, container) (increase(restarts_total[$rateinterval]))</code></td><td>Chi tiết restart theo từng pod/container.</td><td>Cho biết pod/container nào restart, thời điểm và mức lặp lại; giúp phân biệt một pod lỗi với rollout hoặc sự cố ảnh hưởng đồng loạt.</td><td>Mỗi điểm là số restart trong cửa sổ trượt <code>$rateinterval</code>, không phải event tức thời.</td></tr>
<tr><td>Termination reason</td><td><code>max by (pod, reason) (max_over_time(last_reason[$rateinterval]))</code></td><td>Nguyên nhân như OOMKilled, Error, ContainerCannotRun.</td><td>Giúp chuyển đúng hướng xử lý sang memory, application log/exit code, image, command, permission, volume hoặc runtime.</td><td>CrashLoopBackOff là waiting reason; muốn bao phủ đủ cần thêm waiting-reason metric hoặc xem pod events.</td></tr>
<tr><td>Phạm vi pod</td><td><code>^${service_name:regex}(-deployment)?-.*$</code></td><td>Giới hạn metric pod vào workload của service.</td><td>Tránh lấy nhầm service có tên chứa chuỗi trùng như <code>miniportal-vhm-agent-api-*</code>; bảo đảm restart, OOM và tài nguyên đúng scope.</td><td>Mapping theo quy ước tên. Nếu workload không theo quy ước, cần chuẩn hóa label chung thay vì nới regex thành substring.</td></tr>
<tr><td>Phạm vi Deployment</td><td><code>^${service_name:regex}(-deployment)?$</code></td><td>Giới hạn metric cấp Deployment.</td><td>Bảo đảm Ready replicas match đúng workload; không sử dụng nhầm regex pod có phần hash phía sau.</td><td>Cùng giới hạn mapping theo tên như selector pod.</td></tr>
</tbody>
</table>

## 2. Container Resources

<table>
<colgroup>
<col style="width:15%"><col style="width:14%"><col style="width:19%"><col style="width:31%"><col style="width:21%">
</colgroup>
<thead><tr><th>Thuộc tính/panel</th><th>Metric/công thức</th><th>Ý nghĩa</th><th>Giá trị giám sát</th><th>Điều kiện/giới hạn</th></tr></thead>
<tbody>
<tr><td>Container CPU usage theo pod</td><td><code>sum by (pod) (rate(cpu_usage_seconds[$rateinterval]))</code></td><td>Số CPU core từng pod sử dụng.</td><td>Cho biết CPU từng pod và tương quan với traffic; giúp phát hiện hot pod, workload bất thường, lập kế hoạch capacity, scale ngang và right-sizing CPU.</td><td>Đơn vị core. Riêng giá trị 0.8 core chưa cho biết đã gần CPU limit hay chưa.</td></tr>
<tr><td>Container CPU / limit (%)<br><strong>Đề xuất bổ sung</strong></td><td><code>100 × cpu_usage / cpu_limit</code></td><td>Phần trăm CPU sử dụng so với CPU limit.</td><td>Đặt CPU usage vào đúng bối cảnh giới hạn; giúp phát hiện pod sát trần CPU trước khi throttling nặng và hỗ trợ right-sizing.</td><td>Panel này chưa có trong <code>apm-detail-java-full.json</code>. Cần thêm vào dashboard và cần resource-limit metric cho CPU. Pod không đặt limit sẽ No data.</td></tr>
<tr><td>CPU throttling theo pod (%)</td><td><code>100 × rate(throttled) / clamp_min(rate(periods), ε)</code></td><td>Tỷ lệ kỳ CFS bị throttle do chạm CPU limit.</td><td>Cho biết container có bị CPU limit chặn và throttling có trùng lúc latency tăng; phân biệt code chậm với thiếu CPU quota.</td><td>Đây là tỷ lệ kỳ CFS bị throttle, không phải phần trăm CPU time bị mất.</td></tr>
<tr><td>Memory working set theo pod</td><td><code>sum by (pod) (working_set_bytes)</code></td><td>Memory pod giữ theo góc nhìn cgroup/kubelet.</td><td>Cho biết memory thực tế có tăng và có vượt xa Java heap; giúp phát hiện direct buffer, metaspace, native allocation, thread stack, agent hoặc sidecar.</td><td>Query hiện tính các container có image và có thể bao gồm sidecar. Muốn chỉ lấy application container cần selector container cụ thể.</td></tr>
<tr><td>Container memory / limit (%)</td><td><code>100 × working_set / memory_limit</code></td><td>Phần trăm memory sử dụng so với limit.</td><td>Cho biết pod còn bao nhiêu headroom và pod nào gần OOMKill; hỗ trợ quyết định tăng limit, giảm heap, sửa leak hoặc scale workload.</td><td>Cần memory resource-limit metric. Container không đặt limit sẽ No data. Ngưỡng production phải điều chỉnh theo workload.</td></tr>
</tbody>
</table>

## 3. JVM Saturation

<table>
<colgroup>
<col style="width:15%"><col style="width:14%"><col style="width:19%"><col style="width:31%"><col style="width:21%">
</colgroup>
<thead><tr><th>Thuộc tính/panel</th><th>Metric/công thức</th><th>Ý nghĩa</th><th>Giá trị giám sát</th><th>Điều kiện/giới hạn</th></tr></thead>
<tbody>
<tr><td>Heap utilization (%) theo pod</td><td><code>100 × heap_used / heap_limit</code></td><td>Phần trăm heap sử dụng so với giới hạn.</td><td>Cho biết pod JVM nào dùng heap cao và headroom còn lại; phát hiện một replica gần cạn heap để điều chỉnh heap hoặc scale.</td><td>Exporter có thể biểu diễn pool/limit khác nhau theo GC/JDK; cần đối chiếu với <code>-Xmx</code>.</td></tr>
<tr><td>Old Gen theo pod</td><td><code>sum by (instance, pool) (heap_used{pool=~"old|tenured"})</code></td><td>Dung lượng object sống lâu trong JVM.</td><td>Cho biết Old Gen có giảm sau major GC hay tăng liên tục; giúp phát hiện object retention, cache không kiểm soát hoặc memory leak.</td><td>ZGC/Shenandoah có thể không có pool old/tenured nên panel No data dù heap metric tồn tại.</td></tr>
<tr><td>GC pause p99 theo pod</td><td><code>histogram_quantile(0.99, rate(gc_bucket[$rateinterval]))</code></td><td>Thời gian pause p99 của GC theo instance.</td><td>Cho biết HTTP p99 có tăng cùng GC pause; giúp xác định GC là nguyên nhân tail latency và định hướng tuning.</td><td>Cần histogram <code>_bucket</code>. Có <code>_sum</code> không đồng nghĩa có <code>_bucket</code>.</td></tr>
<tr><td>Allocation rate</td><td><code>sum by (instance) (rate(allocated_bytes_total[$rateinterval]))</code></td><td>Số byte JVM cấp phát mỗi giây.</td><td>Cho biết allocation spike dù TPS không tăng; giúp phát hiện object churn, serialization nặng và xác định nhu cầu profiling/JFR.</td><td>Thường là Micrometer metric; No data có thể nghĩa là metric chưa bật.</td></tr>
<tr><td>Thread states</td><td><code>sum by (instance, state) (threads_by_state)</code></td><td>Số thread RUNNABLE, BLOCKED, WAITING...</td><td>Cho biết thread đang chạy, block hay chờ; giúp phát hiện lock contention, nguy cơ deadlock và chọn thời điểm lấy thread dump.</td><td>Metric chỉ cho trạng thái tổng quát, không thay thế stack trace/JFR.</td></tr>
<tr><td>Executor active / pool / queue</td><td><code>active</code>, <code>pool_size</code>, <code>queued</code></td><td>Mức dùng thread pool và số task xếp hàng.</td><td>Cho biết executor còn capacity hay queue đang tăng; phát hiện saturation trước timeout và hỗ trợ điều chỉnh pool/queue hoặc call blocking.</td><td>Executor phải được bind/observe bằng Micrometer; custom executor không bind sẽ không xuất hiện.</td></tr>
</tbody>
</table>

## 4. SLO & Error Budget

<table>
<colgroup>
<col style="width:15%"><col style="width:14%"><col style="width:19%"><col style="width:31%"><col style="width:21%">
</colgroup>
<thead><tr><th>Thuộc tính/panel</th><th>Metric/công thức</th><th>Ý nghĩa</th><th>Giá trị giám sát</th><th>Điều kiện/giới hạn</th></tr></thead>
<tbody>
<tr><td>Availability SLO</td><td><code>$availability_slo</code><br>mặc định <code>0.999</code></td><td>Mục tiêu availability dùng tính allowed error ratio.</td><td>Cho biết service được đánh giá theo mục tiêu 99%–99.95%; tạo chuẩn chung để DevOps, development và product đánh giá reliability.</td><td>Biến dashboard phù hợp mô phỏng. SLO production chính thức nên quản lý bằng configuration/recording rule.</td></tr>
<tr><td>Latency SLO</td><td><code>$slo_bucket</code><br>mặc định <code>0.5s</code></td><td>Ngưỡng latency tối đa được coi là đạt.</td><td>Cho biết request phải hoàn thành dưới ngưỡng nào; giúp đánh giá latency theo cam kết thay vì chỉ nhìn p95/p99.</td><td>Giá trị phải trùng một boundary thật của histogram, nếu không panel sẽ No data.</td></tr>
<tr><td>Availability SLI</td><td><code>100 × (1 - rate(5xx) / rate(total))</code></td><td>Tỷ lệ request không gặp lỗi server theo cửa sổ rate.</td><td>Cho biết availability thực tế và service có đạt SLO; chuyển error rate kỹ thuật thành chỉ số dùng cho báo cáo reliability.</td><td>Có total nhưng không có 5xx thì lỗi = 0; mất total phải giữ No data. Công thức hiện coi 4xx là hợp lệ.</td></tr>
<tr><td>Latency SLI</td><td><code>100 × rate(target_bucket) / rate(+Inf_bucket)</code></td><td>Tỷ lệ request hoàn thành dưới ngưỡng latency.</td><td>Cho biết tỷ lệ request đạt mục tiêu và latency SLO có bị vi phạm; hỗ trợ xác định service/endpoint cần tối ưu.</td><td>Bucket phải được aggregate đúng theo <code>le</code> và <code>$slo_bucket</code> phải tồn tại.</td></tr>
<tr><td>Error budget remaining — rolling 30d</td><td><code>clamp(100 × (1 - error_30d / allowed), 0, 100)</code></td><td>Phần trăm ngân sách lỗi còn lại trong cửa sổ trượt 30 ngày.</td><td>Cho biết budget đã tiêu và mức lỗi còn được phép; là căn cứ để tiếp tục release, dừng release hoặc ưu tiên reliability work.</td><td>Cần Prometheus retention ≥30 ngày và label/job ổn định. Service mới chạy chưa đại diện đủ 30 ngày.</td></tr>
<tr><td>Availability burn rate</td><td><code>error_ratio_window / (1 - SLO)</code></td><td>Tốc độ tiêu error budget so với mức cho phép.</td><td>0 = không có 5xx; 1 = tiêu đúng tốc độ; &gt;1 = tiêu nhanh hơn kế hoạch. So cửa sổ 5m, 30m, 1h, 6h để phân biệt spike với incident kéo dài.</td><td>Hiện mới là visualization; muốn dùng thật cần recording rules và multi-window burn-rate alerts.</td></tr>
</tbody>
</table>
