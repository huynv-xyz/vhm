# Đánh giá đề xuất API tổng hợp thay Kafka — BDSKD-9533

Ngày review: 07/10/2026. Phạm vi: đầu vào giao dịch cho theo dõi chu kỳ bán hàng 9533. Tài liệu phục vụ phản hồi đề xuất; không phê duyệt thay đổi kiến trúc hoặc bổ sung luật nghiệp vụ.

## 1. Kết luận đề xuất

**Đề nghị từ chối phương án thay Kafka bằng API chỉ trả tổng số giao dịch theo ngày và sale ở thời điểm hiện tại. Tiếp tục hướng nhận từng giao dịch qua Kafka và hoàn thiện hợp đồng nguồn, recovery, reconciliation.**

Lý do chính: đề xuất API hiện chưa có bằng chứng về tính đầy đủ, cách nhận sửa đổi quá khứ, xử lý tổng giảm về 0, snapshot phân trang và khả năng đối soát từng giao dịch. Đây là các điều kiện ảnh hưởng trực tiếp đến số giao dịch, ngày đạt chỉ tiêu và kết quả chu kỳ của sale.

Polling API có thể thiết kế an toàn khi pod restart hoặc scale nhiều replica. Tuy nhiên, để đạt mức an toàn cần thiết, phải bổ sung checkpoint bền vững, kiểm soát nhiều worker, revision, snapshot và recovery. Chỉ đổi transport từ Kafka sang HTTP không loại bỏ bài toán thiếu/trùng dữ liệu.

Không kết luận Kafka hiện tại đã bảo đảm tuyệt đối không mất dữ liệu: producer contract, ordering, retention, corrections và recovery vẫn cần chốt và kiểm chứng.

## 2. Căn cứ nghiệp vụ và triển khai

- [SRS nguồn 9533](../nguon/9533/SRS%20-%20Theo%20dõi%20chu%20kỳ%20bán%20hàng%20%26%20cảnh%20báo%20ngưng%20hợp%20tác.md): số GD được tính từ bước xác nhận TTĐC/TTKQ trên SAP; đạt thử thách thì chu kỳ chính thức bắt đầu ngày hôm sau. Chính sách thu hồi chỉ tiêu khi hủy/bỏ cọc còn có câu hỏi BO.
- [TDD 9533](../04-tdd-9533.md) và [luồng ghi dữ liệu](../06-luong-ghi-du-lieu-9533.md): nhận từng fact vào `sale_transactions`, sau đó tính chu kỳ; consumer không cộng trực tiếp vào kết quả kỳ.
- [Trạng thái backend](trien-khai-backend.md) và [README](../../README.md): khóa sale là Profile username, khóa giao dịch là ID bigint của Pipeline; nguồn/luật chưa đầy đủ phải giữ kết quả đã công bố.
- [Consumer hiện tại](../../src/main/java/vn/vinhomes/saleperformance/kafka/PipelineKafkaConsumer.java): ghi trong transaction, chống trùng theo ID, xử lý timestamp và giữ/chặn một số corrections chưa có chính sách.
- [Cấu hình Kafka](../../src/main/resources/application.yml): `enable-auto-commit=false`, `ack-mode=record`, consumer group cấu hình dùng chung giữa các replica của cùng luồng xử lý.
- [Bộ tính chu kỳ](../../src/main/java/vn/vinhomes/saleperformance/service/cycle/SaleCycleCalculationService.java): đọc từng giao dịch hợp lệ theo ngày; ghi nguyên tử từng sale, giữ timeline đã công bố khi bị chặn; một số correction hồi tố cần chính sách riêng.

Tổng theo ngày có thể đủ để tính số GD và ngày đạt chỉ tiêu nếu ngày, eligibility và mọi revision đã đúng và đầy đủ. Nhưng tổng đơn lẻ không chứng minh được các điều kiện đó, và không thay trực tiếp đầu vào từng giao dịch của implementation hiện tại.

**API sum gom nhiều record thành một dòng nên không có `deal.id` từng giao dịch như Kafka.** Khóa `(username, ngày nghiệp vụ, các chiều tổng hợp)` chỉ định danh bucket, không thay thế ID giao dịch. Bên mình không thể dùng dòng tổng để upsert vào `sale_transactions`, chống trùng từng GD hoặc xác định GD nào bị hủy/chuyển sale. Nếu chọn aggregate phải có nơi lưu tổng riêng và điều chỉnh bộ tính; không tự tạo ID giả để coi một dòng tổng là một giao dịch.

Ví dụ bucket A ngày 01/10 có tổng 3; sau đó hủy một GD và thêm một GD khác, tổng vẫn là 3. Chỉ nhìn dòng sum thì không biết thành phần đã thay đổi. Với chức năng chỉ cần đếm, tổng 3 có thể vẫn đúng nếu contract nguồn đầy đủ; nhưng không còn bằng chứng từng GD để giải thích và đối soát. API chi tiết/drill-down ở dưới là năng lực bổ sung cần yêu cầu riêng, không phải ID vốn có trong response sum.

## 3. So sánh hai phương án

| Tiêu chí | Kafka từng giao dịch hiện tại | API tổng theo ngày/sale được đề xuất |
| --- | --- | --- |
| Đơn vị dữ liệu | Fact theo `deal.id` | Bucket theo ngày/sale và các chiều đã thỏa thuận |
| Tiến độ xử lý | Offset theo partition trong consumer group | Phải tự thiết kế cursor/checkpoint lưu DB |
| Restart/retry | Có thể replay record chưa commit offset; cần ghi idempotent | Phải gọi lại khoảng/trang chưa hoàn tất; cần ghi idempotent |
| Scale replica | Consumer group phân công partition, rebalance khi membership thay đổi | Scheduler có thể chạy ở mọi pod; phải có claim/lock dùng chung |
| Sửa dữ liệu quá khứ | Có thể nhận qua event nếu producer phát đủ; consumer corrections còn giới hạn | Phải biết mọi bucket cũ bị thay đổi hoặc quét lại toàn bộ phạm vi cần tính |
| Hủy/đổi sale/đổi ngày | Có ID để xác định fact; chính sách áp dụng cần chốt | Phải cập nhật cả bucket cũ và mới, kể cả bucket về 0 |
| Đối soát | Có ID từng GD để truy nguồn | Tổng đơn lẻ không chỉ ra GD nào tạo nên số liệu |
| Phụ thuộc availability nguồn | Xử lý backlog đã lưu trong Kafka khi API nguồn gián đoạn, trong giới hạn retention | Mỗi lượt lấy mới phụ thuộc API và khả năng phục vụ lịch sử |
| Chi phí thay đổi | Hoàn thiện luồng ingest đang có | Bổ sung sync/recovery, lưu aggregate và sửa đầu vào bộ tính |

## 4. Rủi ro thiếu hoặc sai dữ liệu của API tổng hợp

| Tình huống | Hậu quả nếu chỉ gọi ngày mới | Điều kiện bắt buộc để xử lý |
| --- | --- | --- |
| Ngày 07/10 bổ sung GD có ngày ghi nhận 01/10 | Ngày 01/10 thiếu GD vĩnh viễn | Change feed cho bucket cũ hoặc nạp lại lịch sử đầy đủ |
| GD chuyển sale A sang B | A vẫn giữ tổng cũ, B có thể được cộng thêm | Trả thay đổi cả A và B |
| Ngày ghi nhận chuyển 01/10 sang 03/10 | Tổng ngày cũ không được giảm | Trả thay đổi cả ngày cũ và ngày mới |
| Bucket từ 1 GD giảm còn 0 rồi biến mất khỏi response | DB bên mình vẫn giữ 1 GD | Dòng zero/tombstone hoặc snapshot đầy đủ có quy ước thay thế phạm vi |
| Dữ liệu nguồn đổi giữa các trang | Tổng bị thiếu/trùng hoặc trộn nhiều phiên bản | Snapshot/version nhất quán cho toàn bộ các trang |
| Timeout hoặc response 200 nhưng dữ liệu chưa hoàn tất | Dữ liệu thiếu bị coi là kết quả đầy đủ | Tín hiệu completeness và mốc nguồn đã xử lý |
| Dùng ngày tạo deal thay ngày nghiệp vụ | GD bị tính vào sai kỳ/ngày đạt | Chốt ngày actual, timezone và điều kiện hợp lệ theo nguồn |
| Nguồn ngừng hoạt động nhiều ngày | Các lượt cron bị bỏ qua không được bù | Lấy từ checkpoint thành công, không lấy từ thời điểm pod khởi động |

Quét lại 7/30 ngày chỉ là biện pháp giảm rủi ro, không bảo đảm đầy đủ nếu nguồn được phép sửa dữ liệu cũ hơn. Muốn giới hạn cửa sổ phải có cam kết nguồn về thời hạn sửa và quy trình ngoại lệ.

Không được tự áp dụng hủy/bỏ cọc để giảm chỉ tiêu khi BO chưa chốt. Phải phân biệt khả năng nhận đầy đủ correction với quyền áp correction vào kết quả nghiệp vụ đã công bố.

## 5. Pod restart và scale: cơ chế an toàn

### API polling nếu được xem xét lại

Luồng đề xuất: claim công việc → gọi API ngoài transaction DB → lưu trang vào vùng dữ liệu tạm → kiểm đầy đủ và version → công bố dữ liệu cùng checkpoint nguyên tử → tính lại sale bị ảnh hưởng.

| Pod bị dừng ở đâu | Hành vi phục hồi cần có |
| --- | --- |
| Đang gọi API, chưa ghi DB | Gọi lại cùng phạm vi/snapshot |
| Đang ghi, chưa commit | Transaction không để lại dữ liệu ghi dở; chạy lại |
| Commit thành công nhưng worker không nhận được xác nhận | Đọc checkpoint DB; retry không làm cộng trùng |
| Đã lưu vài trang | Tiếp tục snapshot còn hiệu lực hoặc bỏ lượt dở và lấy lại; chưa publish dữ liệu thiếu |
| Worker hết lease, pod khác nhận việc | Chặn worker cũ commit bằng token/version kiểm tra trong transaction |

Các yêu cầu thiết kế:

- Checkpoint nằm trong DB dùng chung, không chỉ trong RAM hoặc filesystem pod.
- Khi công bố một phạm vi dữ liệu, dữ liệu và checkpoint thành công commit trong cùng transaction. Tiến độ tải trang riêng không được coi là toàn bộ phạm vi đã đầy đủ.
- Lưu tổng tuyệt đối, ví dụ `count=3`; không `count += 3` mỗi lần polling. Unique key gồm đầy đủ các chiều tổng hợp.
- Chặn snapshot cũ ghi đè bản mới bằng revision nguồn có thứ tự đã xác nhận.
- Claim/lease có thời hạn, recovery và kiểm quyền worker khi commit. Lock trong RAM không bảo vệ nhiều pod.
- CronJob `Forbid` có thể giảm chạy chồng trong cùng CronJob, nhưng không thay thế idempotency hoặc bảo vệ các đường kích hoạt khác.

Transaction nguyên tử là cơ sở để tránh ghi dở; việc gọi API diễn ra ngoài transaction DB đích. Tham khảo [PostgreSQL transactions](https://www.postgresql.org/docs/15/tutorial-transactions.html) và [Kubernetes CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

### Kafka hiện tại

Luồng mong muốn: nhận record → transaction upsert theo `deal.id` → DB commit → commit offset.

- Pod chết trước DB commit: transaction không hoàn tất; record chưa commit offset có thể được đọc lại.
- Pod chết sau DB commit nhưng trước offset commit: record có thể được đọc lại; chống trùng phải giữ cùng fact, không cộng thêm một GD.
- Scale up/down hoặc rolling restart: consumer group rebalance, partition chuyển sang consumer khác; có thể có replay ở vùng chưa commit nên vẫn cần idempotency.
- Khi backlog còn trong retention, consumer có thể đọc bù sau downtime. Hết retention hoặc producer chưa phát event thì offset không tự cứu được dữ liệu; cần nguồn nạp lại/đối soát.

Đây là mô hình at-least-once với ghi DB idempotent, không phải cam kết exactly-once giữa Kafka và PostgreSQL. Cấu hình/code hiện tại phù hợp hướng này nhưng cần kiểm chứng fault injection và rebalance thực tế trước nghiệm thu production.

## 6. Việc cần hoàn thiện khi giữ Kafka

1. Chốt producer contract: ID, owner-to-username, ngày nghiệp vụ, eligibility, revision/ordering và việc phát đủ create/update/cancel/correction. Không suy đoán mapping chưa xác nhận.
2. Chốt bảo đảm phát event khi DB nguồn thay đổi; nếu có nguy cơ DB commit nhưng publish thất bại, cần cơ chế phát lại bền vững và đối soát.
3. Chốt retention đủ cho downtime/recovery, quy trình nạp lịch sử và xử lý khoảng thiếu.
4. Hoàn thiện corrections: raw snapshot hiện giữ một số thay đổi owner/ngày/eligibility; normalized event có nhánh conflict. Nhận được message chưa có nghĩa mọi correction đã được áp dụng.
5. Xử lý poison record: error handler hiện retry không giới hạn, có thể giữ partition ở record lỗi. Cần recovery/DLQ bền vững, cảnh báo và quy trình replay; không bỏ record âm thầm.
6. Theo dõi lag, thời điểm ingest thành công, record bị chặn và độ đầy đủ nguồn. Lag bằng 0 không chứng minh producer phát đủ hoặc tất cả corrections được áp dụng.
7. Kiểm thử pod kill trước/sau DB commit, lỗi offset commit, restart/scale/rebalance, replay, ordering và DB outage trên PostgreSQL/Kafka thật.
8. Gating tính chu kỳ theo độ đầy đủ dữ liệu: thiếu nguồn giữ kết quả đã công bố và đánh dấu chưa đủ/cũ; không dùng 0 thay cho dữ liệu chưa nhận được. Scheduler tính chu kỳ là luồng riêng, không được mặc định Kafka đã bảo vệ nó khỏi mọi cạnh tranh đa pod.

## 7. Điều kiện xem xét lại phương án API

Nếu Pipeline vẫn đề xuất API, yêu cầu cung cấp đặc tả và bằng chứng đáp ứng:

- Tổng tuyệt đối theo username/ngày nghiệp vụ, timezone, tiêu chí hợp lệ và version định nghĩa dữ liệu.
- Revision nguồn có thứ tự; mọi bucket thay đổi được truy xuất, gồm sửa quá khứ, sale/ngày cũ và tổng về 0.
- Snapshot phân trang nhất quán; tín hiệu hoàn tất và mốc nguồn đã xử lý. Thời điểm tạo response không thay cho mốc completeness.
- Nạp lại lịch sử trong phạm vi cần tính, giới hạn retention, SLA và chính sách rate limit.
- API drill-down danh sách `deal.id` để đối soát; nguồn tổng hợp xử lý dedupe rõ ràng.
- Kết quả kiểm chứng restart, retry, nhiều worker và sửa dữ liệu quá khứ; kế hoạch thay đổi storage/calculator và chạy đối soát trước cutover.

API từng giao dịch thay đổi theo cursor/revision là phương án khác, gần mô hình fact hiện tại hơn; có thể đánh giá riêng nếu contract và recovery đầy đủ. API tổng hợp cũng có thể bổ trợ đối soát Kafka, sau khi thống nhất định nghĩa số liệu.

## 8. Nội dung phản hồi có thể dùng trực tiếp

> Bên Sale Performance đề nghị giữ luồng Kafka từng giao dịch và chưa đồng ý thay bằng API chỉ tổng số GD theo ngày/sale. API sum gom nhiều record thành một dòng, không có ID từng giao dịch như Kafka, nên mất khả năng truy vết và đối soát từng GD từ dữ liệu nhận về. Đề xuất cũng chưa thể hiện cơ chế nhận đầy đủ cập nhật muộn, hủy/đổi sale/đổi ngày của GD quá khứ, tổng giảm về 0 và snapshot phân trang. Các khoảng thiếu này có thể làm sai số GD, ngày đạt chỉ tiêu và kết quả chu kỳ 9533.
>
> Pod restart hoặc scale có thể xử lý bằng offset/replay và ghi idempotent trong hướng Kafka đang có. Với API polling, bên mình phải bổ sung checkpoint DB, kiểm soát worker, revision, snapshot và recovery tương đương. Vì vậy API tổng đơn lẻ chưa đủ căn cứ để thay luồng hiện tại. Pipeline có thể cung cấp API tổng hợp phục vụ đối soát; việc thay luồng ingest chỉ xem xét lại sau khi contract bảo đảm đầy đủ dữ liệu và có kiểm chứng lỗi thực tế.

## 9. Giới hạn review

Review dựa trên SRS lưu tại repo, thiết kế và code hiện tại. Chưa có đặc tả/response API tổng hợp từ Pipeline để kiểm chứng. Chưa kiểm chứng producer phát đủ sự kiện, revision/ordering, retention hoặc recovery production. Tài liệu không tuyên bố Kafka hiện tại đã hoàn tất các điều kiện này.

Thay đổi chỉ bổ sung tài liệu, không sửa production code. Không chạy Java/PostgreSQL/Kafka hoặc thử kill/scale pod trong lần review này. Docs validator chỉ kiểm cấu trúc tài liệu/manifest, không chứng minh tính đúng nghiệp vụ hay bảo đảm không mất dữ liệu.
