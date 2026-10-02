# Báo cáo thực hành PointPillars — Day 13

> Bản private, nộp vào nơi thu của LC. KHÔNG commit lên GitHub public.

## Nhóm và provenance

- Mã nhóm/phòng: làm cá nhân (1 thành viên)
- Thành viên: Vu Ngoc Huyen
- Trạng thái: `executed-by-group` (tự chạy trên máy cá nhân, làm một mình)
- Người thực sự chạy: Vũ Ngọc Huyền (2A202602323); ngày/giờ: 02/10/2026, 21:35:40–21:36:55 (UTC+7) — theo `smoke.json` (14:35:40–14:36:55 UTC): docker-load 46,0 s, run-A 11,1 s, run-B 9,2 s, run-C 5,6 s, qc-cases 2,8 s; mọi bước `passed`
- Hệ máy/architecture: Windows, Docker Desktop (Linux containers), `docker info` báo `linux x86_64`; runtime trong `smoke.json`: linux/amd64; Python 3.12.10 trên host; gói `student-prelabel-amd64.zip`; giới hạn container 4 CPU / 4 GB
- Image: tag nguồn `day13-pointpillars:lc-20261001-amd64`; image ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82` (linux/amd64)
- Phiên bản repo: release `student-prelabel-v1` (source commit `f5f1de0`); manifest ghi `repo_revision` `0831856d921609312d42c7582c366e5a311bb7b1`, `working_tree_dirty: true` (như release đã công bố)
- PCD: `input/demo.pcd` trong gói Student — KITTI 000008 đã chuyển đổi (CC BY-NC-SA 3.0), 17.238 điểm, `frame=demo`; `input_sha256` `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`; nơi chạy: máy cá nhân, container không dùng mạng
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Code: `preannotate_sha256` `65edf6ac…eb5ca`, `helper_sha256` `c177fc00…eaa7`
- Hash prediction: A `02b1ba0a…0880`, B `c2a8db24…cc80`, C `8eb5011a…d6ec`
- Phạm vi: front-window của checkpoint; score threshold: 0.3; dataset `KITTI`
- Giả định kênh thứ tư/intensity và nguồn z_ground: PCD đã bỏ reflectance thật, adapter dùng kênh hằng số (RGB=0 chỉ là placeholder, không phải intensity). `z_ground = 0.075` m, do script ước lượng từ chính PCD (giống nhau ở cả ba lượt).

## Ba lượt inference thật

Số liệu lấy từ log runner và `run-A/B/C/summary.csv`.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/` boxes-*.json, side-*.png, summary.csv | Chỉ 1 hộp `vehicles`; mean_z thấp nhất (0.330 m) |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/` boxes-*.json, side-*.png, summary.csv | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; nhiều hộp nhất |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/` boxes-*.json, side-*.png, summary.csv | Cả 6 hộp đều là `pedestrian`, không còn `vehicles` |

- **A/B — chỉ đổi delta:** A có 1 hộp, B có 13 hộp. Hai lượt cùng `z_ground = 0.075`, cùng pillar 0.16 m, chỉ khác delta (0 → 1.73 m). Khi delta = 0, sau bước trừ mặt đất các điểm nằm quanh z ≈ 0, trong khi checkpoint KITTI được train với cảm biến cao khoảng 1.73 m trên mặt đường (mặt đường ở z ≈ −1.73 trong hệ model). Input lệch khỏi phân bố lúc train nên model gần như không phát hiện được gì. Đây là model chạy lại trên input khác, không phải hộp cũ bị dịch: số hộp đổi từ 1 lên 13 và mean_z không chênh đúng 1.73 m (0.330 → 1.034). Điều em còn chưa chắc: không có nhãn đối chiếu nên chưa khẳng định cả 13 hộp của B đều đúng; B chỉ là mốc so sánh hợp lý hơn A vì khớp giả định chiều cao cảm biến của checkpoint.
- **B/C — chỉ đổi pillar:** B có 13 hộp, C có 6 hộp. Giữ delta = 1.73, chỉ tăng pillar từ 0.16 lên 0.32 m. Phân bố lớp đổi hẳn: B có 10 `vehicles`, còn C không còn `vehicles` nào mà cả 6 hộp là `pedestrian`; mean_z gần như giữ nguyên (1.034 → 1.091). Checkpoint được train với pillar 0.16 m nên khi gom điểm theo ô 0.32 m, lưới pseudo-image và mật độ đặc trưng khác hẳn lúc train, khiến model nhận sai lớp và bỏ sót xe. Không đủ bằng chứng để nói C tốt hơn; ngược lại, việc mọi hộp chuyển sang `pedestrian` trên một cảnh đường phố là dấu hiệu biểu diễn đầu vào không khớp checkpoint. Ít hộp hơn hoặc lớp khác không tự nói lên đúng/sai khi chưa có nhãn.
- **Giới hạn ROI và góc Side:** chỉ chạy cửa sổ phía trước của checkpoint, nên vật nằm ngoài ROI không được tính là model bỏ sót. Ảnh Side là hình chiếu x-z của cả scene; các xe ở y khác nhau có thể chồng lên nhau, nên Side đủ để thấy độ cao/đáy hộp so với điểm nhưng không đủ để kết luận yaw, chiều rộng hay tâm y của từng hộp. Muốn duyệt hình học từng hộp cần thêm góc Top/Front và ảnh camera.
- **JSON nào chưa đủ cơ sở để import?** Không JSON nào trong bài được import: đây là KITTI demo, khác frame Robotaxi. Nếu là pipeline thật: A không dùng được (sai giả định chiều cao cảm biến); C không dùng được (pillar không khớp checkpoint, lớp bị lệch hết sang `pedestrian`). B là ứng viên hợp lý nhất nhưng vẫn cần kiểm từng hộp bằng Top/Side/Front và camera, kiểm thêm phạm vi ROI và hộp thiếu ngoài cửa sổ trước khi dùng làm pre-label.

## Ca QC có kiểm soát — không import CVAT

Ba ca do helper tạo từ prediction thật của lượt B (13 hộp), ghi tại `qc-cases/`. Lượng lệch theo mô tả helper là `z_ground + delta = 0.075 + 1.73 = 1.805` m.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Quyết định | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không | Không thấy lỗi chuyển frame; vẫn kiểm từng hộp như bình thường | `qc-cases/case-correct.json`, `side-correct.png` |
| case-batch-z | 13 / 13 | −1.805 m (mọi hộp chìm xuống cùng một lượng) | Không — chỉ z đổi | **Dừng batch**: không sửa tay; báo LC kiểm phép chuyển frame (thiếu bước cộng ngược `z_ground + delta`) và yêu cầu tạo lại prediction | `qc-cases/case-batch-z.json`, `side-batch-z.png` |
| case-one-box-z | 1 / 13 | −1.805 m ở một hộp | Không | **Kiểm từng hộp**: 12 hộp còn lại đúng chỗ nên không phải lỗi pipeline; kiểm hộp lệch qua Top/Side/Front và camera rồi sửa đối tượng đó | `qc-cases/case-one-box-z.json`, `side-one-box-z.png` |

Các ca này là biến đổi có chủ đích (training-only) từ prediction lượt B, không phải lượt inference riêng, không phải nhãn đúng và không được import vào CVAT.

## Nhận xét cá nhân — Vũ Ngọc Huyền (2A202602323)

- **Vai trò:** làm cá nhân nên đảm nhận cả bốn vai ở cả ba lượt: chạy lệnh, kiểm cấu hình/JSON, xem hình học, ghi log.
- **Quan sát có dẫn chứng:** Từ log runner và `summary.csv`, cùng một PCD và checkpoint, A (delta 0) chỉ ra 1 hộp, B (delta 1.73) ra 13 hộp gồm 10 xe, còn C (pillar 0.32) ra 6 hộp toàn `pedestrian`. Chỉ một tham số đầu vào thay đổi cũng làm thay đổi mạnh cả số lượng lẫn lớp, nên pre-label phải được kiểm cấu hình trước khi tin.
- **Phép z thuận/ngược:** trước inference `z_model = z_source − z_ground − delta`; khi xuất hộp `z_source = z_model + z_ground + delta`. Đổi delta trước inference làm thay đổi input nên model có thể ra số hộp khác hẳn (A 1 hộp → B 13 hộp). Còn nếu quên phép ngược sau inference thì số hộp giữ nguyên nhưng cả batch lệch cùng 1.805 m, như ca `case-batch-z`.
- **Quyết định lỗi batch:** khi mọi hộp lệch cùng một lượng z, em dừng sửa tay và báo LC kiểm pipeline, vì sửa 13 hộp bằng tay vừa tốn công vừa che mất lỗi hệ thống. Khi chỉ một hộp lệch, em kiểm riêng hộp đó qua nhiều góc nhìn và camera.
- **Điều chưa chắc:** không có nhãn KITTI đối chiếu nên chưa biết trong 13 hộp của B có bao nhiêu hộp đúng; PCD không có intensity thật nên kết quả có thể khác KITTI gốc.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
'@ | Set-Content -Path C:\Lab13\K4-DAY13-VuNgocHuyen\PRE-LABEL-REPORT.md -Encoding UTF8