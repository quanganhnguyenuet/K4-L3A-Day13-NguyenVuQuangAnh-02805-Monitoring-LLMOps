# Alerts và runbook

Các alert dưới đây dựa trên triệu chứng người dùng hoặc SLO. Kênh mặc định là Slack `#llmops-alerts`.

## Alert 1

- Tên: `high_user_latency`
- Severity: critical
- Duration: 5 phút
- Owner: `platform-oncall`
- SLI/SLO liên quan: P95 latency và SLO request thành công trong 3000 ms
- Điều kiện: `latency_p95_ms > 3000` liên tục 5 phút
- Ảnh hưởng: người dùng chờ lâu dù request có thể vẫn trả về thành công
- Ba bước kiểm tra đầu tiên:
  1. Xác định time range tăng latency trên dashboard.
  2. Lọc `response_sent` chậm trong `data/logs.jsonl` và lấy `correlation_id`.
  3. Mở trace cùng ID, so sánh duration của `retriever` và `fake-llm-generation`.
- Mitigation tạm thời: giảm concurrency, tắt incident nếu đang diễn tập, hoặc chuyển retrieval sang fallback nhanh.

## Alert 2

- Tên: `request_reliability_degraded`
- Severity: critical
- Duration: 5 phút
- Owner: `api-oncall`
- SLI/SLO liên quan: error rate tối đa 2% và retrieval success tối thiểu 90%
- Điều kiện: `error_rate_pct > 2` hoặc `retrieval_success_rate_pct < 90` liên tục 5 phút
- Ảnh hưởng: request thất bại hoặc không lấy được context cần thiết
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra breakdown `error_type` và tỷ lệ `tool_success`.
  2. Lấy một log `request_failed` cùng `correlation_id`.
  3. Mở trace tương ứng để kiểm tra status của retriever và generation.
- Mitigation tạm thời: retry có giới hạn, dùng fallback không-RAG và giảm traffic tới dependency lỗi.

## Alert 3

- Tên: `answer_quality_degraded`
- Severity: warning
- Duration: 15 phút
- Owner: `llm-oncall`
- SLI/SLO liên quan: quality proxy trung bình tối thiểu 0.75
- Điều kiện: `quality_score_avg < 0.75` liên tục 15 phút
- Ảnh hưởng: câu trả lời vẫn được trả về nhưng chất lượng thấp hoặc thiếu context
- Ba bước kiểm tra đầu tiên:
  1. So sánh quality theo feature và prompt version.
  2. Kiểm tra retrieval success, token usage và output length.
  3. Đối chiếu trace của phiên bản `candidate` với `production`.
- Mitigation tạm thời: rollback label `production` về prompt ổn định và tắt candidate gây suy giảm.
