# Báo cáo Ngày 3 — Tracking Annotation

Chép file này thành `reports/REPORT.md` rồi điền. Giữ nguyên các tiêu đề.

Họ tên / nhóm: Tô Quang Hưng
Ngày: 15/09/2026

---

## 1. Quá trình gán nhãn

| Mục                                 | Giá trị |
| ------------------------------------ | --------- |
| Công cụ                            | CVAT      |
| Thời gian gán`clip_02` (warm-up) | 45 phút |
| Thời gian gán`clip_01`           | 80 phút |
| Số track đã vẽ trong`clip_01`  | 8         |
| Số keyframe trung bình mỗi track  | 72        |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. **Vật thể bị che khuất một phần :** Tại các frame 105-112, chiếc xe buýt lớn che khuất các xe con phía sau. Cách xử lý: Gán nhãn dựa trên phần còn lộ ra và giữ nguyên ID khi vật thể xuất hiện trở lại.
2. **Vật thể ở xa và kích thước nhỏ:** Các xe ở phía làn đường xa (góc trên bên phải) rất nhỏ và mờ. Cách xử lý: Chỉ gán nhãn khi có thể xác định rõ đó là xe 4 bánh, chấp nhận bỏ qua nếu quá mờ để tránh tạo nhiễu.
3. **Nhầm lẫn giữa các ID khi giao cắt:** Khi các xe đi sát nhau hoặc vượt nhau, các hộp giới hạn (bbox) bị chồng lấp mạnh. Cách xử lý: Dùng tính năng tua đi tua lại trong CVAT để kiểm tra ID trước và sau khi giao cắt nhằm đảm bảo tính nhất quán.

## 2. Tự kiểm và kiểm chéo

Ba lượt tua bắt được gì (lượt 1 nhìn ID, lượt 2 frame đầu/cuối, lượt 3 frame giữa):

* **Lượt 1 (Nhìn ID):** Phát hiện 2 lỗi ID Switch quan trọng tại frame 87 (track 5 nhảy từ ID 17 sang 18) và frame 113 (track 6 nhảy từ ID 24 sang 31). Cần kiểm tra xem bạn đã giữ đúng ID xuyên suốt các frame này chưa.
* **Lượt 2 (Frame đầu/cuối):** Kiểm tra các xe đi vào và ra khỏi khung hình. Có sự bất đồng ở các frame cuối (như frame 190, track 1) nơi detector của model có IoU thấp (0.59) so với nhãn tay, có thể do xe chỉ còn lộ ra một phần nhỏ.
* **Lượt 3 (Frame giữa):** Tập trung vào các frame từ 105 đến 112. Đây là giai đoạn xe buýt che khuất các xe khác, dẫn đến việc model tạo ra các bbox thừa hoặc thiếu (FP/FN) so với nhãn của bạn. Bạn nên kiểm tra kỹ xem mình có bỏ sót vật thể nào khi chúng bị che khuất một phần ở giữa clip không.

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence                                               | Giá trị                                                             |
| ------------------------------------------------------ | --------------------------------------------------------------------- |
| SHA-256 từ`evidence/pre-gold/clip_01/manifest.json` | eb3ce6fb678724a1bab89c8198568b0c6698310829bb864a6a07b4561bb7d4f3      |
| Thời điểm khóa                                     | 12 giờ (2026-09-15T04:52:25.672833+00:00)                            |
| Số row / frame / track trước khi mở reference      | "rows": 635, "frames": 190, "track_ids": [ 1, 2, 3, 4, 5, 6, 7, 8 ] |

|               |   HOTA |   DetA |   AssA |   LocA |   IDF1 |   MOTA |   MOTP | FP | FN | IDSW |
| ------------- | -----: | -----: | -----: | -----: | -----: | -----: | -----: | -: | -: | ---: |
| Bản pre-gold | 0.8034 | 0.7827 | 0.8258 | 0.8941 | 0.9354 | 0.8639 | 0.8860 | 70 |  8 |    0 |
| Sau rework    | 0.8856 |  0.877 | 0.8951 | 0.8973 | 0.9974 | 0.9948 | 0.8878 |  3 |  0 |    0 |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **có**

Sau khi đọc danh sách lỗi, bạn đã sửa cụ thể những gì? Ghi theo frame và ID:

| Loại lỗi  | Frame   | ID         | Đã sửa thế nào                                                                                          |
| ----------- | ------- | ---------- | ------------------------------------------------------------------------------------------------------------ |
| Switch      | 87      | 17,18      | Kiểm tra track 5; thống nhất giữ ID 17 xuyên suốt frame 87 thay vì để nhảy sang 18.                |
| Bbox lệch  | 108     | 7          | Điều chỉnh lại kích thước bbox của track 7 tại frame 108 để khớp với thân xe hơn (tăng IoU). |
| Tách Track | 107-113 | 24, 28, 31 | Kiểm tra track 6; gộp các ID 28 và 31 về lại ID 24 ban đầu để đảm bảo tính liên tục.         |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục                               | Giá trị                                       |
| ---------------------------------- | ----------------------------------------------- |
| Python / ultralytics / torch / lap | 3.13.15 / 8.4.145 / 2.11.0+cu128 / 0.5.13       |
| weights / hai tracker              | yolo26n.pt / bytetrack.yaml & botsort-reid.yaml |
| conf / IoU / imgsz / classes       | 0.25 / 0.7 / 960 / [2, 5, 7]                    |
| device                             | 0 (GPU)                                         |

| So sánh                  |  HOTA |  DetA |  AssA |  LocA |  IDF1 |  MOTA |  MOTP | FP | FN | IDSW |
| ------------------------- | ----: | ----: | ----: | ----: | ----: | ----: | ----: | -: | -: | ---: |
| bạn vs gold              | 0.803 | 0.783 | 0.826 | 0.894 | 0.935 | 0.864 | 0.886 | 70 |  8 |    0 |
| ByteTrack control vs gold | 0.709 | 0.649 | 0.776 | 0.846 | 0.875 | 0.749 | 0.823 | 88 | 54 |    2 |
| BoT-SORT + ReID vs gold   | 0.763 | 0.711 | 0.820 | 0.872 | 0.900 | 0.792 | 0.860 | 91 | 26 |    2 |
| ReID vs bạn              | 0.760 | 0.705 | 0.820 | 0.908 | 0.872 | 0.748 | 0.900 | 80 | 77 |    3 |

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

	MOTA của tôi thấp hơn IDF1.

	Nếu MOTA cao mà IDF1 thấp thì mô hình phát hiện/tracking đối tượng tốt nhưng  giữ danh tính ID kém , dễ xảy ra đổi ID.

	MOTA không phạt nặng lỗi ID vì MOTA gộp cả ba lỗi FN( bỏ sót vật thể), FP ( thùa box), IDSW (ID Switch):
	**MOT**A=**1**−(FN+FP+IDSW)/GT

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**

**So sánh:**

* **IDF1 và AssA:** BoT-SORT + ReID treatment có **IDF1 cao hơn (0.900 so với 0.875)** và **AssA cao hơn (0.820 so với 0.776)** so với ByteTrack control. Điều này cho thấy ReID treatment cải thiện khả năng duy trì ID (IDF1) và chất lượng liên kết (AssA) giữa các đối tượng qua các frame một cách đáng kể. ReID giúp tracker đưa ra quyết định tốt hơn khi các đối tượng có thể bị che khuất hoặc tương tác phức tạp, dẫn đến các track nhất quán hơn.
* **IDSW (ID Switches):** Cả hai mô hình đều có  **IDSW là 2** . Điều này có nghĩa là, trong tổng số các track, cả ByteTrack và BoT-SORT + ReID đều mắc 2 lỗi chuyển đổi ID, tức là chúng đã gán sai ID cho một đối tượng đã được theo dõi trước đó trong 2 trường hợp. Mặc dù ReID treatment có IDF1 và AssA tốt hơn, nó không hoàn toàn loại bỏ được lỗi ID switch trong trường hợp cụ thể này, có thể do các trường hợp occlusion khó hoặc sự tương đồng về ngoại hình giữa các đối tượng.

Ví dụ frame sequence để giải thích treatment tốt hơn:

* ByteTrack control: Trong phần chẩn đoán lỗi của ByteTrack, `track gold 4` được liệt kê trong mục "TÁCH TRACK" với thông báo: `track gold 4 (95 frame) bị chia cho ID [15, 14]`. Điều này có nghĩa là ByteTrack đã không duy trì được ID nhất quán cho đối tượng này trong suốt 95 frame mà đã gán cho nó hai ID khác nhau (14 và 15) tại một thời điểm nào đó (cụ thể là frame 59, theo chẩn đoán ID switch).
* BoT-SORT + ReID treatment: Trong phần chẩn đoán lỗi của ReID treatment, `track gold 4` không hề xuất hiện trong mục  "TÁCH TRACK" . Điều này cho thấy BoT-SORT + ReID đã thành công trong việc duy trì `track gold 4` như một track duy nhất với ID nhất quán trong toàn bộ 95 frame. Khả năng này có thể đến từ việc ReID sử dụng thông tin ngoại hình để giúp phân biệt đối tượng và giữ đúng ID của nó ngay cả khi motion và IoU không đủ để đưa ra quyết định chính xác.

3. DetA, FP và FN đổi thế nào**? Lỗi còn lại là detector hay association?**
4. DetA (Detection Accuracy): Đã tăng  từ 0.649 lên 0.711. DetA là một chỉ số kết hợp đánh giá cả chất lượng phát hiện và chất lượng liên kết. Sự gia tăng này cho thấy BoT-SORT + ReID treatment có khả năng khớp các bounding box được phát hiện với các đối tượng thực tế (gold) tốt hơn.
5. FP  tăng từ 88 lên 91. Điều này có nghĩa là ReID treatment đã tạo ra thêm một vài bounding box không tương ứng với bất kỳ đối tượng nào trong gold. Đây có thể là do model phát hiện nhầm các vật thể tĩnh, hoặc đôi khi duy trì một track quá lâu sau khi đối tượng đã rời khỏi khung hình hoặc bị che khuất hoàn toàn.
6. FN đã giảm rất đáng kể từ 54 xuống 26. Đây là một cải thiện lớn. Nó cho thấy BoT-SORT + ReID treatment đã bỏ sót ít đối tượng thực tế hơn so với ByteTrack control. Việc giảm FN là một đóng góp lớn vào sự cải thiện của DetA.

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

	Ở frame 79, theo kết quả phân tích (`only_b`: 2, `only_a`: 1), nhãn của tôi có 2 bounding box mà model ReID không có, trong khi model chỉ có 1 bounding box mà tôi không có. Điều này cho thấy model có thể đã bỏ sót việc phát hiện 2 vật thể mà tôi đã gán nhãn chính xác. Các vật thể bị bỏ sót thường là do bị che khuất hoặc có kích thước nhỏ, nằm ở rìa khung hình, khiến detector của model kém tự tin hơn.

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**

	Ở frame 107, theo kết quả phân tích bất đồng: có 2 bounding box chỉ có trong kết quả của model ReID (`only_a`: 2), và 1 bounding box chỉ có trong nhãn của tôi (`only_b`: 1). Điều này gợi ý rằng có thể bạn đã bỏ sót 2 vật thể mà model ReID phát hiện được. Các vật thể này có thể là xe nhỏ hoặc xe ở rìa khung hình, những vị trí dễ bị bỏ sót trong quá trình gán nhãn thủ công. Cần kiểm tra lại frame này để xác định xem model ReID có phát hiện đúng các vật thể này hay không, hay đây là các False Positive của model, model nhầm đèn của nhà hàng thành xe.

## 6. Nếu phải gán thêm 10 clip nữa

Bạn sẽ sửa gì trong `GUIDELINE_MINI.md`, và đổi gì trong quy trình làm việc của mình?

Tôi đã sửa `GUIDELINE_MINI.md`:

1. Họ và tên
2. Luật ID — phần quan trọng nhất
3. Luật bbox
4. Ít nhất ba ca mơ hồ đã gặp thật
5. Sửa gì sau khi chấm với gold và sau khi kiểm chéo

## 7. Tệp đã nộp

- [X] `annotations/clip_01/gt.txt`
- [X] `annotations/clip_02/gt.txt`
- [X] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [X] `GUIDELINE_MINI.md` đã điền
- [X] `outputs/eval_vs_gold.json`
- [X] `outputs/model_bytetrack_clip_01.txt`
- [X] `outputs/model_reid_clip_01.txt`
- [X] `outputs/model_run_config.json`
- [X] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [X] `reports/review_partner.md`
- [X] `reports/REPORT.md` (file này)
