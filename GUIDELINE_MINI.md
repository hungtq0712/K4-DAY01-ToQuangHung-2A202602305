# Mini annotation guideline — Ngày 3 (tracking)

> Điền file này **trong lúc gán nhãn**, không phải sau khi xong. Mỗi lần bạn dừng
> lại nghĩ "cái này tính sao nhỉ?" thì đó là một dòng phải ghi vào đây.
>
> Đây là tài liệu mà người gán nhãn tiếp theo sẽ đọc để làm giống bạn. Nếu hai
> người trong nhóm gán khác nhau, gần như luôn là vì file này chưa nói rõ — chứ
> không phải vì ai kém.

Nhóm / tên: Tô Quang Hưng
Clip: `clip_01`, `clip_02`

---

## 1. Phạm vi: gán cái gì, không gán cái gì

Một lớp duy nhất: **`vehicle`** — xe bốn bánh (xe con, van, xe buýt, xe tải).

| Gán                            | Không gán                                                   |
| ------------------------------- | ------------------------------------------------------------- |
| xe con, SUV, taxi, xe bán tải | người đi bộ                                               |
| van, minivan                    | xe đạp                                                      |
| xe buýt, minibus               | **xe máy / mô tô**                                   |
| xe tải, xe đầu kéo          | xe trong ảnh quảng cáo, trong gương, dưới bóng nước |


## 2. Luật ID — phần quan trọng nhất

| Tình huống                          | Luật của nhóm                                                                                             | Vì sao                                                                                                                                                      |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Xe bị che một phần rồi hiện lại | giữ nguyên ID nếu bị che**dưới ... frame** (mặc định của lab: 25 frame = 2 giây @ 12.5 fps) | Tránh phân mảnh ID (ID Switch) cho các xe bị che khuất ngắn hạn bởi cây cối hoặc cột đèn.                                                     |
| Xe bị che lâu hơn ngưỡng trên   | Đánh ID mới.                                                                                              | Quá thời gian quy định hoặc khi xe biến mất quá lâu, ReID và chuyển động dễ bị sai lệch nên gán ID mới để đảm bảo tính nhất quán. |
| Xe rời khung hình rồi quay lại    | mặc định:**track mới**                                                                             | Theo tiêu chuẩn chung, khi vật thể hoàn toàn ra khỏi góc máy và quay trở lại thì được tính là một thực thể xuất hiện mới.            |
| Hai xe cắt nhau / chồng lên nhau   | Giữ nguyên ID gốc của mỗi xe trước và sau khi giao cắt.                                             | Cần bám sát quỹ đạo của từng xe dựa trên hướng chuyển động tuyến tính và đặc điểm ngoại quan.                                         |

## 3. Luật bbox

| Tình huống                                   | Luật của nhóm                                                                                                                                           |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Xe bị cắt bởi rìa ảnh                     | bbox chạm đúng rìa, không đoán phần ngoài ảnh                                                                                                    |
| Xe bị xe khác che một phần                 | bbox ôm phần**nhìn thấy được**                                                                                                                |
| Xe vừa xuất hiện, còn rất nhỏ / rất mờ | bắt đầu track từ frame đầu tiên xác định được là xe bốn bánh; ngưỡng nhóm chọn: 25 frame = 2 giây @ 12.5 fps                          |
| Xe đang đỗ, không di chuyển               | Vẫn gán nhãn và giữ nguyên ID xuyên suốt nếu xe vẫn xuất hiện rõ ràng trong khung hình.                                                     |
| Keyframe đặt dày ở đâu                   | Đặt keyframe dày hơn (mỗi 2-3 frame) ở các đoạn xe chuyển hướng nhanh, tăng/giảm tốc đột ngột, hoặc khi bắt đầu/kết thúc occlusion |

## 4. Ít nhất ba ca mơ hồ đã gặp thật

Ghi **frame cụ thể** và **ID cụ thể**, không ghi chung chung.

### Ca 1

- Clip / frame / ID: clip_01 / Frame 107 / ID 24, 28
- Tình huống: Xe bị xe buýt che khuất một phần lớn (Partial Occlusion). Model tách ID 24 thành ID 28.
- Quyết định: Giữ nguyên ID 24 cho đến khi xe biến mất hoàn toàn hoặc hiện ra lại.
- Lý do: Duy trì tính nhất quán của đối tượng (Identity Consistency) thay vì tạo ID mới khi detector bị nhiễu do vật cản.

### Ca 2

- Clip / frame / ID: clip_01 / Frame 87 / ID 17, 18
- Tình huống: Model bắt được vật thể nhỏ (T7) ở xa, nhưng mắt người khó xác định đó là xe hay vật thể tĩnh trên đường.
- Quyết định:Chỉ gán nếu vật thể có sự di chuyển rõ rệt qua ít nhất 10 frames.
- Lý do: Khi ngoại hình bị lẫn, vector chuyển động (motion) là cue tin cậy hơn.

### Ca 3

- Clip / frame / ID: clip_01 / Frame 16-116 / ID 7
- Tình huống: Xe bị che khuất tạm thời bởi một cột biển báo giao thông lớn.
- Quyết định: Tiếp tục duy trì ID cũ sau khi xe lộ diện lại ở frame 106, thực hiện nội suy tuyến tính hộp giới hạn qua các frame bị che.
- Lý do: Tránh gán nhầm các vật thể tĩnh (False Positive) vào tập dữ liệu tracking.

## 5. Sửa gì sau khi chấm với gold và sau khi kiểm chéo

Luật nào trong file này hoá ra còn thiếu hoặc còn mơ hồ? Viết lại cho rõ:


1. **Quy tắc nhất quán ID (Identity Consistency):**
   * *Cũ:* Đảm bảo mỗi xe một ID.
   * *Viết lại:* Tại các điểm giao cắt hoặc đi sát nhau, tâm của bbox phải luôn bám sát trọng tâm của cùng một đối tượng. Tuyệt đối không để tâm bbox nhảy sang đối tượng bên cạnh dù chỉ 1 frame (tránh ID Switch như trường hợp ID 17/18 tại frame 87).

2. **Quy tắc kích thước bbox tối thiểu (Small Objects):**

* *Cũ:* Gán hết các xe trong hình.
* *Viết lại:* Chỉ gán nhãn các phương tiện có kích thước tối thiểu 20x20 pixel. Đối với xe ở quá xa và mờ, nếu không thể xác định rõ loại phương tiện hoặc không có sự di chuyển rõ rệt qua 10 frame, hãy bỏ qua để tránh tạo ra nhiễu False Positive.
