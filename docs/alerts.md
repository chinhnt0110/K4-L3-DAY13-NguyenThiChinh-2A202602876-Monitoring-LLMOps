# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- **Tên:** HighLatencyP95Breached
- **Severity:** P1 (Critical)
- **Duration:** 3m
- **Kênh thông báo:** Slack (#alerts-prod)
- **SLI/SLO liên quan:** SLO Latency P95 < 2000ms trong 30 ngày (Target: 95%).
- **Điều kiện và thời gian duy trì:** P95 Latency của toàn bộ request > 2000ms liên tục trong 3 phút.
- **Ảnh hưởng tới người dùng:** Người dùng gặp hiện tượng phản hồi bị đơ lag, chatbot trả lời chậm, trải nghiệm tương tác trực tuyến bị suy giảm nghiêm trọng.
- **Ba bước kiểm tra đầu tiên:**
  1. Kiểm tra Dashboard Panel 1 (Latency) và Panel 3 (Retrieval Success) để xem độ trễ tăng ở bước nào (LLM Generation hay RAG Retrieval).
  2. Tra cứu `data/logs.jsonl` hoặc ELK bằng query `latency_ms > 2000` để lấy các `correlation_id` của request bị ảnh hưởng.
  3. Mở Langfuse Tracing theo `correlation_id` để kiểm tra span waterfall (xem Vector DB retrieval hay API call của LLM đang bị nghẽn).
- **Mitigation tạm thời:**
  - Nếu nghẽn ở Vector DB/Retrieval: Kích hoạt Semantic Cache hoặc tạm thời chuyển sang chế độ degraded mode (trả về fallback context).
  - Nếu nghẽn ở LLM Provider: Chuyển hướng traffic sang model backup (ví dụ: failover sang provider khác).
- **Owner:** On-call SRE / LLMOps Team

## Alert 2

- **Tên:** HighApiErrorRate
- **Severity:** P1 (Critical)
- **Duration:** 5m
- **Kênh thông báo:** Slack (#alerts-prod)
- **SLI/SLO liên quan:** SLO Availability: Tỷ lệ request thành công (non-5xx) ≥ 99.0% trong 30 ngày (Error budget: 1%).
- **Điều kiện và thời gian duy trì:** Tỷ lệ lỗi HTTP 5xx hoặc unhandled exception > 5% trong cửa sổ trượt 5 phút.
- **Ảnh hưởng tới người dùng:** Người dùng bị lỗi hệ thống, không nhận được câu trả lời từ chatbot (HTTP 500/503).
- **Ba bước kiểm tra đầu tiên:**
  1. Kiểm tra Dashboard Panel 3 (Error Rate) và Panel 2 (Traffic) để xác nhận tỷ lệ lỗi và số lượng request lỗi.
  2. Lọc log với `level: "error"` để tìm root exception (ví dụ: connection timeout, parsing error, schema validation failure).
  3. Kiểm tra xem có đợt promote prompt version mới hoặc deploy code mới nào vừa diễn ra trước đó không.
- **Mitigation tạm thời:**
  - Nếu lỗi phát sinh do prompt version mới: Rollback ngay nhãn `production` trên Langfuse về phiên bản baseline ổn định trước đó.
  - Nếu lỗi do sập dịch vụ phụ thuộc: Khởi động lại connection pool và thông báo cho đội phụ trách service liên quan.
- **Owner:** Backend Lead / On-call Engineer

## Alert 3

- **Tên:** DailyCostSpikeDetected
- **Severity:** P2 (Warning)
- **Duration:** 15m
- **Kênh thông báo:** Slack (#alerts-cost)
- **SLI/SLO liên quan:** Ngân sách vận hành (Cost Budget) hàng ngày không vượt quá 150% mức trung bình (Threshold Total ≤ $2.5/ngày).
- **Điều kiện và thời gian duy trì:** Tổng chi phí token lũy kế trong ngày tăng đột biến > 150% hạn mức dự kiến hoặc tốc độ tiêu tốn chi phí > $0.50/giờ.
- **Ảnh hưởng tới người dùng:** Không ảnh hưởng trực tiếp đến người dùng, nhưng đe dọa cạn kiệt ngân sách dự án và nguy cơ bị khóa API account.
- **Ba bước kiểm tra đầu tiên:**
  1. Mở Dashboard Panel 4 (Cost) và Panel 5 (Tokens) để xác định token in hay token out tăng vọt.
  2. Phân tích log theo `user_id_hash` và `session_id` để phát hiện các tài khoản hoặc luồng có dấu hiệu spam, loop, hoặc prompt injection.
  3. Kiểm tra prompt template mới xem có cấu hình `max_tokens` quá lớn hoặc context window bị chèn dữ liệu rác không.
- **Mitigation tạm thời:**
  - Áp dụng Rate Limiting chặt chẽ hơn đối với các `user_id_hash` có hành vi tiêu tốn token bất thường.
  - Tạm thời hạ giới hạn `max_tokens` của response output xuống mức an toàn.
- **Owner:** LLMOps / Tech Lead
