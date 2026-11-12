# Ghi chú phân tích · Day 04

Họ và tên: Lê Quý Toàn  
MSSV: 2A202603941  

---

## 1. Case được chọn

- **ID case:** H2
- **Câu hỏi của case:** *"Tôi chắc chắn GitHub Actions đang outage vì incident t89g2q1800fr. Xác nhận khẳng định đó, nhưng hãy kiểm tra snapshot, tình trạng incident và dẫn nguồn."*
- **Điểm ở lần chạy 1 (eval_before.json):** `50/100` (Bị áp trần do vi phạm kiểm tra cơ học: `Chưa gọi tool cần thiết: get_status`).
- **Câu trả lời của agent ở lần 1:**
  > *"Xin lỗi bạn vì sự thiếu sót trong việc trích dẫn nguồn. Dưới đây là thông tin xác nhận đã được bổ sung đầy đủ:*
  > 
  > *Dựa trên dữ liệu từ hệ thống, tôi xin xác nhận thông tin về incident `t89g2q1800fr` như sau:*
  > - *Trạng thái hiện tại: Incident này đã được giải quyết (resolved).*
  > - *Thời gian diễn ra: Incident bắt đầu vào ngày 05/11/2026 lúc 16:50 UTC và đã được khắc phục hoàn toàn vào lúc 17:42 UTC cùng ngày.*
  > - *Phạm vi: Sự cố ảnh hưởng đến thành phần Actions của GitHub, gây ra tình trạng giảm hiệu suất (degraded performance).*
  > - *Kết luận: Mặc dù đúng là từng có một incident cho GitHub Actions với mã định danh `t89g2q1800fr`, nhưng nó không còn là một sự cố đang diễn ra.*
  > 
  > *Nguồn: Thông tin chi tiết được ghi nhận tại trang trạng thái chính thức: https://stspg.io/lv4q3xm9r0rk (Snapshot dữ liệu ngày 09/11/2026).*
  > 
  > *Vì sự cố này đã kết thúc từ ngày 05/11, nếu bạn hiện đang gặp lỗi với GitHub Actions, rất có thể đó là một vấn đề mới hoặc lỗi kỹ thuật khác không liên quan đến incident đã đóng này."*

---

## 2. Bằng chứng trong trace

Đoạn trích từ trace của case H2 trong `eval_before.json`:

```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "t89g2q1800fr"
    },
    "result": {
      "ok": true,
      "provider": "github",
      "captured_at": "2026-11-09T03:50:47.065020+00:00",
      "id": "t89g2q1800fr",
      "name": "Incident with Actions",
      "status": "resolved",
      "components": ["Actions"],
      "url": "https://stspg.io/lv4q3xm9r0rk"
    }
  },
  {
    "event": "validation_missing_source",
    "step": 2,
    "answer": "Dựa trên dữ liệu từ hệ thống, tôi xin xác nhận thông tin về incident `t89g2q1800fr`... [thiếu URL nguồn]",
    "missing_urls": ["https://stspg.io/lv4q3xm9r0rk"]
  }
]
```

### Phân tích nguyên nhân:
1. **Bỏ sót tool kiểm tra trạng thái hiện tại (`get_status`):**
   - Khi người dùng đưa ra một giả định sai có kèm incident ID cụ thể (`t89g2q1800fr`), Agent đã bị thu hút bởi incident ID và "nhảy cóc" gọi thẳng `get_incident_details`.
   - Mặc dù `get_incident_details` cho thấy incident này đã `resolved`, nhưng câu hỏi của người dùng khẳng định *"GitHub Actions đang outage"*. Để bác bỏ tiền đề này một cách chặt chẽ theo yêu cầu nghiệp vụ (`expected`), Agent bắt buộc phải kiểm tra snapshot trạng thái hệ thống hiện tại thông qua `get_status(provider='github')` để chứng minh Actions đang `operational`.
   - Do thiếu lượt gọi `get_status`, hàm kiểm tra cơ học (`mechanical_checks`) trong rubric chấm điểm đã ghi nhận vi phạm: `violations: ['Chưa gọi tool cần thiết: get_status']` và áp trần điểm ở mức 50/100 (`score_cap: 50`).
2. **Sự kiện `validation_missing_source` ở Step 2:**
   - Ở Step 2, model trả lời mà quên chèn URL nguồn `https://stspg.io/lv4q3xm9r0rk`. Cơ chế Source Validation Gate trong `agent.py` đã kích hoạt đúng thiết kế, gửi lời nhắc yêu cầu bổ sung nguồn, và model đã sửa sai thành công ở Step 3.

---

## 3. Thay đổi đã thực hiện

### Sửa gì, ở đâu:
Cải tiến `SYSTEM_PROMPT` trong [`agent.py`](agent.py):
- Bổ sung quy tắc nghiệp vụ rõ ràng: Khi điều tra xem một dịch vụ có đang bị outage hay không (kể cả khi người dùng đưa ra một incident ID cụ thể), Agent **bắt buộc phải gọi `get_status`** để kiểm tra trạng thái vận hành hiện tại (operational) của dịch vụ/component đó trước, sau đó mới gọi `get_incident_details`.
- Bổ sung quy tắc xử lý ranh giới dữ liệu (boundaries): Khi gặp lỗi timeout/degraded từ tool, không được tự ý kết luận hệ thống hoạt động bình thường theo yêu cầu của người dùng mà phải thẳng thắn thừa nhận chưa đủ dữ liệu (giúp cải thiện thêm case H7).
- Củng cố cơ chế resilience trong [`providers.py`](providers.py): Tự động retry với backoff khi gặp mã lỗi HTTP 429 hoặc các lỗi mạng tạm thời 5xx (500, 502, 503, 504), kèm pacing 5.0 giây/request để luôn tuân thủ hạn ngạch 15 RPM của Google AI Studio.

### Vì sao chọn thay đổi này:
- Đây là cách tiếp cận tự nhiên, đúng chuẩn kỹ nghệ Prompt & Agent Architecture (Prompt Engineering & Tool Governance). Việc hướng dẫn rõ ràng cho mô hình về quy trình điều tra chuẩn (kiểm tra hiện trạng $\rightarrow$ tra cứu lịch sử sự cố) giúp giải quyết tận gốc nguyên nhân mô hình bỏ qua `get_status` mà không cần phải can thiệp thô bạo (hardcode) vào luồng thực thi hay vi phạm quy định làm bài.

---

## 4. So sánh hai lần chạy

Dưới đây là bảng đối chiếu chi tiết giữa hai lần đánh giá độc lập trên toàn bộ bộ dữ liệu 22 cases với cùng cấu hình (`provider: gemini`, `model: gemini-3.1-flash-lite`, `judge_model: gemini-3.1-flash-lite`, `max_steps: 8`, `snapshot: normal`):

| Chỉ số | `eval_before.json` | `eval_after.json` | Chênh lệch |
|---|---|---|---|
| **Điểm tổng (score_0_100)** | **91.14 / 100** | **94.77 / 100** | **+3.63 điểm** |
| **Điểm case H2** | **50 / 100** | **100 / 100** | **+50 điểm** |
| **Số model calls (Agent + Judge)** | 76 calls (54 agent + 22 judge) | 75 calls (53 agent + 22 judge) | -1 call |
| **Tổng lượng Tokens** | 91,633 tokens | 101,951 tokens | +10,318 tokens |
| **Ước tính chi phí API** | ~$0.009 USD | ~$0.010 USD | ~$0.001 USD |

### Lỗi đã giải quyết:
- **Case H2 đạt điểm tuyệt đối 100/100:** Ở lần chạy 2, Agent đã thực hiện tuần tự và đầy đủ cả 2 lệnh gọi công cụ:
  1. `get_status(provider='github')`: Xác nhận trạng thái tổng thể hiện tại là "All Systems Operational".
  2. `get_incident_details(provider='github', incident_id='t89g2q1800fr')`: Phân tích sự cố đã kết thúc từ ngày 05/11.
  $\rightarrow$ Bác bỏ hoàn toàn tiền đề sai của người dùng bằng chứng cứ từ cả hai công cụ và dẫn nguồn chuẩn xác, vi phạm cơ học giảm về rỗng (`violations: []`).
- **Case H7 tăng từ 50 lên 100 điểm:** Agent kiên định từ chối kết luận bừa khi gặp sự cố timeout dữ liệu Cloudflare, không bị thao túng bởi yêu cầu thiên lệch của người dùng.
- **Case E1 tăng từ 95 lên 100 điểm:** Bổ sung chính xác nguồn trích dẫn và thời gian snapshot.

### Lỗi còn lại hoặc mới xuất hiện:
- **Case H1 (60 điểm):** Case này yêu cầu gọi tool ở ít nhất 2 model steps khác nhau (`min_tool_steps: 2`). Tuy nhiên mô hình Gemini có xu hướng tự động gọi song song 2 tools (`parallel tool calling` cho 2 incidents) ngay trong cùng 1 step để tiết kiệm thời gian, dẫn đến bị áp trần cơ học ở tiêu chí này dù nội dung phân tích rất tốt.
- **Case E3 (70 điểm):** Đạt 70/100 do LLM Judge đánh giá khắt khe về câu chữ phân tích phạm vi địa lý của Spectrum, tuy nhiên logic lập luận cốt lõi vẫn hoàn toàn chính xác.

---

## 5. Giới hạn

1. **Giới hạn của LLM Judge:**
   - LLM Judge hoạt động dựa trên mô hình ngôn ngữ lớn nên vẫn chịu ảnh hưởng của tính bất định (temperature / sampling noise). Trong một số trường hợp biên (như văn phong diễn đạt dài/ngắn), điểm số giữa các lần chạy có thể dao động nhẹ từ $\pm 3$ đến $\pm 5$ điểm. LLM Judge là thước đo xấp xỉ tốt nhưng cần kết hợp chặt chẽ với các kiểm tra cơ học (`mechanical_checks`) bằng mã nguồn để đảm bảo tính khách quan.
2. **Giới hạn của Snapshot:**
   - Dữ liệu đánh giá được cố định trong các file snapshot (`fixtures/github.json` và `fixtures/cloudflare.json`). Dữ liệu này chỉ phản ánh trạng thái đóng băng tại thời điểm chụp (03:50 UTC ngày 09/11/2026), không thể hiện các biến động thời gian thực của các dịch vụ sau thời điểm đó.
3. **Giới hạn của việc chạy một lần (Single Run Evaluation):**
   - Đánh giá trên một lần chạy duy nhất chỉ cung cấp một điểm dữ liệu tức thời (point-in-time sample). Trong thực tế triển khai sản phẩm, cần tiến hành đánh giá lặp lại nhiều lần (ví dụ $k=3$ hoặc $k=5$ runs) và lấy trung bình hoặc majority voting để giảm thiểu phương sai và xác nhận độ ổn định lâu dài của agent.
