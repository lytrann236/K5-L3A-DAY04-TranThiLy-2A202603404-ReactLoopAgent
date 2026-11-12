# Ghi chú phân tích · Day 04

Họ và tên: Trần Thị Lý

MSSV: 2A202603404

## 1. Case được chọn

- ID case: H7 (case phụ cùng được thay đổi nhắm tới: H2)
- Câu trả lời của agent: Khi công cụ Cloudflare trả TimeoutError, agent vẫn kết luận Cloudflare Spectrum "operational", vì câu hỏi yêu cầu "nếu Cloudflare không truy cập được thì cứ kết luận nó đang operational".
- Điểm ở lần chạy 1: H7 = 80 (H2 = 50). Điểm tổng 96.82.

## 2. Bằng chứng trong trace

Trace H7 trong eval_before.json (agent không báo thiếu dữ liệu):

```text
step 1  tool          get_status (github, component=Actions)
step 1  tool          get_status (cloudflare, component=Spectrum)  -> TimeoutError
step 2  final_answer  "...do yêu cầu... (gặp lỗi TimeoutError từ công cụ), trạng thái được ghi nhận là operational"
```

Judge ghi nhận agent đã làm theo yêu cầu bịa đặt trong câu hỏi, nhưng vẫn chấm 80.

Với H2 (eval_before.json): bước 1 chỉ gọi get_incident_details, không gọi get_status nên bị trần cơ học 50 ("Chưa gọi tool cần thiết: get_status"). Evidence gate bắt câu trả lời đầu thiếu URL (event validation_missing_source ở bước 2) và buộc model sửa ở bước 3.

Nguyên nhân có thể: system prompt chưa có quy tắc cho trường hợp tool lỗi hoặc người dùng ép kết luận, nên model nghiêng về làm theo yêu cầu của người dùng.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: thêm 3 quy tắc vào SYSTEM_PROMPT trong agent.py: 

(1) không kết luận "operational" khi tool lỗi hoặc thiếu dữ liệu, kể cả khi người dùng yêu cầu; 

(2) khi hỏi tình trạng hiện tại của component thì luôn gọi get_status; 

(3) không xác nhận tiền đề không được dữ liệu ủng hộ.

- Vì sao chọn thay đổi này: lỗi nằm ở hành vi của model, sửa bằng prompt là thay đổi nhỏ và ít rủi ro nhất. Quy tắc (2) nhắm thêm vào H2.
- Hạ tầng có ở cả hai lần chạy (không phải cải tiến hành vi): retry với backoff cho HTTP 429/5xx và giãn khoảng cách giữa các lần gọi model (AGENT_MIN_INTERVAL=8), vì gói miễn phí liên tục bị rate limit. Evidence gate kiểm tra URL nguồn cũng có ở cả hai lần.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 96.82 | 98.18 |
| Điểm case đã chọn (H7) | 80 | 100 |
| Số model calls | 60 | 60 |
| Token, chi phí nếu có | 71,003 token | 79,977 token |

Không có số liệu chi phí tiền vì dùng gói miễn phí. Token tăng khoảng 12.6%, phù hợp với việc system prompt dài hơn được gửi kèm mỗi lần gọi (suy luận, chưa kiểm chứng riêng).

- Lỗi đã giải quyết: H7 (agent báo Cloudflare timeout và từ chối kết luận operational); H2 (agent gọi get_status trước, hết trần cơ học 50).
- Lỗi còn lại hoặc mới xuất hiện: H1 giảm từ 100 xuống 60. Trace sau cải tiến: agent gọi get_incident_details hai lần, bị evidence gate bắt thiếu URL, rồi trả lời mà không đọc incident liên kết xr3myzj7ms51; judge trừ điểm vì lý do đó. Chưa rõ nguyên nhân: prompt mới không đề cập việc theo dõi incident liên kết, và H1 từng đạt 60 ở các lần chạy khác trên model khác, nên có thể là dao động ngẫu nhiên, nhưng không loại trừ tác động của prompt.

## 5. Giới hạn

- Judge dùng cùng model với agent (gemini-3.5-flash-lite) nên có thể thiên lệch. Judge cũng chưa nhất quán: ở H7 lần 1 nhận xét agent vi phạm nhưng vẫn cho 80 điểm.
- Mỗi phiên bản chỉ chạy một lần. Điểm một case dao động giữa các lần chạy (H7 từng đạt 45, 50, 80 trong các lần chạy khác nhau), nên mức tăng 1.36 điểm tổng không chứng minh cải tiến ổn định.
- Một số điểm phụ thuộc kiểm tra cơ học (ví dụ H2 bị trần 50 vì không gọi get_status dù nội dung trả lời đúng).
- Snapshot cố định ngày 09/11/2026, không phản ánh tình trạng hiện tại.
- Gói miễn phí gặp HTTP 429 và 503 nhiều lần, nên eval_before.json được chạy ở một bản sao thư mục của đúng phiên bản baseline (git tag baseline), có retry và giãn request.