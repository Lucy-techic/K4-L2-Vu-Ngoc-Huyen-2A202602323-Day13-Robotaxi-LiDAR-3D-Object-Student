# Báo cáo thực hành PointPillars — Day 13

## 1. Thông tin học viên và provenance

- Họ và tên: **Mai Lưu Ly**
- Mã số học viên: **2A202602157**
- Mã nhóm/phòng: **Thực hiện cá nhân**
- Trạng thái: `executed-by-individual`
- Người trực tiếp chạy: Mai Lưu Ly
- Thời gian chạy: **02/10/2026, khoảng 09:16–09:18 (UTC+7)**
- Hệ máy: Windows, CPU Intel Core i5-1240P, kiến trúc AMD64
- Docker runtime: Linux/amd64, Docker Server 29.7.2
- Giới hạn container: 4 CPU, 4 GB RAM
- Image tag: `day13-pointpillars:lc-20261001-amd64`
- Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
- Phiên bản repo trong bundle: `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD/frame: `input/demo.pcd`, frame `demo`, KITTI 000008 đã chuyển đổi
- SHA-256 input: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`
- SHA-256 checkpoint: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: `front-window`
- Score threshold: `0.3`
- `z_ground` ước lượng: `0.075 m`
- Giả định kênh thứ tư: reflectance thật đã bị bỏ; pipeline dùng kênh hằng theo adapter, RGB=0 chỉ là placeholder và không phải intensity LiDAR được khôi phục.
- Kết quả smoke test: `passed`

## 2. Ba lượt inference thật

A/B/C được chạy tuần tự trên cùng PCD bằng cùng checkpoint. Số hộp và `mean_z` dưới đây được lấy trực tiếp từ `summary.csv` của từng lượt.

| Lượt | Delta (m) | Pillar XY (m) | Số hộp | mean_z (m) | File bằng chứng | Quan sát |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/summary.csv`, `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png` | Chỉ có 1 hộp `vehicles`, ở vùng x khoảng 12–15 m trên ảnh Side. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/summary.csv`, `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png` | Có 10 `vehicles`, 1 `two-wheels` và 2 `pedestrian`; hộp xuất hiện từ vùng gần x khoảng 3 m đến vùng xa x khoảng 56 m. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/summary.csv`, `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png` | Có 6 hộp và tất cả mang nhãn `pedestrian`; nhiều hộp `vehicles` quan sát ở B không còn xuất hiện. |

### So sánh A/B — chỉ đổi delta

A có 1 hộp, trong khi B có 13 hộp. Ảnh Side của B cho thấy nhiều hộp mới tại các vùng x khoảng 3–25 m, 33–41 m và khoảng 56 m. Thành phần class cũng đổi từ chỉ một `vehicles` ở A thành 10 `vehicles`, 1 `two-wheels` và 2 `pedestrian` ở B.

Sự khác nhau không phải do lấy hộp cũ rồi cộng trực tiếp 1,73 m vào z. Pipeline đã biến đổi input trước inference và chạy lại model nên số lượng, class và vị trí prediction có thể cùng thay đổi. Vì không có ground truth trong thí nghiệm này, chưa đủ bằng chứng để kết luận B đúng hơn A chỉ vì B có nhiều hộp hơn.

### So sánh B/C — chỉ đổi pillar

B có 13 hộp còn C có 6 hộp. Khi Pillar XY tăng từ 0,16 m lên 0,32 m, các prediction thay đổi mạnh: nhiều hộp `vehicles` của B biến mất và cả 6 hộp trong C đều được gán `pedestrian`. Điều này cho thấy thay đổi cách gom điểm thành pillar đã ảnh hưởng đến biểu diễn đầu vào và đầu ra detector.

Chưa đủ bằng chứng để khẳng định B hoặc C tốt hơn. C vẫn sử dụng checkpoint pretrained ban đầu, không phải một model được train riêng cho pillar 0,32 m. Số hộp nhiều hơn, ít hơn hoặc `mean_z` thấp hơn không tự chứng minh chất lượng tốt hơn.

### Giới hạn khi đọc kết quả

- Phạm vi inference là front-window; vật thể ngoài ROI không thể được dùng để kết luận model bỏ sót.
- Ảnh Side là hình chiếu x-z nên các vật thể khác y có thể chồng lên nhau.
- Ảnh Side không đủ để kết luận yaw, tâm hoặc kích thước 3D của từng hộp; cần xem thêm Top/Front/góc xoay và ảnh camera.
- Các JSON là prediction thí nghiệm, chưa đủ cơ sở để import vào CVAT Robotaxi và không phải ground truth.

## 3. Ca QC có kiểm soát — không import CVAT

Helper tạo ba ca từ prediction thật của lượt B. Với `delta=1.73 m` và `z_ground=0.075 m`, độ lệch có chủ đích là:

`delta + z_ground = 1.805 m`.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Quyết định | Bằng chứng |
| --- | ---: | ---: | --- | --- | --- |
| `case-correct` | 0/13 | 0 m | Không | Giữ làm bản đối chiếu chuyển đổi nguồn, nhưng không coi là nhãn đúng | `qc-cases/case-correct.json`, `qc-cases/side-correct.png` |
| `case-batch-z` | 13/13 | -1.805 m | Không | Dừng sửa tay cả batch; báo kiểm tra phép chuyển frame và tạo lại prediction bằng pipeline đúng | `qc-cases/case-batch-z.json`, `qc-cases/side-batch-z.png` |
| `case-one-box-z` | 1/13 | -1.805 m | Không | Kiểm riêng hộp đầu tiên qua nhiều view; chưa kết luận toàn pipeline sai | `qc-cases/case-one-box-z.json`, `qc-cases/side-one-box-z.png` |

Các ca trên là biến đổi huấn luyện có chủ đích từ lượt B, không phải ba lượt inference mới và không phải annotation reference.

## 4. Nhận xét cá nhân — Mai Lưu Ly

Tôi tự thực hiện toàn bộ quy trình cá nhân: chuẩn bị Docker/Python, kiểm SHA-256 của bundle, chạy runner A/B/C, kiểm `smoke.json`, đọc JSON/CSV/ảnh Side và phân tích ba ca QC.

Quan sát chính của tôi là thay đổi delta trước inference làm đầu ra đổi từ 1 hộp ở A thành 13 hộp ở B; vì model được chạy lại trên input khác, đây không phải phép dịch đồng loạt các hộp sau inference. Khi giữ delta và tăng pillar từ 0,16 m lên 0,32 m, số hộp giảm từ 13 xuống 6 và thành phần class thay đổi hoàn toàn, cho thấy detector nhạy với biểu diễn pillar trong thí nghiệm này.

Phép đổi z của pipeline được hiểu như sau:

```text
z_model  = z_source - z_ground - delta
z_source = z_model  + z_ground + delta
```

Nếu quên phép đổi ngược, toàn bộ hộp có thể thấp hơn 1,805 m. Khi cả 13 hộp cùng lệch đúng lượng này, tôi sẽ dừng batch và yêu cầu kiểm pipeline thay vì sửa từng hộp. Khi chỉ một hộp lệch, tôi sẽ kiểm riêng đối tượng qua nhiều góc nhìn. Điều tôi chưa thể kết luận từ bài này là cấu hình nào chính xác hơn, vì không có ground truth và ảnh Side không đủ để duyệt đầy đủ hình học 3D.

## 5. Tự kiểm trước khi nộp

- [x] Runner chạy thật trên máy cá nhân.
- [x] `smoke.json` có trạng thái `passed`.
- [x] Có đủ JSON, Side PNG và CSV của A/B/C.
- [x] Có đủ ba ca QC và manifest chỉ rõ `training_only`.
- [x] Báo cáo phân biệt inference A/B/C với các ca QC được biến đổi từ B.
- [x] Không import prediction KITTI hoặc `case-*.json` vào CVAT Robotaxi.
- [x] Có nhận xét cá nhân và quyết định xử lý lỗi batch/từng hộp.
- [x] Điền mã phòng nếu LC yêu cầu.
- [x] Nộp thư mục này vào nơi thu private do LC chỉ định.

## 6. LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Xác nhận chạy thật/cần bổ sung:
- Output đủ và không đưa ca lỗi vào CVAT:
- Nhận xét và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC hoặc cần bổ sung:

