# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Võ Đức Trí
- **MSSV:** 2A202602603
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/VoDucTri/K4-L3A-Day13-VoDucTri-2A202602603-Monitoring-LLMOps
- **Commit SHA cuối:**
- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602603`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 | 100/100 | Đạt điểm tối đa: đầy đủ correlation_id, context enrichment và che PII thành công |
| `validate_dashboard.py` | 6/6 panel hợp lệ | 6/6 panel hợp lệ | Dashboard contract đạt chuẩn cấu hình 6 panels |
| `pytest` | 22 passed | 24 passed | Thêm tests cho CCCD và Credit Card vào tests/test_pii.py |
| Số traces hợp lệ | 2+ | 28 traces | Đã kết nối Langfuse Cloud project day13-k4-l3a-2A202602603 với đầy đủ span tree |
| Số PII leak | 0 | 0 | Không còn PII nguyên văn trong logs sau khi scrub |
| Latency P95 / TTFT P95 | 152.0ms / 50.0ms | 152.0ms / 50.0ms | Đo được từ /metrics sau khi chạy load test |
| Retrieval success rate | 100% | 100% | Toàn bộ requests hoàn thành thành công |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:**
  - Trong `CorrelationIdMiddleware`, trước mỗi request thực hiện `clear_contextvars()` để xóa context cũ, ngăn rò rỉ giữa các request.
  - Kiểm tra header `x-request-id`: nếu client truyền lên thì sử dụng lại, nếu không có hoặc rỗng thì tự động sinh mới theo định dạng `req-<8-hex>` (`f"req-{uuid.uuid4().hex[:8]}"`).
  - Gắn correlation ID vào structlog thông qua `bind_contextvars(correlation_id=correlation_id)` và lưu vào `request.state.correlation_id`.
  - Trả về correlation ID và thời gian xử lý trong response headers: `x-request-id` và `x-response-time-ms`.
- **Các metadata được ghi vào structured log:**
  - Thông tin chuẩn: `ts` (ISO timestamp UTC), `level`, `service`, `event`, `correlation_id`.
  - Request enrichment: `user_id_hash` (băm sha256 12 ký tự), `session_id`, `feature`, `model`, `env` (được bind ngay trước khi log `request_received`).
  - Operational metrics trong `response_sent`: `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`, cùng `payload` tóm tắt preview.
- **Cách bảo đảm PII được scrub trước khi ghi:**
  - Định nghĩa biểu thức chính quy nhận diện PII trong `app/pii.py` cho 4 loại: `email`, `phone_vn`, `cccd` (12 số), `credit_card` (16 số có hoặc không có dấu gạch/khoảng trắng).
  - Viết hàm `_scrub_value` duyệt đệ quy qua dictionary, list, string để thay thế triệt để mọi vị trí chứa PII thành token chuẩn: `[REDACTED_EMAIL]`, `[REDACTED_PHONE_VN]`, `[REDACTED_CCCD]`, `[REDACTED_CREDIT_CARD]`.
  - Đăng ký processor `scrub_event` vào pipeline structlog ngay trước `JsonlFileProcessor` và `JSONRenderer`, đảm bảo toàn bộ dữ liệu trước khi xuất ra console hay ghi vào `data/logs.jsonl` đều đã được làm sạch.
- **Cách kiểm chứng kết quả:**
  - Chạy `python scripts/validate_logs.py`: đạt điểm tuyệt đối **100/100** (0 missing required, 0 missing enrichment, 10 unique correlation IDs, 0 PII leaks).
  - Bổ sung và chạy unit test `pytest -q tests/test_pii.py`: kiểm tra đủ các định dạng email, phone VN (+84, 09x, dấu chấm, dấu gạch), CCCD 12 số, thẻ tín dụng 16 số dạng có và không có dấu cách/gạch nối. Toàn bộ 24/24 tests pass.
  - Kiểm tra trực tiếp file `data/logs.jsonl` và response headers qua curl: xác nhận có `x-request-id` và `x-response-time-ms`.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
  - Traces được gửi trực tiếp đến project Langfuse cá nhân `day13-k4-l3a-2A202602603` (ID: `cmumc88kj12cead0cuajxv6ie`) thuộc tổ chức `VoDucTri's Organization Hobby` (gắn liền tài khoản `voductri744464@...`).
  - Mọi trace đều mang environment `dev`, tags `["lab", feature, "claude-sonnet-4-5"]`, và `user_id_hash` (băm sha256 12 ký tự của user test, ví dụ: `b87190f89839`).
  - Evidence danh sách traces được chụp tại [06-trace-list.png](evidence/06-trace-list.png).
- **Cấu trúc root/retrieval/generation observations:**
  - Root observation: `@observe(name="lab-agent-run", as_type="agent")` bao quát toàn bộ vòng đời thực thi của agent request (~1,199ms).
  - Child 1: `@observe(name="retrieval", as_type="retriever")` bọc bước tìm kiếm tài liệu từ corpus (~2.1ms, `doc_count=1`).
  - Child 2: `@observe(name="generation", as_type="generation")` bọc bước gọi `FakeLLM.generate()`, ghi nhận model `claude-sonnet-4-5`, usage (input 28 tokens, output 89 tokens), cost ($0.001419 USD) và liên kết prompt `day13-chat`.
  - Phân tích waterfall: Bước retrieval diễn ra cực nhanh (~2ms), thời gian phản hồi chủ yếu nằm ở bước generation (TTFT ~50ms + sinh token), được thể hiện rõ ràng trên biểu đồ waterfall tại [07-trace-waterfall.png](evidence/07-trace-waterfall.png).
- **Cách nối trace với log:**
  - Mỗi request log ghi nhận một `correlation_id` duy nhất (ví dụ: `req-prod-v1-004`).
  - Khi agent thực thi, `correlation_id` được gắn vào Langfuse trace metadata thông qua `propagate_attributes(metadata={"correlation_id": correlation_id, ...})`.
  - Tìm kiếm `correlation_id` trên thanh Search của Langfuse Traces sẽ định vị chính xác trace và ngược lại. Metadata được thể hiện chi tiết tại [08-trace-metadata.png](evidence/08-trace-metadata.png).
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (`Feature={{feature}}\nDocs={{docs}}\nQuestion={{message}}`), mang nhãn `baseline` và `production`.
- **Version/label candidate:** Version 2 (thêm chỉ dẫn trả lời ngắn gọn theo bullet points), mang nhãn `candidate`. Xem danh sách version tại [09-prompt-versions.png](evidence/09-prompt-versions.png).
- **Trace ID của mỗi version:**
  - Baseline (v1, label `baseline`): `4f619d26ee432f128362605e76c64d9a`
  - Candidate (v2, label `candidate`): `0f1cd82b79079ed2bd53569a5402967e`
  - Production (v2 sau promote): `4d4820c4a758fea1d1c12c4de10dcf41`
  - Production (v1 sau rollback): `07587221e5e7c782a51eb3a554aeee9e`
- **Cách promote và rollback `production`:**
  - Promote: Cập nhật nhãn `production` từ v1 sang v2 thông qua Langfuse API (`client.update_prompt(name="day13-chat", version=2, new_labels=["candidate", "production"])`). Các request tiếp theo với `LANGFUSE_PROMPT_LABEL=production` tự động tải v2 mà không cần restart server.
  - Rollback: Khi cần quay lại bản an toàn, cập nhật nhãn `production` trở về v1 (`client.update_prompt(name="day13-chat", version=1, new_labels=["baseline", "production"])`). Hệ thống tự động xóa cache và nạp lại v1 ngay lập tức. Bằng chứng được lưu tại [10-prompt-rollback.png](evidence/10-prompt-rollback.png).

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
  - File cấu hình `config/dashboard.yaml` đáp ứng đầy đủ contract, được xác thực bởi `python scripts/validate_dashboard.py` (6/6 panel hợp lệ):
    1. `latency`: P50 (120ms), P95 (152ms), P99 (182ms), TTFT P95 (50ms). Ngưỡng P95 <= 3000ms.
    2. `traffic`: Thông lượng request mỗi phút (10 req/min, tổng 31 requests). Ngưỡng >= 1 req/min.
    3. `errors`: Tỷ lệ lỗi `error_rate_pct` (0.0%, ngưỡng <= 2%) và tỷ lệ thành công của retrieval `tool_success_rate_pct` (100.0%).
    4. `cost`: Chi phí theo thời gian ($0.043 USD lũy kế, ngưỡng <= $2.50 USD/ngày).
    5. `tokens`: Số token in/out (in: 840, out: 3,720, tổng: 4,560 tokens, ngưỡng <= 50,000 tokens).
    6. `quality`: Điểm chất lượng trung bình dựa trên heuristic (0.81 / 1.0, ngưỡng >= 0.75).
  - Tổng quan dashboard thực tế được ghi nhận tại [11-dashboard-overview.png](evidence/11-dashboard-overview.png).
- **SLO và lý do chọn:**
  - Primary SLO: `fast_successful_requests` với mục tiêu 99.5% requests thành công và có độ trễ <= 3000ms trong khoảng thời gian 28 ngày.
  - Lý do: Phản hồi dưới 3 giây là tiêu chuẩn để người dùng không cảm thấy gián đoạn khi trò chuyện với AI. Ngưỡng 99.5% cho phép sai số nhỏ do biến động mạng Internet mà vẫn giữ cam kết dịch vụ cao.
- **Cách tính error budget:**
  - Error budget = $100\% - 99.5\% = 0.5\%$.
  - Trong chu kỳ 28 ngày với giả định 100,000 requests, ngân sách lỗi cho phép là $100,000 \times 0.5\% = 500$ requests thất bại hoặc có latency vượt quá 3,000ms.
  - Khi error budget cạn kiệt, đội ngũ kỹ thuật phải ngừng deploy tính năng mới để tập trung vá lỗi hệ thống và tối ưu hóa hiệu năng.
- **Ba alert và runbook tương ứng:**
  - Alert 1: `high_latency_p95` (Critical, điều kiện `p95_latency_ms > 3000` trong 5m). Runbook [docs/alerts.md#alert-1](../docs/alerts.md#alert-1): Kiểm tra dashboard phân biệt bottleneck giữa retrieval và generation; mitigation: bật cache hoặc chuyển model dự phòng.
  - Alert 2: `high_error_rate` (Critical, điều kiện `error_rate_pct > 2.0` trong 3m). Runbook [docs/alerts.md#alert-2](../docs/alerts.md#alert-2): Kiểm tra error distribution, xem log event `request_failed` và stack trace trên Langfuse; mitigation: kích hoạt circuit breaker hoặc fallback response.
  - Alert 3: `retrieval_degradation` (Warning, điều kiện `retrieval_success_rate_pct < 90.0` trong 5m). Runbook [docs/alerts.md#alert-3](../docs/alerts.md#alert-3): Kiểm tra tỷ lệ thành công của retrieval tool và quality score; mitigation: restart vector service hoặc dùng local cached corpus.

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** 16:35:46 – 16:39:30 ngày 29/09/2026 (UTC+7)
- **Triệu chứng từ metrics:**
  - Khi thực hiện kiểm thử tải với concurrency 5 (`python scripts/load_test.py --challenge --concurrency 5`), panel `latency` ghi nhận tail latency P95 tăng vọt nghiêm trọng từ mức bình thường 152.0ms lên tới **14,579.0ms** (hơn 14.5 giây).
  - Độ trễ này vượt xa ngưỡng cảnh báo của bài toán (`latency_threshold_ms: 2000`) và vi phạm nghiêm trọng SLO dịch vụ (`latency <= 3000ms`). Bằng chứng metric tăng vọt được lưu tại [12-incident-metric.png](evidence/12-incident-metric.png).
- **Log line và correlation ID liên quan:**
  - Correlation ID bất thường: `req-17ba3c06` (thuộc tính năng `feature: monitoring`).
  - Dòng log trích xuất từ `data/logs.jsonl`:
    ```json
    {"ts":"2026-09-29T09:36:10.519000Z","level":"info","service":"api","event":"response_sent","correlation_id":"req-17ba3c06","env":"dev","latency_ms":3762,"ttft_ms":50,"tokens_in":34,"tokens_out":112,"cost_usd":0.001782,"quality_score":0.8,"tool_name":"retrieval","tool_success":true,"payload":{"answer_preview":"Starter answer. You should improve this..."}}
    ```
  - Dòng log xác nhận request kéo dài tới **3,762ms** (vượt ngưỡng 3,000ms), trong đó bước nghiệp vụ có sử dụng `tool_name: retrieval`. Bằng chứng log được lưu tại [13-incident-log.png](evidence/13-incident-log.png).
- **Trace ID và span gây ảnh hưởng:**
  - Trace ID tương ứng trên Langfuse Cloud: `193ea7aa926d6c94ebd6c78e797e9112`.
  - Phân tích chi tiết thời gian các span trong trace:
    - Root span `lab-agent-run`: **3,764ms** (3.76s).
    - Child span `retrieval`: **2,504ms** (chiếm 66.5% tổng thời gian request, là điểm nghẽn nghiêm trọng).
    - Child span `generation`: chỉ mất **152ms** (TTFT 50ms, thời gian sinh token của LLM hoàn toàn bình thường).
  - Kết luận: Điểm nghẽn gây trễ nằm hoàn toàn ở bước `retrieval`, không phải do LLM generation. Bằng chứng waterfall tại [14-incident-trace.png](evidence/14-incident-trace.png).
- **Root cause:**
  - Kịch bản sự cố `rag_slow` được kích hoạt thông qua file cấu hình challenge, dẫn tới hàm `retrieve()` trong `app/mock_rag.py` bị delay nhân tạo 2.5s (`time.sleep(2.5)`) cho mỗi request.
  - Khi hệ thống chịu tải đồng thời (concurrency 5), việc các request đều bị chặn ở bước retrieval đã gây ra hiện tượng nghẽn hàng đợi (head-of-line blocking) và cạn kiệt thread pool xử lý, đẩy tổng thời gian phản hồi ở các request sau lên tới 14,579ms.
- **Fix action:**
  - Ngay lập tức gọi API tắt sự cố bằng `python scripts/inject_incident.py --disable` (endpoint `POST /incidents/rag_slow/disable`).
  - Kiểm tra lại request sau khi phục hồi: thời gian phản hồi quay về mức bình thường **270.3ms**, hệ thống hồi phục 100%.
  - Trong môi trường production thực tế: Tái khởi động/scale out cụm vector search database, đặt timeout nghiêm ngặt cho client gọi retrieval (ví dụ: 800ms) kèm cơ chế fallback trả về câu trả lời tổng quát hoặc dữ liệu local cache khi vector store quá tải.
- **Preventive measure:**
  - Đặt timeout và circuit breaker ở tầng gọi retrieval: nếu bước retrieval quá 1,000ms thì ngắt kết nối ngay để tránh cascade latency làm tắc nghẽn API.
  - Cấu hình alert rule `high_latency_p95` (P1) kích hoạt khi P95 > 3000ms trong 5 phút để đội ngũ On-call phát hiện sớm trước khi cạn kiệt error budget.
  - Thiết lập bộ nhớ đệm (Semantic Caching với Redis) cho các câu hỏi phổ biến nhằm giảm 70-80% tải trực tiếp lên vector database.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
  - Đặt PII Scrubber đệ quy (`scrub_event`) ngay trước file writer và JSON renderer trong pipeline Structlog. Quyết định này bảo đảm toàn bộ dữ liệu log dù ở bất kỳ tầng nào cũng không bao giờ bị rò rỉ thông tin cá nhân (email, SĐT, CCCD, thẻ ngân hàng) ra đĩa hay màn hình console mà không cần sửa code ở từng hàm nghiệp vụ.
- **Một lỗi/blocker đã gặp:**
  - Lỗi OTLP batch export timeout khi gửi spans lên Langfuse Cloud (`cloud.langfuse.com` đặt tại Frankfurt) do độ trễ mạng quốc tế đôi lúc vượt quá 5s mặc định, dẫn đến spans không kịp flush trước khi process kết thúc.
- **Cách tìm nguyên nhân và xử lý:**
  - Xem stack trace từ OpenTelemetry exporter, phát hiện timeout 5.0s. Xử lý bằng cách tăng cấu hình `OTEL_EXPORTER_OTLP_TIMEOUT=30` và `LANGFUSE_TIMEOUT=30` trong `.env`, đồng thời thêm lời gọi `client.flush()` tường minh trong lifespan shutdown và sau các batch request.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - **Metrics** đóng vai trò là "chuông báo động" đầu tiên (cho biết *khi nào* có sự cố và *triệu chứng* gì, ví dụ: P95 latency tăng vọt lên 14s).
  - **Logs** cung cấp "bằng chứng ngữ cảnh" (thông qua `data/logs.jsonl`, xác định chính xác *request nào* bị ảnh hưởng với `correlation_id`, thời điểm, feature và user).
  - **Traces** là "kính hiển vi" (mở Langfuse theo `correlation_id` để soi từng span con trong waterfall, khoanh vùng chính xác *hàm nào / span nào* gây nghẽn — ở đây là `retrieval` mất 2.50s thay vì LLM).
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
  - Prompt là logic cốt lõi của ứng dụng GenAI. Quản lý prompt version bằng managed platform (Langfuse) giúp tách biệt prompt khỏi codebase, cho phép cập nhật hoặc rollback nhãn `production` về bản cũ (v1) tức thì với zero-downtime khi bản candidate (v2) gặp sự cố. Việc giám sát token/cost và đặt SLO ngăn chặn chi phí tăng vọt và bảo đảm cam kết dịch vụ với khách hàng.
- **Điều quan trọng nhất đã học:**
  - Hiểu sâu sắc và thực hành trọn vẹn kiến trúc quan sát 3 trụ cột (Metrics - Logs - Traces) trong một ứng dụng LLMOps chuẩn production; làm chủ quy trình băm PII, liên kết correlation ID xuyên suốt và quản trị vòng đời prompt an toàn.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**
  - Đã hoàn thành 100% tất cả các checkpoint CP0, CP1, CP2, CP3; toàn bộ tests, validators đều đạt điểm tối đa và thu thập đầy đủ 14 bằng chứng thực tế.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
