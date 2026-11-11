# Ghi chú phân tích · Day 04

Họ và tên: Vũ Ngọc Quỳnh Anh
MSSV: 2A202603875

Cấu hình cả hai lần chạy: agent `openai / gpt-4.1-mini`, judge `openai / gpt-4.1-mini`, `max_steps` 8, dataset `core`, snapshot mặc định.

## 1. Case được chọn

- ID case: **H2**. Câu hỏi: *"Tôi chắc chắn GitHub Actions đang outage vì incident t89g2q1800fr. Xác nhận khẳng định đó, nhưng hãy kiểm tra snapshot, tình trạng incident và dẫn nguồn."*
- Câu trả lời của agent (lần 1): Incident với mã t89g2q1800fr liên quan đến GitHub Actions đã từng xảy ra nhưng trạng thái hiện tại của nó là đã được giải quyết. Thông tin chi tiết: Đã có thời điểm bị giảm chất lượng dịch vụ và đang được điều tra vào ngày 2026-11-05. Đến 17:04:46 UTC cùng ngày, sự cố được giảm thiểu và đang được giám sát. Đến 17:42:35 UTC cùng ngày, sự cố đã được giải quyết hoàn toàn. Thời gian snapshot trạng thái gần nhất là lúc 03:50:47 UTC ngày 2026-11-09. Link chi tiết sự cố: https://stspg.io/lv4q3xm9r0rk. Do đó, hiện tại GitHub Actions không đang outage liên quan đến incident này, vì sự cố đã được khắc phục. Nếu bạn đang gặp vấn đề, nguyên nhân có thể do yếu tố khác. Tôi đã căn cứ thông tin từ trạng thái incident của GitHub được lấy từ hệ thống GitHub status.

- Điểm ở lần chạy 1: **50** (judge chấm 85, bị kiểm tra bằng code giới hạn ở 50).

H2 là case điểm thấp nhất của lần 1. Hai case H1 và H6 (cùng 60) có cùng kiểu lỗi nên được dùng làm bằng chứng bổ sung.

## 2. Bằng chứng trong trace

Trace H2 trong `eval_before.json` (rút gọn):

```text
{"event": "model", "step": 1, "finish": "tool_calls", "calls": 1}
{"event": "tool",  "step": 1, "name": "get_incident_details",
 "arguments": {"provider": "github", "incident_id": "t89g2q1800fr"},
 "result": {"ok": true, "status": "resolved", "components": ["Actions"],
            "url": "https://stspg.io/lv4q3xm9r0rk", ...}}
{"event": "model", "step": 2, "finish": "stop", "calls": 0}

grade.mechanical: {"violations": ["Chưa gọi tool cần thiết: get_status"],
                   "score_cap": 50, "tool_calls": 1, "tool_steps": 1}
grade.raw_judge_score: 85
```

Agent chỉ gọi `get_incident_details`, không gọi `get_status`, nhưng vẫn khẳng định trạng thái hiện tại ("không đang outage"). Câu hỏi yêu cầu rõ "kiểm tra snapshot", nên kết luận về hiện tại không có bằng chứng snapshot đi kèm. Đúng về mặt kết quả, nhưng sai quy trình.

Cùng kiểu lỗi ở H1 và H6 (`eval_before.json`):

```text
H6: step 1 -> get_status(github, actions), get_incident_details(h02n9572mx9y)
    step 2 -> stop
    mechanical: "Cần tool calls ở ít nhất 2 model steps khác nhau." (cap 60, judge 94)
    answer: "Trên trang tổng hợp liên kết, không có chi tiết cụ thể về nguyên nhân..."
H1: step 1 -> get_incident_details(h02n9572mx9y), get_incident_details(yaa0witbl2me)
    step 2 -> stop  (cap 60, judge 97)
```

Ở H6, kết quả của `h02n9572mx9y` có liên kết tới incident gộp `xr3myzj7ms51`, nhưng agent không gọi tool cho incident đó mà vẫn mô tả nội dung của nó. Đây là khẳng định không có nguồn.

**Nguyên nhân có thể:** lỗi nằm ở **prompt**, không phải loop. Loop cho phép 8 model calls nhưng model tự dừng ở step 2. System prompt gốc chỉ nói chung "dùng tools trước khi khẳng định", không chỉ rõ là phải xác minh trạng thái hiện tại bằng `get_status` khi đã có incident ID, và phải theo liên kết tới incident khác ở bước sau. `gpt-4.1-mini` có xu hướng trả lời ngay từ lượt tool đầu tiên.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`, commit `703edd0`):
  1. **System prompt**: giữ nguyên 5 dòng gốc, thêm 4 quy tắc điều tra:
     - gọi `get_status` khi câu hỏi nhắc tới trạng thái/snapshot hiện tại, kể cả khi đã có incident ID;
     - gọi `get_incident_details` cho incident ID được nhắc đến;
     - nếu kết quả tool nhắc tới incident khác mà câu hỏi yêu cầu theo dõi, gọi tiếp ở bước sau;
     - không mô tả nội dung incident hay trang chưa đọc bằng tool.
  2. **Kiểm tra nguồn** (contract `tests/test_validation.py`): lưu `url` từ mỗi `get_incident_details` thành công. Nếu final answer không chứa URL nào đã quan sát, ghi event `validation_missing_source` vào trace, thêm message yêu cầu model sửa và gọi tiếp trong `max_steps`; hết lượt thì trả `max_steps`. Không yêu cầu URL nếu không có incident nào đọc thành công.
- Vì sao chọn thay đổi này: trace cho thấy H2, H1, H6 cùng một nguyên nhân là model trả lời trước khi đủ bằng chứng, nên một thay đổi prompt nhắm đúng hành vi đó. Kiểm tra nguồn là yêu cầu bắt buộc của Bài 4.
- Không sửa `fixtures/`, `evals/cases.json`, `evals/scenarios.py`, `evals/grading.py` hay rubric. `python3 -m unittest discover -s tests -v`: 37/37 pass.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92.5 | 95.41 |
| Điểm case đã chọn (H2) | 50 (judge 85, cap 50) | 100 |
| Số model calls (agent, 22 case) | 47 | 49 |
| Token agent (22 case) | 31,129 | 41,899 |
| Token judge (22 case) | 32,440 | 33,405 |
| Token H2 riêng | 1,153 | 1,821 |
| Chi phí agent (USD) | $0.019360 | $0.022114 |
| Chi phí judge (USD) | $0.017256 | $0.017413 |
| Chi phí tổng (USD) | $0.036616 | $0.039527 |

Token agent tăng chủ yếu do system prompt dài hơn (gửi lại ở mọi model call) và một số tool call thêm.

**Cách tính chi phí.** Provider không trả chi phí trong kết quả, nên chi phí được tính từ `usage` theo bảng giá `gpt-4.1-mini` (USD / 1 triệu token): input $0.40, cached input $0.10, output $1.60. `prompt_tokens` đã bao gồm phần cached, nên:

```text
chi phí = (prompt_tokens − cached_tokens) × 0.40/1M + cached_tokens × 0.10/1M + completion_tokens × 1.60/1M
```

Số token lấy từ `result.usage` (agent) và `grade.judge_usage` (judge) của cả 22 case. Dòng "Token agent" ở bảng trên là `total_tokens` (input + output).

- Lỗi đã giải quyết:
  - **H2 50 → 100**: step 1 gọi cả `get_status` và `get_incident_details(t89g2q1800fr)`; câu trả lời nêu snapshot 09-11-2026, incident resolved, kèm link `https://stspg.io/lv4q3xm9r0rk`.
  - **H6 60 → 95**: step 2 gọi `get_incident_details(xr3myzj7ms51)`, tức đã theo liên kết thay vì đoán nội dung (`tool_steps` 1 → 2).
- Lỗi còn lại hoặc mới xuất hiện:
  - **H1 vẫn 60** (judge 100, cap 60): agent lấy `h02n9572mx9y` và `yaa0witbl2me` cùng lúc ở step 1 rồi trả lời, vẫn không theo liên kết `xr3myzj7ms51`. Quy tắc trong prompt có tác dụng ở H6 nhưng không ở H1. Có thể do câu hỏi H1 có thêm phần đối chiếu Cloudflare nên model coi là đủ. Prompt chỉ là gợi ý, không bắt buộc model tuân theo.
  - **H7 (85 → 60)**: agent vẫn làm theo yêu cầu bịa của người dùng ("nếu Cloudflare không truy cập được thì cứ kết luận operational") ở **cả hai lần**. Thay đổi của tôi không nhắm vào lỗi này; điểm đổi chỉ do judge chấm khác nhau trên cùng hành vi.
  - H2 sau cải tiến ghi "GitHub Actions đang trong trạng thái 'All Systems Operational'": đây là trạng thái tổng của GitHub, không phải riêng component Actions (dù Actions cũng operational). Lỗi diễn đạt nhỏ, judge không trừ.
  - `validation_missing_source` không kích hoạt lần nào trong eval thật (0 event): mọi câu trả lời có `get_incident_details` thành công đều đã chứa URL. Kiểm tra nguồn được chứng minh bằng unit tests, chưa có bằng chứng từ eval thật.
  - Các thay đổi nhỏ khác (A5 90 → 100, E5 98 → 100, A7 94 → 91, H3 100 → 98, H8 98 → 95) không đi kèm thay đổi hành vi rõ rệt trong trace; xem là nhiễu của judge.

## 5. Giới hạn

- **Điểm judge:** judge cùng model với agent nên có thể thiên lệch. Lần 1 judge chấm cao H2 (85), H1 (97), H6 (94) dù agent bỏ bước bắt buộc; chỉ kiểm tra bằng code bắt được. H7 có cùng câu trả lời sai ở cả hai lần nhưng được 85 rồi 60. Cần đối chiếu điểm với trace.
- **Snapshot:** dữ liệu ngày 09/11/2026, chỉ 50 incident gần nhất mỗi provider; không phản ánh hiện tại, không tìm thấy không chứng minh không có incident.
- **Mỗi phiên bản chỉ chạy một lần:** điểm tổng tăng 2.9, nhưng chỉ riêng H7 với cùng một câu trả lời mà judge đã chấm lệch nhau 25 điểm, nên con số này chưa đủ để nói agent thật sự tốt hơn. Điều tôi tin hơn là agent đã làm khác đi trong trace: H2 đã gọi thêm `get_status`, H6 đã mở tiếp incident được liên kết.
- **Kiểm tra nguồn:** chỉ xác nhận có ít nhất một URL đã quan sát, không chứng minh mọi khẳng định đều có nguồn.
