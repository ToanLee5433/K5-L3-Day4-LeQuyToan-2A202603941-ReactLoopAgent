# Ghi chú phân tích · Day 04

Họ và tên: Lê Quý Toàn  
MSSV: 2A202603941  

---

## 1. Case được chọn

- **ID case:** H2
- **Câu hỏi của case:** *"Tôi chắc chắn GitHub Actions đang outage vì incident t89g2q1800fr. Xác nhận khẳng định đó, nhưng hãy kiểm tra snapshot, tình trạng incident và dẫn nguồn."*
- **Điểm ở lần chạy 1 (eval_before.json):** `50/100` (Bị áp trần điểm do vi phạm kiểm tra cơ học: `Chưa gọi tool cần thiết: get_status`).
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
      "components": [
        "Actions"
      ],
      "url": "https://stspg.io/lv4q3xm9r0rk"
    }
  },
  {
    "event": "validation_missing_source",
    "step": 2,
    "answer": "Dựa trên dữ liệu từ hệ thống, tôi xin xác nhận thông tin về incident `t89g2q1800fr`... [thiếu URL nguồn]",
    "missing_urls": [
      "https://stspg.io/lv4q3xm9r0rk"
    ]
  }
]
```

### Phân tích nguyên nhân:
1. **Thiếu lượt gọi tool kiểm tra hiện trạng (`get_status`):**
   - Khi đọc câu hỏi, mô hình thấy người dùng cung cấp sẵn mã sự cố cụ thể (`t89g2q1800fr`), nên nó đã lập tức gọi hàm `get_incident_details` để tra cứu luôn.
   - Mặc dù kết quả trả về chỉ ra sự cố đã `resolved`, nhưng câu hỏi của người dùng đưa ra một giả định sai là *"GitHub Actions đang outage"*. Để phản bác giả định này một cách thuyết phục và đúng theo yêu cầu đánh giá của đề bài (`expected`), agent không chỉ cần biết sự cố cũ đã đóng, mà còn phải đối chiếu với trạng thái tổng thể của hệ thống tại thời điểm snapshot bằng `get_status(provider='github')` để khẳng định chắc chắn Actions đang `operational`.
   - Do agent không gọi `get_status`, hàm chấm điểm cơ học (`mechanical_checks`) đã gắn cờ vi phạm: `violations: ['Chưa gọi tool cần thiết: get_status']` và áp trần điểm số tối đa cho case này là 50/100 (`score_cap: 50`).
2. **Kích hoạt Evidence Gate ở Step 2:**
   - Ở lần trả lời đầu tiên (Step 2), mô hình quên đính kèm URL nguồn `https://stspg.io/lv4q3xm9r0rk`. Logic kiểm tra nguồn trong `agent.py` đã bắt được lỗi này (`validation_missing_source`), đưa phản hồi yêu cầu dẫn nguồn và mô hình đã bổ sung đầy đủ link ở Step 3.

---

## 3. Thay đổi đã thực hiện

### Sửa gì, ở đâu:
1. **Cải tiến `SYSTEM_PROMPT` trong [`agent.py`](agent.py):**
   - Bổ sung chỉ dẫn rõ ràng về quy trình điều tra: Khi người dùng hỏi về tình trạng hoạt động/outage của một dịch vụ (dù có kèm theo mã incident cụ thể hay không), agent **bắt buộc phải gọi `get_status`** trước để kiểm tra trạng thái hiện tại (operational) của dịch vụ, sau đó mới gọi `get_incident_details` để đối chiếu thông tin sự cố.
   - Thêm quy định về ranh giới dữ liệu: Khi gặp lỗi truy vấn (như timeout, degraded) hoặc thiếu dữ liệu từ tool, agent tuyệt đối không được tự ý suy đoán hệ thống hoạt động bình thường theo định hướng của người dùng, mà phải nêu rõ dữ liệu chưa được xác nhận (giúp cải thiện thêm case H7).
2. **Hoàn thiện khả năng chịu lỗi trong [`providers.py`](providers.py):**
   - Bổ sung cơ chế tự động thử lại (retry) khi gặp các mã lỗi tạm thời từ mạng hoặc máy chủ Google (HTTP 429, 500, 502, 503, 504), đồng thời duy trì khoảng nghỉ 5.0 giây giữa các request (`_pace_request`) để không vượt quá giới hạn 15 RPM của Google AI Studio.

### Vì sao chọn thay đổi này:
- Thay vì can thiệp bằng code cứng (hardcode) ép buộc agent phải gọi tool gì theo từng case cụ thể, việc tinh chỉnh System Prompt giúp mô hình hiểu đúng trách nhiệm nghiệp vụ của một trợ lý điều tra hệ thống: muốn khẳng định một hệ thống có đang sập hay không thì phải kiểm tra bảng trạng thái chung trước khi xem chi tiết một sự cố trong quá khứ. Cách này vừa tự nhiên, giữ được tính tổng quát của ReAct loop, vừa tuân thủ đúng quy định làm bài.

---

## 4. So sánh hai lần chạy

Bảng đối chiếu kết quả giữa lần chạy 1 (`eval_before.json`) và lần chạy 2 (`eval_after.json`) trên toàn bộ 22 cases với cấu hình đồng nhất (`provider: gemini`, `model: gemini-3.1-flash-lite`, `judge_model: gemini-3.1-flash-lite`, `max_steps: 8`, `snapshot: normal`):

| Chỉ số | `eval_before.json` | `eval_after.json` | Chênh lệch |
|---|---|---|---|
| **Điểm tổng (score_0_100)** | **91.14 / 100** | **94.77 / 100** | **+3.63 điểm** |
| **Điểm case H2** | **50 / 100** | **100 / 100** | **+50 điểm** |
| **Số lượt gọi model (Agent + Judge)** | 76 lượt (54 agent + 22 judge) | 75 lượt (53 agent + 22 judge) | -1 lượt |
| **Tổng số Tokens sử dụng** | 91,633 tokens | 101,951 tokens | +10,318 tokens |
| **Ước tính chi phí API** | ~$0.009 USD | ~$0.010 USD | ~$0.001 USD |

### Lỗi đã giải quyết:
- **Case H2 đạt 100/100:** Ở lần chạy 2, agent đã thực hiện đúng trình tự:
  1. Gọi `get_status(provider='github')` để kiểm tra trạng thái chung (xác nhận "All Systems Operational").
  2. Gọi `get_incident_details(provider='github', incident_id='t89g2q1800fr')` để kiểm tra sự cố (xác nhận đã giải quyết xong từ 05/11).
  $\rightarrow$ Bác bỏ hoàn toàn tiền đề sai của người dùng dựa trên bằng chứng xác thực từ cả hai phía, vi phạm cơ học giảm về 0 (`violations: []`).
- **Case H7 tăng từ 50 lên 100:** Agent kiên định từ chối kết luận bừa khi dữ liệu Cloudflare bị timeout, không bị ảnh hưởng bởi lời dẫn dụ của người dùng.
- **Case E1 tăng từ 95 lên 100:** Cung cấp đầy đủ và chuẩn xác hơn thông tin trích dẫn nguồn và thời điểm snapshot.

### Lỗi còn lại:
- **Case H1 (60 điểm):** Case này yêu cầu gọi tool ở ít nhất 2 bước model khác nhau (`min_tool_steps: 2`). Tuy nhiên mô hình Gemini có xu hướng gọi song song cả 2 tool (`parallel tool calls`) ngay trong bước 1 để tối ưu thời gian phản hồi, dẫn đến việc bị trừ điểm cơ học ở tiêu chí này dù nội dung phân tích rất chính xác.
- **Case E3 (70 điểm):** LLM Judge chấm tương đối khắt khe ở phần giải thích về ranh giới địa lý của Spectrum, dù logic suy luận cốt lõi của agent vẫn đáp ứng đúng bản chất câu hỏi.

---

## 5. Giới hạn

1. **Giới hạn của LLM Judge:**
   - Việc dùng chính LLM để chấm điểm mang lại sự linh hoạt nhưng vẫn tồn tại sai số ngẫu nhiên (nhiễu từ quá trình sinh từ/sampling). Ở một số câu hỏi có ranh giới đánh giá hẹp, điểm số giữa các lần chạy có thể lệch nhẹ khoảng $\pm 3$ đến $\pm 5$ điểm. Vì vậy, LLM Judge chỉ nên xem là công cụ định hướng, cần kết hợp với các hàm kiểm tra cơ học (`mechanical_checks`) bằng mã nguồn để đảm bảo tính công bằng.
2. **Giới hạn của dữ liệu Snapshot:**
   - Môi trường kiểm thử sử dụng dữ liệu tĩnh được đóng băng tại thời điểm chụp snapshot (03:50 UTC ngày 09/11/2026 trong `fixtures/github.json` và `fixtures/cloudflare.json`). Dữ liệu này giúp kết quả đánh giá có tính tái lập (reproducible), nhưng không phản ánh được tính biến động liên tục của các sự cố ngoài đời thực.
3. **Giới hạn của việc đánh giá một lần (Single Run Evaluation):**
   - Một lượt chạy hoàn chỉnh chỉ thể hiện hành vi của hệ thống tại một thời điểm nhất định. Trong môi trường thực tế, để đưa ra kết luận chắc chắn về độ ổn định của agent, cần thực hiện nhiều lần lặp (ví dụ 3 đến 5 lượt chạy) và tính toán khoảng tin cậy thống kê để loại bỏ các dao động ngẫu nhiên của mô hình ngôn ngữ lớn.
