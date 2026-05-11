# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Anh Hào
**Submission date:** 2004-06-25
**Lab repo URL:** https://github.com/nahao204/day23-observability-lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "28.3.0"
  },
  "compose_v2": {
    "ok": true,
    "version": "2.38.2-desktop.1"
  },
  "ram_gb_available": 3.75,
  "ram_ok": false,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "all_ports_free": false
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/0.png`.

### Burn-rate panel

Drop `submission/1.0.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `submission/2.4.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `submission/2.5.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | — |

### One thing surprised me about Prometheus / Grafana

Điều làm tôi ngạc nhiên nhất là khả năng quan sát (observability) không chỉ dừng lại ở việc xem log, mà còn có thể liên kết chặt chẽ giữa metrics và traces. Việc nhìn thấy một điểm bất thường trên biểu đồ Grafana và có thể nhảy ngay sang Jaeger để xem chính xác chuyện gì đã xảy ra với request đó thực sự rất mạnh mẽ.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/1.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 10, "output_tokens": 14, "quality": 0.746, "duration_seconds": 0.2459, "trace_id": "8fd2c40097059038f49830e794de8ad1", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T04:49:32.228252Z"}
```
**Trace ID:** `8fd2c40097059038f49830e794de8ad1`

### Tail-sampling math

Hệ thống sử dụng Tail-sampling để lọc dữ liệu: giữ lại 100% lỗi, 100% các request chậm (>2000ms), và 1% các request thành công bình thường.
Nếu dịch vụ tạo ra 100 traces/giây, giả sử có 5 lỗi và 5 request chậm:
- Số trace giữ lại: 5 (lỗi) + 5 (chậm) + 1% của 90 (bình thường) = 10.9 traces/giây.
- Tỷ lệ giữ lại: 10.9%.

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

- **prompt_length**: Dùng **PSI** vì đây là dữ liệu dạng phân phối số lượng, PSI giúp đánh giá sự thay đổi tổng thể của hành vi người dùng một cách trực quan.
- **embedding_norm**: Dùng **KS Test** để phát hiện những thay đổi tinh vi trong phân phối vector mà PSI có thể bỏ sót.
- **response_length**: Dùng **PSI** để theo dõi sự ổn định trong độ dài phản hồi của mô hình.
- **response_quality**: Dùng **PSI** để đánh giá sự sụt giảm chất lượng tổng thể của mô hình theo thời gian.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Chỉ số khó trích xuất nhất là các số liệu từ các ngày trước đó (như Day 19/20) vì nó đòi hỏi phải thiết lập các exporter phù hợp và đảm bảo mạng giữa các container Docker có thể giao tiếp với nhau mà không bị cấu hình sai URL của các service cũ.

---

## 6. The single change that mattered most

> **Grader reads this closest.**

Thay đổi quan trọng nhất mà tôi đã thực hiện là việc tối ưu hóa cấu hình **Tail-sampling** trong OTEL Collector. Trong một hệ thống thực tế, việc thu thập 100% dữ liệu trace là cực kỳ tốn kém và gây nhiễu cho việc phân tích. Bằng cách thiết lập chính sách thông minh — chỉ giữ lại 1% các trace thành công nhưng giữ lại 100% các trace bị lỗi hoặc có độ trễ cao — tôi đã làm cho hệ thống quan sát trở nên thực sự hữu ích cho việc debug mà vẫn tiết kiệm tài nguyên.

Điều này kết nối trực tiếp với khái niệm về "Signal-to-Noise Ratio" (Tỷ lệ Tín hiệu trên Nhiễu) trong Observability. Một stack giám sát tốt không phải là stack lưu trữ nhiều dữ liệu nhất, mà là stack cung cấp thông tin có giá trị nhất để giúp kỹ sư nhanh chóng tìm ra nguyên nhân gốc rễ (Root Cause) của vấn đề. Việc tập trung vào "ngoại lệ" thay vì "thông thường" giúp giảm MTTR (Mean Time To Repair) đánh kể trong môi trường production.
