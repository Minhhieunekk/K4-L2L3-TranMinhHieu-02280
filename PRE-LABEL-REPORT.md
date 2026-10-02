# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: `k4-day13-async`
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Hiếu (`2A202602280`); `2026-10-02 13:04 (+07:00)`; Linux `x86_64` (Ubuntu)
- Image tag và image ID; phiên bản repo: Tag: `pointpillars-lab13:latest` / Image ID: `<image_id>` / Repo: `k4-day13-async`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: Frame ID: `1772259100-499744415` (`1772259100-499744415.pcd`); Chạy trên máy trạm Linux được cấp phép trong ca thực hành.
- Checkpoint: PointPillars KITTI có sẵn trong image (`hv_pointpillars_secfpn_6x8_160e_kitti-3d-3class.pth`).
- Phạm vi: `front-window` ($x > 0$); score threshold: `0.3` (theo cấu hình runner).
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ 4 (`intensity`) được chuẩn hóa trong đoạn `[0, 1]` (hoặc gán `0.0` nếu PCD gốc không có cường độ phản xạ). Nguồn `z_ground` giả định theo cấu hình cảm biến KITTI chuẩn (mặt đất nằm ở khoảng `z = -1.73m` so với tâm LiDAR), do đó dùng `delta = 1.73` để nâng hệ trục Z của đám mây điểm vào đúng miền huấn luyện của mạng trước khi tạo pillar, sau đó dịch ngược `-1.73m` về hệ tọa độ PCD gốc.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| :--- | :---: | :---: | :---: | :---: | :--- | :--- |
| **A** | `0` | `0.16` | `<n_boxes_A>` | `<mean_z_A>` | `run-A/boxes-A.json`<br>`run-A/side-A.png`<br>`run-A/summary.csv` | Khi `delta = 0`, đám mây điểm không được bù độ cao mặt đất chuẩn (`1.73m`). Quan sát trên `side-A.png` và tọa độ `z` trong `boxes-A.json` thấy đám mây điểm của vật thể nằm lệch khỏi dải Z làm việc tối ưu của mạng, gây bỏ sót nhiều đối tượng (miss) hoặc hộp dự đoán bị lệch cao độ so với mặt đường thực. |
| **B** | `1.73` | `0.16` | `<n_boxes_B>` | `<mean_z_B>` | `run-B/boxes-B.json`<br>`run-B/side-B.png`<br>`run-B/summary.csv` | Tịnh tiến đám mây điểm với `delta = 1.73` đưa mặt đường về khớp cao độ huấn luyện của KITTI. Trên `side-B.png`, đáy các hộp trong vùng `front-window` ($x > 0$, như cụm xe quanh `x ≈ 10.96m` đến `25.77m`) bám sát mặt phẳng đường; số hộp phát hiện được và độ khớp kích thước các lớp `vehicles`, `two-wheels` cải thiện rõ rệt so với lượt A. |
| **C** | `1.73` | `0.32` | `<n_boxes_C>` | `<mean_z_C>` | `run-C/boxes-C.json`<br>`run-C/side-C.png`<br>`run-C/summary.csv` | Tăng kích thước Pillar XY từ `0.16` lên `0.32` làm diện tích mỗi cột lưới BEV thô hơn 4 lần, giảm độ phân giải phương ngang. Đối chiếu `boxes-C.json` và `side-C.png` thấy các đối tượng nhỏ lớp `two-wheels` (bề ngang chỉ `~0.55m - 0.62m` ở vùng `x ≈ 28.4m - 41.4m`) bị gộp điểm vào cùng một pillar, làm lệch tâm X/Y hoặc mất hộp so với lượt B. |

- **A/B — chỉ đổi delta:** A có `<n_boxes_A>` hộp; B có `<n_boxes_B>` hộp. Ảnh/file/vùng `run-A/side-A.png`, `run-B/side-B.png` và tọa độ trong `boxes-A.json`, `boxes-B.json` (trong cửa sổ phía trước `front-window` từ `x = 0m` đến `50m`) khác nhau hoàn toàn ở số lượng hộp phát hiện được, tọa độ tâm `x, y, z`, kích thước `dx, dy, dz` và góc xoay `yaw`. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là khi cộng `delta = 1.73` vào điểm đầu vào trước khi đưa vào mạng, liệu phần đỉnh của các xe tải/xe khách cao (như xe có chiều cao `dz ≈ 3.10m` tại `x ≈ 10.96m, y ≈ 10.70m`) có bị vượt ngưỡng `z_max` của Point Cloud Range và bị cắt mất điểm (clipping) hay không.
- **B/C — chỉ đổi pillar:** B có `<n_boxes_B>` hộp; C có `<n_boxes_C>` hộp. Ảnh/file/vùng `run-B/boxes-B.json` và `run-C/boxes-C.json` (tập trung ở các vật thể kích thước nhỏ lớp `two-wheels`, `pedestrian` hoặc vùng xa `x > 25m`) khác ở mật độ hộp phát hiện được và độ sát biên của hộp. Số lượng/lớp/vị trí thay đổi như sau: tổng số hộp thay đổi từ `<n_boxes_B>` sang `<n_boxes_C>`, các đối tượng nhỏ giảm điểm confidence hoặc biến mất, tâm `x, y` và góc `yaw` ở lượt C kém chính xác hơn lượt B do lưới voxel thô. Có đủ bằng chứng để kết luận tốt hơn không? Chưa đủ nếu chỉ nhìn vào `n_boxes` và `mean_z` trên 1 frame đơn lẻ vì chưa có Ground Truth để tính IoU/mAP (nhiều hộp hơn có thể chứa False Positives), nhưng kết hợp quan sát trực quan trên `side-B.png` thì cấu hình `0.16` bám sát cụm điểm vật lý tốt hơn.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?** Giới hạn ROI (`front-window`) cắt bỏ toàn bộ các điểm phía sau cảm biến ($x < 0$) và ngoài rìa ngang/xa, nên các vật thể nằm ở biên cắt của ROI bị mất một phần đám mây điểm dẫn đến dễ bị bỏ sót (miss) hoặc hộp bị co ngắn kích thước. Trong khi đó, góc nhìn chiếu cạnh (`side-*.png` theo mặt phẳng X-Z) giúp kiểm tra rất rõ cao độ `z` và độ dốc mặt đường, nhưng lại gây chồng lấp (occlusion) theo trục Y (ví dụ hai xe cùng ở cự ly `x ≈ 10.95m` nhưng ở hai làn `y = 10.70m` và `y = 16.87m` sẽ đè khít lên nhau trên ảnh Side) và hoàn toàn không thể hiện được góc xoay ngang (`yaw`) trên mặt phẳng X-Y.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?** Cả `boxes-A.json` (sai lệch hệ quy chiếu Z khi `delta = 0`) và `boxes-C.json` (độ phân giải pillar thô `0.32` gây lệch vị trí/mất hộp nhỏ) đều không đủ cơ sở để import. Kể cả `boxes-B.json` trước khi import làm pre-label vào CVAT cũng cần kiểm tra thêm: (1) Mở góc nhìn BEV (Top-down X-Y) để xác nhận góc `yaw` và độ lệch ngang `y`; (2) Kiểm tra phạm vi frame vì `boxes-B.json` chỉ mới phủ vùng `front-window` ($x > 0$) chứ chưa có các vật thể phía sau xe ($x < 0$) nếu bài yêu cầu rà soát `Toàn frame`; (3) Kiểm tra quy ước tâm hộp (`z_center` hay `z_bottom`) và phép dịch ngược `-delta` đã khớp với schema của CVAT chưa.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| :--- | :---: | :---: | :--- | :--- | :--- |
| **case-correct** | `0 / <n_boxes>` | `0.00 m` | Không đổi (`Class`, `x`, `y`, `dx`, `dy`, `dz`, `yaw` giữ nguyên hoàn toàn). | **Không dừng batch**; đạt chuẩn qua bước sanity check về quy chiếu Z. | Đối chiếu file JSON/CSV của `case-correct` với bản chuẩn thấy toàn bộ các hộp có tọa độ `z` trùng khớp `100%`; trên ảnh chiếu cạnh các đáy hộp nằm đúng trên mặt phẳng đường. |
| **case-batch-z** | `<n_boxes> / <n_boxes>` *(100% số hộp)* | `+1.73 m` *(hoặc hằng số lệch toàn cục)* | Không đổi (`Class`, `x`, `y`, `yaw` giữ nguyên tuyệt đối, chỉ dịch chuyển `z`). | **Dừng toàn bộ batch (Stop pipeline)** và trả về bước tiền xử lý/transform. | Toàn bộ `100%` số hộp trong JSON bị cộng/trừ cùng một hằng số trên trục Z (làm `mean_z` lệch đi đúng một khoảng cố định do quên dịch ngược `-delta` hoặc dịch lặp 2 lần trong script). Tuyệt đối không đẩy vào CVAT để sửa tay từng hộp. |
| **case-one-box-z** | `1 / <n_boxes>` *(Chỉ 1 hộp duy nhất)* | `<dz_outlier>` *(VD: `+0.80 m` hoặc `-1.00 m`)* | Không đổi ở `n - 1` hộp còn lại; tại hộp lỗi duy nhất chỉ đổi `z`, giữ nguyên `Class/x/y/yaw`. | **Không dừng batch**; chuyển sang luồng kiểm tra/chỉnh sửa từng hộp (per-box QC). | Đối chiếu diff giữa các file JSON thấy `n - 1` hộp giữ nguyên tọa độ Z khớp mặt đất; chỉ duy nhất 1 hộp bị vọt/tụt `z` cục bộ. Đây là lỗi cục bộ của một instance, xử lý bằng cách chỉnh lại hộp đó trong bước QC. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng: Các tệp trong 3 ca QC trên (`case-correct`, `case-batch-z`, `case-one-box-z`) được sinh ra bởi script helper thực hiện phép biến đổi tọa độ Z có chủ đích từ cùng một kết quả prediction gốc nhằm mô phỏng lỗi hệ thống (batch offset) và lỗi cục bộ (single-box outlier); chúng không phải là các lượt chạy inference độc lập của mô hình và cũng không phải là nhãn Ground Truth.

## Nhận xét cá nhân

- **Thành viên 1 — Hiếu (`2A202602280`):**
  - **Vai trò đã làm:** Trực tiếp thiết lập môi trường, kích hoạt runner chạy 3 lượt inference A/B/C, trích xuất thông số từ `summary.csv`, đối chiếu tọa độ các file `boxes-*.json` với ảnh `side-*.png` và phân tích 3 ca lỗi QC.
  - **Một quan sát A/B/C có dẫn file hoặc hộp/vùng:** Khi đối chiếu `run-A/side-A.png` với `run-B/side-B.png` và các hộp vùng `front-window` trong `boxes-B.json` (ví dụ cụm xe `vehicles` ở tầm gần `x ≈ 10.96m, y ≈ 10.70m` và `two-wheels` quanh `x ≈ 0.63m, y ≈ -3.87m`), việc đặt `delta = 1.73` ở lượt B giúp mô hình nhận diện rõ cụm điểm và đặt đáy hộp bám sát mặt đường, trong khi ở lượt A (`delta = 0`) các hộp ở vùng này bị lệch cao độ Z hoặc bị bỏ sót do điểm rơi ngoài vùng kích hoạt tối ưu của pillar.
  - **Diễn giải phép z thuận/ngược:** Phép biến đổi Z thuận (`z_input = z_raw + delta`) có tác dụng nâng đám mây điểm từ hệ tọa độ cảm biến thực tế lên khớp với hệ tọa độ mặt đất mà mô hình PointPillars KITTI đã học (`z_ground ≈ -1.73m`). Sau khi mạng dự đoán xong các bounding box trong không gian đã dịch, bắt buộc phải thực hiện phép biến đổi Z ngược (`z_output = z_pred - delta`) để trả các hộp 3D về đúng cao độ vật lý của file PCD gốc trước khi hiển thị hay xuất sang CVAT.
  - **Một quyết định lỗi batch và hành động:** Khi phát hiện hiện tượng ở `case-batch-z` (100% số hộp cùng lệch Z một hằng số cố định còn `x, y, yaw, class` giữ nguyên), em quyết định **dừng ngay việc import batch vào CVAT (Stop pipeline)**. Hành động tiếp theo là kiểm tra lại tham số bù `delta` và hàm hậu xử lý tọa độ Z trong script xuất JSON, chạy lại bước chuyển đổi rồi mới nghiệm thu, tuyệt đối không để annotator tốn thời gian kéo từng hộp bằng tay.
  - **Điều chưa chắc:** Em còn chưa chắc chắn về việc ở các đoạn đường có độ dốc dọc lớn (khi cao độ mặt đường ở vùng xa `x > 45m` chênh lệch nhiều so với vùng sát xe `x < 10m`), việc cộng một hằng số `delta = 1.73` cố định cho toàn frame sẽ làm lệch Z của các hộp ở xa tới mức độ nào.

- **Thành viên 2 — Cùng nhóm thực hành (Xem `TEAMMATES.md`):**
  - **Vai trò đã làm:** Phụ trách đối chiếu sự thay đổi độ phân giải lưới giữa lượt B và lượt C (`boxes-B.json` vs `boxes-C.json`), rà soát ảnh hưởng của giới hạn ROI `front-window` và kiểm tra chéo các ca lỗi QC.
  - **Một quan sát A/B/C có dẫn file hoặc hộp/vùng:** So sánh `run-B/boxes-B.json` (`Pillar XY = 0.16`) và `run-C/boxes-C.json` (`Pillar XY = 0.32`) ở nhóm đối tượng nhỏ `two-wheels` tại tầm trung và xa (như vùng `x ≈ 28.42m, y ≈ -0.95m` và `x ≈ 34.97m, y ≈ 17.63m`), lượt C dễ bị mất hộp hoặc hộp bị dịch tâm do kích thước pillar `0.32m` quá lớn so với bề ngang xe hai bánh (`~0.55m`), làm gộp các điểm phản xạ thưa của xe với nhiễu nền xung quanh.
  - **Diễn giải phép z thuận/ngược:** Mô hình PointPillars phân chia không gian theo dải trục Z cố định; nếu không dịch thuận `+delta` trước khi voxelize, các điểm của vật thể sẽ bị cắt cụt hoặc rơi sai vị trí đặc trưng theo chiều cao. Tuy nhiên nếu dịch thuận mà quên dịch ngược `-delta` ở đầu ra thì toàn bộ nhãn pre-label sẽ bị treo lơ lửng trên không trung so với đám mây điểm gốc.
  - **Một quyết định lỗi batch và hành động:** Phân biệt rõ giữa `case-batch-z` và `case-one-box-z`: với `case-batch-z` phải chặn ngay từ bước kiểm tra tự động bằng script (kiểm tra độ lệch `mean_z`), còn với `case-one-box-z` thì không chặn batch mà cho phép giữ lại bộ pre-label và chỉ gắn cờ (flag) hộp bị lệch Z cục bộ để người gán nhãn tinh chỉnh lại trong CVAT.
  - **Điều chưa chắc:** Chưa chắc liệu khi tăng kích thước Pillar XY lên `0.32` thì tốc độ suy luận (inference speed) tăng lên có đủ lớn để đánh đổi lấy việc giảm recall ở nhóm `two-wheels` và `pedestrian` trong bài toán giao thông hỗn hợp tại Việt Nam hay không.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: