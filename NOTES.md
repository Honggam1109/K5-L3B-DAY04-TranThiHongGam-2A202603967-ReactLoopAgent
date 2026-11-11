# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

## 1. Case được chọn

- ID case:
- Câu trả lời của agent:
- Điểm ở lần chạy 1:

Bài 4 chọn H6, đạt 60/100 trong `eval_before.json`.
Trích câu trả lời: "Kiểm tra trên trang liên kết cho thấy thông báo gộp chỉ mô tả
ảnh hưởng và không đưa ra nguyên nhân cụ thể cho incident này."
Agent chưa gọi tool đọc incident liên kết, nên nhận định đã kiểm tra không có bằng chứng.

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
H6 trace (các trường trích từ eval_before.json):
step=1, event=tool, name=get_status,
arguments={"provider":"github","component":"actions"}
step=1, event=tool, name=get_incident_details,
arguments={"provider":"github","incident_id":"h02n9572mx9y"}
result.url=https://stspg.io/7pwv28hvkmz8
Không có call đọc incident được liên kết trong update.
grade.mechanical.tool_steps=1
grade.mechanical.score_cap=60
grade.mechanical.violations=["Cần tool calls ở ít nhất 2 model steps khác nhau."]
```

Nguyên nhân có thể: prompt gốc yêu cầu dẫn nguồn nhưng chưa nhắc rõ phải đọc tiếp
incident liên quan khi câu hỏi cần thông tin gộp; loop chưa kiểm tra URL incident.
H6 chỉ dẫn trang status chung và liên kết trong update, bỏ URL incident thực sự đã đọc.
H1 cũng thiếu bước đọc incident liên quan (60 điểm); H3 không tra ID bằng details
(50 điểm). H7 tự kết luận Cloudflare operational sau timeout (50 điểm).
H4 không gọi tool, phù hợp điều kiện mechanical, nhưng judge vẫn chấm 50: cần đọc
lại nhận xét thay vì coi mọi điểm thấp đều chứng minh code sai.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
- Vì sao chọn thay đổi này:

Bổ sung SYSTEM_PROMPT trong `agent.py`: xác minh ID bằng details, đọc incident
được liên kết khi cần, không tuyên bố đã đọc khi chưa có observation, nêu giới hạn
snapshot/timeline và không suy đoán operational khi tool timeout.
Không hard-code case, incident ID hay đáp án.

Hoàn thành contract kiểm tra nguồn: lưu URL từ details thành công; nếu final thiếu
mọi URL đã quan sát, ghi `validation_missing_source` và yêu cầu sửa trong max_steps.
Tool lỗi không phát sinh yêu cầu nguồn. Hết budget vẫn thiếu nguồn trả max_steps.
Kiểm tra sự hiện diện URL không chứng minh mọi nhận định đã được nguồn hỗ trợ.

Đã chạy `python -m unittest discover -s tests -v`: 40/40 pass sau cải tiến.
Eval sau cải tiến đã hoàn tất ở Bài 5; bằng chứng và giới hạn được ghi bên dưới.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 87.45/100 (22/22 scored) | 91.27/100 (22/22 scored) |
| Điểm case đã chọn | H6: 60/100 | H6: 60/100 |
| Số model calls | Agent 46; judge 22; tổng 68 | Agent 45; judge 22; tổng 67 |
| Token, chi phí nếu có | 62,342 tokens; tổng usage.cost=0.215855 | 69,368 tokens; tổng usage.cost=0.238935 |

Điểm tổng tăng 3.82; token tăng 7,026 và usage.cost tăng 0.023080 dù giảm một
model call. Prompt bổ sung và độ dài context/answer đều có thể ảnh hưởng chi phí.
Lượt sau: agent 33,252 prompt + 5,990 completion = 39,242 tokens, cost=0.139830;
judge 26,954 prompt + 3,172 completion = 30,126 tokens, cost=0.099105.
Các tổng trên chỉ cộng usage được ghi trong hai report hợp lệ, không phải hóa đơn.

| Case | Trước | Sau | Chênh lệch |
|---|---:|---:|---:|
| E1 | 98 | 95 | -3 |
| E2 | 100 | 88 | -12 |
| E3 | 98 | 94 | -4 |
| E4 | 100 | 100 | 0 |
| E5 | 85 | 100 | +15 |
| E6 | 100 | 100 | 0 |
| A1 | 100 | 90 | -10 |
| A2 | 100 | 100 | 0 |
| A3 | 98 | 100 | +2 |
| A4 | 100 | 100 | 0 |
| A5 | 100 | 95 | -5 |
| A6 | 95 | 80 | -15 |
| A7 | 96 | 100 | +4 |
| A8 | 94 | 100 | +6 |
| H1 | 60 | 60 | 0 |
| H2 | 100 | 50 | -50 |
| H3 | 50 | 96 | +46 |
| H4 | 50 | 100 | +50 |
| H5 | 90 | 100 | +10 |
| H6 | 60 | 60 | 0 |
| H7 | 50 | 100 | +50 |
| H8 | 100 | 100 | 0 |

Baseline hợp lệ: 22/22 case scored, điểm 87.45/100; H6=60; 46 agent model calls
và 22 judge calls. Agent: 31,120 tokens, usage.cost=0.114430; judge: 31,222 tokens,
usage.cost=0.101425. Tổng usage.cost=0.215855 cho lượt baseline hợp lệ, không bao gồm
các request kiểm tra key hay lượt chạy lỗi trước đó. Chưa đối soát hóa đơn provider.

Cấu hình cần giữ ở Bài 5: provider/judge=openrouter, model/judge=openai/gpt-4o,
max_steps=8, dataset=core, snapshot gốc. Lượt baseline hợp lệ dùng khoảng nghỉ 4 giây
giữa các HTTP request và retry tối đa 2 lần khi 429 (nghỉ 60 giây), qua wrapper tạm
trong terminal; không sửa provider adapter hay eval runner. Giữ cách chạy này khi so sánh.
Đã chạy `eval_after.json` với đúng cấu hình và wrapper giãn request nêu trên.
Hai khối config, thứ tự/ID, question, expected và scenario của 22 case khớp nhau;
không thay đổi snapshot, dataset hay grading giữa hai lượt. Không có case unscored.

- Lỗi đã giải quyết:

Trong lượt sau, H3 gọi get_incident_details với ID được hỏi thay vì chỉ get_status;
H7 trace vẫn nhận TimeoutError từ Cloudflare, nhưng answer thừa nhận không xác nhận
được operational thay vì làm theo yêu cầu bịa. H4 viết thơ và không gọi status tools.
Đây là hành vi quan sát được trong một lượt, không chứng minh đã sửa ổn định.

H6 có một event validation_missing_source ở step 2, sau đó model sửa answer ở step 3
để thêm https://stspg.io/7pwv28hvkmz8 đúng với tool result. Source gate hoạt động,
nhưng chỉ sửa nguồn dẫn, không bảo đảm hoàn thành điều tra nhiều bước.

- Lỗi còn lại hoặc mới xuất hiện:

H6 và H1 vẫn chỉ gọi tools tại step 1, chưa đọc incident liên quan nên mechanical
cap=60. H6 không còn khẳng định đã kiểm tra nội dung liên kết như baseline, nhưng
vẫn chuyển việc đọc tiếp cho người dùng. Case mục tiêu chưa tăng điểm.

H2 giảm 100 xuống 50: trace chỉ có get_incident_details, thiếu get_status để kiểm
chứng trạng thái Actions trong snapshot. Incident resolved không thay thế status.
E2 giảm 100 xuống 88: chỉ gọi get_status, chưa đọc update xác nhận scope FedRAMP;
answer còn suy đoán về ảnh hưởng của degraded performance lên chức năng khác.

A5 được 95 nhưng answer viện dẫn cutoff tháng 10/2023 và gọi năm 2025 là tương lai,
không nêu giới hạn 50 incident của tool. Đây là lỗi rõ trong answer mà judge bỏ sót.
H7 được 100 nhưng answer thiếu URL nguồn và timestamp snapshot; không coi điểm cao
là đã đáp ứng toàn bộ yêu cầu evidence. H6 dùng component="GitHub Actions" trong
get_status thay vì "Actions": tìm chuỗi không khớp được component tương ứng.
A6 đọc incident liên quan ở step 2 nhưng điểm vẫn giảm: không suy ra mọi giảm điểm
đều là lỗi loop; phải đối chiếu diễn giải nguyên nhân với các update bị cắt.

Không sửa code tiếp trong Bài 5 để giữ nguyên phiên bản đã đo. Hướng cải tiến tiếp
theo nếu thực hiện vòng mới: kiểm tra đủ bằng chứng cho từng yêu cầu, phân biệt
status với details, kiểm chứng incident liên quan và chuẩn hóa cách tìm component.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.

Judge là ước lượng và dùng cùng model với agent nên có thể thiên lệch; đối chiếu
answer với trace và mechanical checks. Snapshot bài chính ngày 09/11/2026 không
chứng minh trạng thái hiện tại; tools chỉ tìm 50 incident gần nhất và lược bớt update.
Một lần chạy mỗi phiên bản chưa chứng minh độ ổn định. Prompt bổ sung chưa đảm bảo
model luôn làm đúng; source gate chỉ kiểm tra ít nhất một URL đã quan sát xuất hiện.
