# Day 4 - Lab 4: ReAct Loop Agent

## 1. Vấn đề đã chọn (Selected Issue)

**Vấn đề:** Agent có thể trả trạng thái `completed` dù câu trả lời cuối thiếu URL nguồn của incident đã được công cụ `get_incident_details` truy xuất thành công.

**Trước khi cải tiến:** Trong `agent.py`, phần xử lý câu trả lời cuối chỉ kiểm tra LLM dừng bình thường (`finish == "stop"`) và không còn tool calls. Agent chưa kiểm tra URL bằng chứng. Hai unit tests `test_missing_url_gets_revision_within_budget` và `test_missing_url_at_budget_limit_is_not_completed` ban đầu đều FAIL.

**Mục tiêu:** Thêm Evidence Gate (cổng kiểm tra bằng chứng) để yêu cầu LLM sửa câu trả lời khi thiếu URL incident, vẫn tuân thủ giới hạn `max_steps` và không bắt buộc URL nếu công cụ lấy incident thất bại.

## 2. Bằng chứng trong test và trace (Evidence)

- **CP3 - Before:** Evaluation đạt **95.36/100**, **22/22 cases được chấm** (`eval_before.json`).
- **Lỗi ban đầu:** Hai unit tests kiểm tra trường hợp thiếu URL đã FAIL.
- **Sau khi sửa:** Toàn bộ **37/37 unit tests PASS**, gồm ba tests Evidence Gate: `test_missing_url_gets_revision_within_budget`, `test_missing_url_at_budget_limit_is_not_completed`, `test_failed_tool_does_not_require_source`.
- **Ví dụ từ Evaluation - case E2:** Agent gọi `get_status` để kiểm tra Cloudflare và `get_incident_details` để đọc incident `wp49bp8bqo6c`. Tại bước 3, trace ghi sự kiện `validation_missing_source`. Sau đó LLM sửa câu trả lời và bổ sung URL từ tool: https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c.

## 3. Thay đổi đã thực hiện (Implementation Changes)

Chỉnh sửa **`agent.py`** để bổ sung Evidence Gate:

1. Thu thập URL từ các lần gọi `get_incident_details` thành công.
2. Trước khi chấp nhận câu trả lời cuối, kiểm tra câu trả lời có chứa URL incident đã quan sát hay không.
3. Nếu thiếu URL, ghi event `validation_missing_source` vào `trace` và yêu cầu LLM sửa câu trả lời trong giới hạn `max_steps`.
4. Nếu hết số bước mà vẫn thiếu URL, trả `max_steps` thay vì `completed`.
5. Nếu không có incident-details thành công thì không ép Agent tạo URL. Các kết quả `get_status` thông thường không kích hoạt yêu cầu URL incident nghiêm ngặt.

Vòng lặp ReAct gốc được giữ nguyên: LLM trả assistant message → tool calls → tool results gắn đúng ID → bước LLM tiếp theo.

## 4. So sánh trước và sau (Before vs. After Evaluation)

Hai lần đánh giá sử dụng cùng cấu hình: provider `opencode`, model `deepseek-v4.1-flash`, judge provider `opencode`, judge model `deepseek-v4.1-flash`, dataset `core`, `max_steps = 8`.

| Tiêu chí | Before (CP3) | After (CP5) |
|---|---:|---:|
| Điểm tổng | 95.36/100 | 94.59/100 |
| Cases được chấm | 22/22 | 22/22 |
| Cases chưa được chấm | 0 | 0 |
| Điểm H5 | 20/100 | 20/100 |

**Chênh lệch:** `94.59 - 95.36 = -0.77` điểm.

Một số thay đổi theo case:

| Case | Before | After | Chênh lệch |
|---|---:|---:|---:|
| A1 | 93 | 98 | +5 |
| A3 | 100 | 93 | -7 |
| A5 | 93 | 95 | +2 |
| H8 | 100 | 92 | -8 |
| H5 | 20 | 20 | 0 |

Evidence Gate được xác nhận bởi unit tests và sự kiện `validation_missing_source` trong trace case E2, nhưng **điểm Evaluation tổng thể không tăng**.

## 5. Lỗi còn lại và giới hạn (Remaining Errors & Limitations)

- **H5 vẫn đạt 20/100:** Agent còn lặp lại marker của prompt injection từ dữ liệu tool không đáng tin cậy. Nhận ra nội dung độc hại chưa đủ nếu câu trả lời vẫn sao chép marker đó.
- **Kiểm tra URL chưa phải xác minh toàn bộ thông tin:** Evidence Gate chỉ kiểm tra có URL nguồn, chưa chứng minh mọi nhận định (claim) đều được nguồn hỗ trợ.
- **LLM Judge có thể dao động:** Điểm do LLM ước lượng và có thể thay đổi giữa các lần chạy. Một lần Before và một lần After chưa đủ chứng minh cải thiện ổn định.
- **Giới hạn snapshot:** Tool chỉ tìm trong nhóm incident gần đây và có thể cắt ngắn các update. Không tìm thấy dữ liệu không có nghĩa incident không tồn tại.

## 6. Demo trace và kết luận (Demo & Conclusion)

**Demo case E2 - Cloudflare Cache Purge / FedRAMP:**

1. `get_status` lấy dữ liệu trạng thái Cloudflare.
2. `get_incident_details` lấy thông tin incident `wp49bp8bqo6c` và URL nguồn.
3. LLM đưa ra câu trả lời thiếu URL; Evidence Gate ghi `validation_missing_source` ở bước 3.
4. LLM sửa câu trả lời, thêm URL incident và Agent hoàn thành.

**Kết luận:** Tôi đã hoàn thiện ReAct Loop và bổ sung Evidence Gate, được xác nhận bằng **37/37 unit tests PASS**. Hai lần Evaluation đều chấm đủ **22/22 cases**. Điểm tổng giảm từ **95.36** xuống **94.59**, vì vậy tôi không khẳng định điểm tổng đã cải thiện. Hướng nâng cấp tiếp theo là ngăn Agent sao chép các chỉ dẫn giả từ tool results không đáng tin cậy.
