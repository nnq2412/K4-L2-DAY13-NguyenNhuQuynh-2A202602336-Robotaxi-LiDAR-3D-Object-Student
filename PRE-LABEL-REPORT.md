# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: SOLO
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Như Quỳnh (@nnq2412); 2026-10-03; Windows x86_64 (amd64) / Docker Desktop
- Image tag và image ID; phiên bản repo: day13-pointpillars:lc-20261001-amd64; sha256:e7b6032b36dfc01b51da2fb29d752942bb51f7e9f5c3fc5d0e73276a7c8e9931; revision 0831856d921609312d42c7582c366e5a311bb7b1
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: demo.pcd / demo; máy trạm cá nhân; sha256: 3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: /opt/PointPillars/pretrained/epoch_160.pth (sha256: 482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1)
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ tư là constant placeholder (RGB=0, reflectance nguồn đã lược bỏ để dùng adapter kênh hằng); z_ground = 0.075 m (ước lượng từ độ cao các điểm mặt đất trong scan demo).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-...json`, `run-A/side-...png`, `run-A/summary.csv` | Chỉ detect được 1 xe duy nhất tại cự ly gần (x≈12.3m, y≈-1.6m, z≈0.33m). Do delta=0, toàn bộ điểm trong model frame bị dịch cao hơn phân bố chuẩn của checkpoint KITTI, làm trượt khỏi ROI hoặc ngưỡng kích hoạt của các anchor boxes. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-...json`, `run-B/side-...png`, `run-B/summary.csv` | Baseline chuẩn với delta=1.73m. Model detect được 13 hộp: 10 vehicles, 2 pedestrian, 1 two-wheels bám sát các cụm điểm thực tế trên mặt đường (z bám quanh dải 0.8m - 1.5m). |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-...json`, `run-C/side-...png`, `run-C/summary.csv` | Kích thước ô pillar tăng gấp đôi (0.32m). Số hộp giảm xuống 6 hộp và toàn bộ đều bị gán nhãn pedestrian. Mất hoàn toàn các hộp vehicles và two-wheels do độ phân giải pillar thô làm nhòe đặc trưng hình học footprint của xe. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh/file/vùng `side-*.png` khác ở mật độ hộp dọc trục x từ 10m đến 45m (A chỉ có 1 hộp đơn lẻ ở x≈12m, B có 13 hộp trải đều). Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là liệu một số cụm điểm ở xa (x>40m) trong B là xe thật hay false positive do chùm tia phản xạ thưa.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh/file/vùng `side-*.png` và `boxes-*.json` khác ở phân loại lớp và kích thước hộp (B nhận diện được 10 vehicles, 1 two-wheels, 2 pedestrian; C chỉ còn 6 pedestrian với footprint nhỏ hẹp). Số lượng/lớp/vị trí thay đổi như sau: mất hoàn toàn các hộp xe lớn, các cụm điểm xe bị gộp hoặc model phân loại nhầm thành pedestrian. Có đủ bằng chứng để kết luận tốt hơn không? Không, C dùng lại checkpoint được train trên voxel 0.16m, việc đổi sang voxel 0.32m làm suy giảm chất lượng biểu diễn không gian, không thể kết luận C tốt hơn.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Giới hạn front-window cắt bỏ các đối tượng phía sau và hai bên ngoài biên quét, các đối tượng nằm ngoài ROI không thể coi là lỗi model bỏ sót. Góc Side là hình chiếu phẳng 2D (x-z), các xe có cùng x nhưng khác y sẽ bị chiếu chồng lên nhau, và góc Side hoàn toàn không thể hiện được góc xoay yaw (hướng đầu xe) trên mặt phẳng x-y.
- JSON nào còn chưa đủ cơ sở để import? Cả 3 file JSON A/B/C đều là pre-label từ mô hình KITTI thử nghiệm trên 1 PCD demo, không được import vào CVAT Robotaxi vì khác dataset, khác hệ sensor và chưa qua rà soát đa góc nhìn kết hợp ảnh camera.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm tra từng hộp (nếu cần tinh chỉnh) | Giữ nguyên 100% tọa độ và kích thước từ prediction B, các hộp nằm đúng cao độ mặt đường. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi class/x/y/yaw, chỉ z bị trừ 1.805m | DỪNG BATCH, báo LC kiểm tra pipeline | Tất cả 13 hộp trong batch đều bị chìm sâu xuống lòng đất đúng một lượng delta + z_ground = 1.73 + 0.075 = 1.805m. Đây là lỗi hệ thống do pipeline quên thực hiện phép biến đổi ngược z_source = z_model + delta + z_ground. |
| case-one-box-z | 1 / 13 | -1.805 m | Chỉ hộp đầu tiên đổi z, 12 hộp giữ nguyên | KIỂM TRA TỪNG HỘP | Chỉ có duy nhất hộp index 0 bị chìm xuống z=-0.75m, 12 hộp còn lại vẫn nằm trên mặt đất. Đây là lỗi cục bộ, không phải lỗi pipeline, cần mở CVAT để chỉnh tay riêng hộp đó. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

**Nguyễn Như Quỳnh (@nnq2412 / 2A202602336):**
- **Vai trò thực hiện**: Phân tích kết quả được cấp (do máy bị lỗi API Docker không tự chạy được lệnh), đọc file JSON, quan sát hình học các ảnh chiếu Side và phân tích các trường hợp QC.
- **Quan sát A/B/C**: Trong `run-A`, việc đặt delta=0 làm model chỉ nhận diện được 1 hộp ở cự ly gần. Khi đổi sang delta=1.73m trong `run-B`, số hộp tăng vọt lên 13 hộp bám theo mặt đường. Việc chuyển sang pillar lớn 0.32m ở `run-C` làm mất hoàn toàn hình khối ô tô, chỉ còn sót lại vài hộp gán nhãn pedestrian.
- **Diễn giải phép z thuận/ngược**: Pipeline đưa điểm từ hệ nguồn vào model qua phép thuận $z_{model} = z_{source} - z_{ground} - \delta$. Do đó khi model xuất tọa độ hộp, pipeline bắt buộc phải cộng ngược lại $z_{source} = z_{model} + z_{ground} + \delta$ để đưa hộp về đúng vị trí thực.
- **Quyết định lỗi batch và hành động**: Khi phát hiện lỗi trong `case-batch-z` (tất cả 13 hộp cùng bị chìm 1.805m), hành động duy nhất đúng là **dừng sửa tay** và báo đội ngũ làm pipeline kiểm tra lại code; tuyệt đối không ngồi dịch z thủ công từng hộp. Nếu chỉ có 1 hộp bị lệch như `case-one-box-z`, thì mới dùng CVAT để chỉnh tay riêng hộp đó.
- **Điều em chưa chắc**: Các điểm phản xạ ở cự ly xa ngoài 40m trong kết quả lượt B khá thưa thớt, cần đối chiếu thêm với ảnh camera ở bài nguồn Robotaxi thật mới có thể khẳng định chắc chắn đó là xe thật hay tín hiệu nhiễu.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
