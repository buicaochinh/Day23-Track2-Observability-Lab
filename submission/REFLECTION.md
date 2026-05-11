# Day 23 Lab Reflection

**Student:** Bùi Cao Chinh - **Mã HV:** 2A202600001
**Submission date:** 2026-05-11
**Lab repo URL:** [Day23-Track2-Observability-Lab](https://github.com/buicaochinh/Day23-Track2-Observability-Lab)

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (27.4.0)
Compose v2:    OK  (2.31.0-desktop.2)
RAM available: 7.65 GB (OK)
Ports free:    OK
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Tôi khá bất ngờ khi biết Alertmanager không hỗ trợ nạp biến môi trường trực tiếp từ file YAML. Việc phải sử dụng lệnh `sed` trong `entrypoint` của Docker Compose để thay thế `SLACK_WEBHOOK_URL` tại thời điểm chạy (runtime) là một giải pháp tình thế (workaround) rất thực tế và thông minh để bảo mật webhook mà không cần hardcode.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```log
INFO:day23-app:Request processed trace_id=45a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1 latency=145ms
```

### Tail-sampling math

Chính sách `tail_sampling` được cấu hình để giữ lại 100% các vết có lỗi (`status_code: ERROR`) hoặc chậm (`latency > 2000ms`), và chỉ giữ lại 1% (`probabilistic: 1`) các vết bình thường.
Nếu dịch vụ tạo ra **N = 100 traces/sec**, giả sử 5% là lỗi hoặc chậm:
- Số vết lỗi/chậm giữ lại: $100 \times 0.05 = 5$ traces.
- Số vết bình thường giữ lại: $(100 - 5) \times 0.01 = 0.95$ traces.
- **Tổng cộng:** $5.95$ traces/sec được gửi đến Jaeger (tương đương ~6% tổng lưu lượng).

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": { "psi": 3.461, "drift": "yes" },
  "embedding_norm": { "psi": 0.0187, "drift": "no" },
  "response_length": { "psi": 0.0162, "drift": "no" },
  "response_quality": { "psi": 8.8486, "drift": "yes" }
}
```

### Which test fits which feature?

- `prompt_length`: **KS Test** - phù hợp để so sánh sự thay đổi phân phối của dữ liệu số liên tục (chiều dài văn bản).
- `embedding_norm`: **PSI** - dùng để kiểm tra độ ổn định của vector nhúng qua thời gian, nhận diện sự dịch chuyển nhỏ nhưng tích lũy.
- `response_length`: **KS Test** - tương tự `prompt_length`, giúp phát hiện nếu mô hình đột ngột trả về câu quá ngắn hoặc quá dài.
- `response_quality`: **PSI/Chi-square** - vì chất lượng thường được đánh giá qua các thang điểm (categorical), PSI giúp đo lường sự thay đổi tỉ lệ giữa các nhóm điểm (ví dụ: tỉ lệ điểm 1 tăng đột biến).

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Phần tích hợp dữ liệu từ Ngày 19 (Vector Store) và Ngày 20 (Llama Server) là thử thách nhất. Lý do là chúng ta không chạy trực tiếp các dịch vụ của những ngày đó mà phải dùng script giả lập (stubs), đồng thời phải cấu hình lại Prometheus `scrape_configs` để trỏ đúng vào `host.docker.internal` vì script chạy trên host máy thật chứ không phải trong mạng Docker.

---

## 6. The single change that mattered most

Thay đổi quan trọng nhất giúp hệ thống từ trạng thái "đang chạy" (works) trở nên "hữu ích" (useful) chính là việc **tinh chỉnh cấu hình Ring và Storage của Loki** kết hợp với việc gỡ bỏ hardcode `instance_addr`.

Trước đó, Loki liên tục gặp lỗi `503 Service Unavailable` do xung đột địa chỉ IP khi khởi tạo cụm Ring, khiến toàn bộ hệ thống logs bị tê liệt. Sau khi điều chỉnh lại để Loki tự động nhận diện IP và nâng log level lên `info`, hệ thống đã thu thập được logs ổn định, cho phép Grafana thực hiện truy vấn Logs-to-Metrics và tương quan hóa với Traces. Đây là khái niệm "Correlation" cốt lõi trong slide bài giảng, giúp rút ngắn thời gian MTTR (Mean Time To Resolution) khi gặp sự cố thực tế.
