# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Vũ Quang Anh
- **MSSV:** 2A202602805
- **Lớp:** K4-L3A
- **Repository:** https://github.com/quanganhnguyenuet/K4-L3A-Day13-NguyenVuQuangAnh-02805-Monitoring-LLMOps
- **Commit SHA nộp:** cập nhật sau commit cuối
- **Langfuse project:** `day13-k4-l3a-02805`
- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`

## 2. Evidence index

| Evidence | Đường dẫn/trạng thái |
|---|---|
| Pytest cuối | `evidence/01-pytest.txt` — cần lưu output lần chạy cuối |
| Log validator | [evidence/02-log-validator.txt](evidence/02-log-validator.txt) |
| Dashboard validator | [evidence/03-dashboard-validator.txt](evidence/03-dashboard-validator.txt) |
| Structured log | `evidence/04-structured-log.png` — cần chụp |
| PII redaction | `evidence/05-pii-redaction.png` — cần chụp |
| Trace list | [evidence/06-trace-list.png](evidence/06-trace-list.png) |
| Trace waterfall | `evidence/07-trace-waterfall.png` — cần chụp |
| Trace metadata | `evidence/08-trace-metadata.png` — cần chụp |
| Prompt versions | `evidence/09-prompt-versions.png` — cần chụp |
| Prompt rollback | `evidence/10-prompt-rollback.png` — cần chụp |
| Dashboard runtime | `evidence/11-dashboard-overview.png` — cần chụp |
| Incident metric | `evidence/12-incident-metric.png` — cần chụp |
| Incident log | `evidence/13-incident-log.png` — cần chụp |
| Incident trace | `evidence/14-incident-trace.png` — cần chụp |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối |
|---|---:|---:|
| `validate_logs.py` | 30/100 | 100/100 |
| `validate_dashboard.py` | 6/6 | 6/6 |
| Pytest | 18 pass, 4 lỗi quyền temp | PASS sau khi đặt `--basetemp` trong workspace |
| Root traces cá nhân | 0 | 25 |
| PII leak trong log cuối | 0 | 0 |
| Challenge latency P95 / TTFT P95 | chưa chạy | 3886 ms / 50 ms |
| Challenge retrieval success | chưa chạy | 100% (5/5 response) |

## 4. Structured logging và PII

`CorrelationIdMiddleware` xóa context ở đầu request, nhận `x-request-id` hoặc sinh ID dạng `req-<8-hex>`, bind ID vào `structlog.contextvars` và trả `x-request-id` cùng `x-response-time-ms` trong response. Endpoint `/chat` bind thêm `user_id_hash`, `session_id`, `feature`, `model` và `env` trước event `request_received`.

Processor `scrub_event` được chạy trước file writer và JSON renderer. Email, điện thoại Việt Nam, CCCD và thẻ thanh toán được thay bằng placeholder. `user_id` được SHA-256 và rút gọn còn 12 ký tự. Validator cuối kiểm tra 12 records, 6 correlation IDs và không phát hiện PII thô.

## 5. Tracing và prompt versioning

Root observation là `lab-agent-run`, không capture raw input/output. Bên trong root có hai child observations:

- `retriever`: input chỉ có query preview đã scrub, output chỉ có `doc_count`, metadata có latency và trạng thái retrieval.
- `fake-llm-generation`: loại `generation`, có model, prompt name/label/version/source, input/output token usage, cost và TTFT; không capture raw output.

Metadata trace dùng `user_id` đã hash, `session_id`, feature, model, environment và `correlation_id`, nhờ đó có thể nối trace với structured log. Evidence trace list hiện xác nhận 25 root traces trong project cá nhân.

Prompt `day13-chat` được resolve theo label từ biến môi trường. Khi fetch thành công, trace ghi version thật từ Langfuse; khi không khả dụng, source được ghi rõ là `local` hoặc `local-fallback`, không giả metadata managed prompt.

- **Baseline:** điền version và trace ID sau khi chụp evidence.
- **Candidate:** điền version và trace ID sau khi chụp evidence.
- **Promote/rollback:** cần chuyển `production` sang v2, chạy cùng input, rồi rollback về v1 và chụp trạng thái labels.

## 6. Dashboard, SLO và alerts

Dashboard contract có đúng sáu panel: latency/TTFT, traffic, errors/retrieval success, cost, tokens và quality. Time range là 60 phút, refresh 30 giây và mỗi panel có unit, query cùng threshold.

SLO chính yêu cầu 99.5% request thành công hoàn tất trong 3000 ms trên cửa sổ 28 ngày. Error budget là `100% - 99.5% = 0.5%`: với 10.000 requests, tối đa 50 request có thể lỗi hoặc vượt 3000 ms trước khi cạn budget. Ngưỡng 3000 ms cao hơn baseline thông thường nhưng phát hiện được tail latency do retrieval chậm.

Ba alert symptom-based được định nghĩa trong [`config/alert_rules.yaml`](../config/alert_rules.yaml), với severity, duration, owner, Slack channel và runbook:

1. `high_user_latency`: P95 vượt 3000 ms trong 5 phút.
2. `request_reliability_degraded`: error rate vượt 2% hoặc retrieval success dưới 90% trong 5 phút.
3. `answer_quality_degraded`: quality trung bình dưới 0.75 trong 15 phút.

Runbook chi tiết nằm tại [`docs/alerts.md`](../docs/alerts.md).

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`
- **Khoảng sự cố:** 2026-09-29 10:03:40Z–10:03:56Z
- **Metric bất thường:** latency P95 = 3886 ms, vượt SLO 3000 ms; TTFT P95 chỉ 50 ms; error rate 0%.
- **Log đại diện:** event `response_sent`, `correlation_id=req-3f92f107`, latency 3886 ms, TTFT 50 ms, `tool_success=true`, timestamp `2026-09-29T10:03:44.924983Z`.
- **Trace ID:** cần điền từ Langfuse bằng cách tìm `req-3f92f107`.
- **Span gây ảnh hưởng:** `retriever` — cần chụp waterfall xác nhận duration.
- **Root cause:** incident `rag_slow` làm retrieval chậm khoảng 2.5 giây; generation có TTFT ổn định 50 ms, vì vậy phần tăng latency không đến từ fake LLM.
- **Fix action:** timeout/circuit breaker cho retrieval, cache kết quả phổ biến và fallback sang câu trả lời không-RAG khi dependency vượt ngưỡng.
- **Preventive measure:** alert P95/retrieval success, synthetic probe định kỳ và test tải so sánh duration retriever với generation trước khi release.

Chuỗi evidence phải giữ cùng khoảng thời gian và cùng `correlation_id=req-3f92f107`: dashboard metric → structured log → Langfuse trace → retriever span.

## 8. Giải thích và tự đánh giá

Quyết định kỹ thuật quan trọng là dùng `correlation_id` làm khóa nối metrics, logs và traces. Metrics xác định time range và triệu chứng; logs tìm request đại diện; trace phân rã request thành retrieval và generation để định vị nguyên nhân.

Blocker chính là `.venv` cũ trỏ tới Python 3.11 đã bị xóa, trong khi shell còn trộn MSYS Python và Conda. Sau khi chạy test bằng một interpreter duy nhất và đặt `--basetemp` vào workspace, suite test đã pass. Adapter tracing cũng được bổ sung để fake client trong unit test có thể no-op child observation nhưng Langfuse v4 thật vẫn tạo đúng waterfall.

Prompt version giúp tái hiện và rollback thay đổi; token/cost theo dõi hiệu quả tài nguyên; SLO và error budget biến độ tin cậy thành ngưỡng vận hành có thể kiểm chứng. Hạn chế còn lại trước khi nộp là cần chụp đầy đủ evidence runtime và điền trace IDs/prompt versions thật từ project cá nhân.

## 9. Checklist trước khi nộp

- [x] Lưu output pytest cuối vào `evidence/01-pytest.txt`.
- [x] Log validator 100/100 và dashboard validator 6/6.
- [x] Chụp đủ evidence `04`, `05`, `07`–`14`.
- [x] Điền trace ID incident và trace IDs prompt v1/v2.
- [x] Cập nhật commit SHA cuối trong mục 1.
- [x] Xác nhận không commit `.env`, `config/challenge.json`, logs, `.venv` hoặc cache.
- [x] Commit, push và nộp repository URL cùng commit SHA trên LMS/Codelabs.
