# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: high_latency_p95
- Severity: critical
- Duration: 5m
- Kênh thông báo: Slack (#llmops-alerts)
- SLI/SLO liên quan: `primary_slo.fast_successful_requests` (SLO 99.5% requests <= 3000ms trong 28 ngày)
- Điều kiện và thời gian duy trì: `p95_latency_ms > 3000` liên tục trong 5 phút.
- Ảnh hưởng tới người dùng: Người dùng bị trễ phản hồi lâu (>3s), trải nghiệm chatbot bị đơ hoặc timeout trên UI client.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard kiểm tra panel `latency` và TTFT (Time to First Token) để xác định điểm nghẽn là ở bước retrieval hay LLM generation.
  2. Truy vấn logs `data/logs.jsonl` lọc các dòng có `latency_ms > 3000`, đối chiếu `feature` và `model` để xem lỗi cục bộ hay diện rộng.
  3. Mở Langfuse Cloud, tìm trace theo `correlation_id` của request bị chậm để phân tích waterfall chi tiết span `retrieval` vs span `generation`.
- Mitigation tạm thời: Chuyển hướng traffic sang model nhanh hơn (fallback model), hoặc hạ timeout của vector database, bật caching response cho các câu hỏi phổ biến.
- Owner: oncall-llmops

## Alert 2

- Tên: high_error_rate
- Severity: critical
- Duration: 3m
- Kênh thông báo: Slack (#llmops-alerts)
- SLI/SLO liên quan: `guardrails.error_rate_pct_max` (tối đa 2.0%) và `primary_slo.fast_successful_requests`
- Điều kiện và thời gian duy trì: `error_rate_pct > 2.0` duy trì trong 3 phút liên tiếp.
- Ảnh hưởng tới người dùng: Người dùng nhận thông báo lỗi 500/503 hoặc không nhận được câu trả lời từ hệ thống.
- Ba bước kiểm tra đầu tiên:
  1. Mở panel `errors` trên dashboard, xem phân bố các loại lỗi qua `count_by(error_type)`.
  2. Truy vấn log file tìm event `request_failed`, trích xuất `error_type`, `detail` và correlation ID tương ứng.
  3. Mở Langfuse trace tương ứng với correlation ID để kiểm tra observation báo lỗi và xem stack trace chi tiết.
- Mitigation tạm thời: Bật fallback response tĩnh cho tính năng bị lỗi; nếu do dependency bên thứ ba (LLM API hoặc vector store) thì kích hoạt circuit breaker để tránh cascade failure.
- Owner: oncall-llmops

## Alert 3

- Tên: retrieval_degradation
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#llmops-alerts)
- SLI/SLO liên quan: `guardrails.retrieval_success_rate_pct_min` (tối thiểu 90.0%) và `guardrails.quality_score_avg_min` (>= 0.75)
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90.0` duy trì trong 5 phút.
- Ảnh hưởng tới người dùng: Chatbot không lấy được tài liệu ngữ cảnh chính xác, câu trả lời bị generic (fallback) hoặc chất lượng thông tin giảm sút.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel `errors` (sub-metric `tool_success_rate_pct`) và panel `quality` xem chất lượng câu trả lời có sụt giảm dưới 0.75 không.
  2. Kiểm tra log line tìm các event có `tool_name == "retrieval"` và `tool_success == false`.
  3. Mở Langfuse quan sát span `retrieval`: kiểm tra `doc_count`, query message tóm tắt và lỗi kết nối corpus / vector database.
- Mitigation tạm thời: Tạm thời fallback sang tài liệu cached cục bộ hoặc nới lỏng ngưỡng similarity tìm kiếm; restart vector service nếu có hiện tượng connection pool cạn kiệt.
- Owner: rag-platform-team
