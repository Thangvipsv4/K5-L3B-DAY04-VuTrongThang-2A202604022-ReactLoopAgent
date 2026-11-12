
# Ghi chú phân tích · Day 04 – Status Investigation Agent

Họ và tên: Vũ Trọng Thắng
MSSV: 2A202604022

## 1. Case được chọn

- ID case: A5
- Câu hỏi: Yêu cầu danh sách đầy đủ tất cả incident GitHub Actions trong năm 2025 và phải nêu rõ độ đầy đủ của dữ liệu.
- Điểm ở lần chạy 1: 35/100.
- Vấn đề: Agent khẳng định không có incident GitHub Actions trong năm 2025 dù dữ liệu chỉ chứa tối đa 50 incident gần nhất.

## 2. Bằng chứng trong trace

Ở lần chạy đầu tiên (`eval_before.json`), Agent thực hiện:

- Gọi `get_status` với provider GitHub và component Actions.
- Snapshot ghi nhận thời điểm `2026-11-09T03:50:47.065020+00:00`.
- Tool trả về ghi chú: chỉ tìm kiếm trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố.
- Có 23 incident phù hợp với component Actions trong phạm vi tìm kiếm 50 incident gần nhất.
- Các incident được trả về thuộc năm 2026, không đủ để xác nhận danh sách incident trong năm 2025.

Tuy nhiên, Agent kết luận rằng không có incident GitHub Actions nào trong năm 2025.

Nguyên nhân có thể xảy ra: Agent chưa phân biệt được việc không tìm thấy dữ liệu trong phạm vi tìm kiếm giới hạn với việc chứng minh một sự kiện không tồn tại.

## 3. Thay đổi đã thực hiện

### Thay đổi thứ nhất – cải thiện giới hạn bằng chứng

Sửa `SYSTEM_PROMPT` trong `agent.py`:

- Nêu rõ tool chỉ có thể cung cấp tối đa 50 incident gần nhất.
- Không kết luận không có incident chỉ vì không tìm thấy trong dữ liệu giới hạn.
- Không khẳng định danh sách đầy đủ nếu dữ liệu không bao phủ toàn bộ khoảng thời gian.
- Phân biệt thời điểm snapshot với thời điểm incident xảy ra, được tạo hoặc cập nhật.
- Không thay thế incident trong khoảng thời gian được hỏi bằng incident ở thời gian khác.

Kết quả: A5 tăng từ 35 lên 75 điểm trong lần chạy thứ hai.

### Thay đổi thứ hai – kiểm tra nhiều nguồn

Bổ sung quy tắc vào `SYSTEM_PROMPT`:

- Khi người dùng hỏi về incident cụ thể và đồng thời yêu cầu xác nhận trạng thái hệ thống hoặc component hiện tại, Agent phải gọi cả `get_incident_details` và `get_status`.
- Dùng `get_incident_details` để kiểm tra incident cụ thể.
- Dùng `get_status` để kiểm tra trạng thái tổng thể và component tại snapshot.
- Không suy luận trạng thái toàn hệ thống từ trạng thái của một incident.

Kết quả: H2 tăng từ 50 lên 100 điểm trong lần chạy thứ ba.

Các thay đổi chỉ tác động đến `SYSTEM_PROMPT`; giữ nguyên cấu trúc Agent Loop và cơ chế kiểm tra URL nguồn.

## 4. So sánh các lần chạy

### Lần chạy 1 và lần chạy 2

| Chỉ số | eval_before.json | eval_after.json |
|---|---:|---:|
| Điểm tổng | 80,00/100 | 82,73/100 |
| Case được chấm | 22/22 | 22/22 |
| A5 | 35 | 75 |
| Agent model calls | 53 | 48 |
| Agent tokens | 44.799 | 48.914 |
| Judge tokens | 40.605 | 35.796 |
| Tổng tokens | 85.404 | 84.710 |
| Tool calls | 47 | 35 |

Cấu hình chung:
- Provider: OpenAI
- Model: gpt-4o-mini
- Judge model: gpt-4o-mini
- Max steps: 8
- Dataset: core

Nhận xét:
- Điểm tổng tăng 2,73 điểm.
- A5 tăng 40 điểm.
- Số Agent model calls giảm 5.
- Tổng token giảm 694.
- Số tool calls giảm 12.

### Lần chạy 3 – eval_final.json

| Chỉ số | Kết quả |
|---|---:|
| Tổng số case | 22 |
| Case được chấm | 20 |
| Case chưa được chấm | 2 |
| Điểm tổng | Không có |
| A5 | 55 |
| H2 | 100 |

Lần chạy thứ ba không có điểm tổng hợp lệ do E2 và E3 bị unscored.

H2 tăng từ 50 lên 100 sau khi bổ sung quy tắc kiểm tra nhiều nguồn. A5 giảm từ 75 xuống 55, cho thấy điểm Judge có dao động giữa các lần chạy.

Không sử dụng trung bình của 20 case để thay thế điểm tổng chính thức.

### Đánh giá chung

- Lỗi A5 được cải thiện đáng kể sau thay đổi đầu tiên.
- H2 được cải thiện sau thay đổi thứ hai.
- Một số case khác vẫn có điểm dao động.
- Chưa thể kết luận Agent cải thiện đồng đều trên toàn bộ dataset.

## 5. Giới hạn

1. **LLM Judge:** Điểm được đánh giá tự động bằng LLM Judge nên có thể dao động giữa các lần chạy và không hoàn toàn phản ánh chất lượng thực tế.

2. **Snapshot:** Dữ liệu sử dụng snapshot tại `2026-11-09T03:50:47.065020+00:00`, không phải dữ liệu trạng thái trực tiếp tại thời điểm chạy đánh giá.

3. **Số lần chạy:** Mỗi phiên bản chỉ có một lần chạy eval chính thức. Kết quả chưa đủ để khẳng định tính ổn định hoặc khả năng tổng quát hóa.

4. **Lần chạy thứ ba:** E2 và E3 không được chấm nên không có điểm tổng hợp lệ.

5. **Token và chi phí:** Token được thống kê từ Agent và Judge. Chưa có dữ liệu billing thực tế để xác định chi phí bằng tiền.

6. **Kiểm tra nguồn:** Cơ chế xác thực URL chỉ kiểm tra URL incident đã được tool trả về có xuất hiện trong câu trả lời, không chứng minh mọi thông tin trong câu trả lời đều chính xác.

7. **Giới hạn tìm kiếm:** Tối đa 50 incident gần nhất không đủ để chứng minh danh sách lịch sử đầy đủ cho mọi khoảng thời gian.
