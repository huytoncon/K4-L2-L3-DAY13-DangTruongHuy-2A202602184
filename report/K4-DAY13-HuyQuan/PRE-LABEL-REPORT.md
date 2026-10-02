# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm HuyQuan / Phòng: <!-- TODO -->
- Thành viên: xem `TEAMMATES.md` — Đặng Trường Huy (2A202602184), Tô Văn Anh Quân (2A202602231); nhóm 2 người.
- Trạng thái: `executed-by-group` — chạy thật trên laptop của Đặng Trường Huy, không phải `provided-results`. Lệnh Docker/runner được thực hiện qua Claude Code (AI assistant) theo yêu cầu của Huy, không gõ tay từng lệnh; nếu LC yêu cầu tự tay vận hành, nhóm báo rõ điều này.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Đặng Trường Huy (hỗ trợ bởi Claude Code); 2026-10-01 08:00:06–08:00:24 UTC (15:00 giờ VN); Windows 11 + Docker Desktop (Linux containers), linux/amd64, giới hạn container 4 CPU / 4 GB.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` từ gói Release `student-prelabel-v1` (amd64, SHA256 ZIP khớp `SHA256SUMS.txt`); image ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; `repo_revision` `0831856d921609312d42c7582c366e5a311bb7b1`, `smoke.json` ghi `working_tree_dirty: true` (giá trị do gói ghi lúc đóng gói, nhóm không sửa code).
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` trong gói Student (KITTI 000008 đã chuyển đổi, 17 238 điểm), frame_id=`demo`; chạy trên máy nhóm; input_sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth` có sẵn trong image, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window (không `--full-scene`, `rear=0` ở cả ba lượt); score threshold 0.3, giữ cố định A/B/C.
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance thật bị bỏ khỏi PCD, RGB=0 là placeholder; script đọc cloud hai lần với reflectance hằng (0 cho `vehicles`, 0.7 cho `pedestrian`/`two-wheels`) — không phải intensity được phục hồi. `z_ground = 0.075 m` do script ước lượng từ PCD, không phải đo mặt đường.

## Ba lượt inference thật

Số hộp lấy từ `n_boxes`, mean_z từ `mean_z` trong `output/run-*/summary.csv`. Số hộp/mean_z không phải điểm chất lượng hay đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `output/run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Chỉ 1 hộp `vehicles` tại (x 13.15, y −0.45), score 0.32 — sát ngưỡng 0.3. Các cụm điểm khác trên Side không có hộp |
| B | 1.73 | 0.16 | 13 | 1.034 | `output/run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; tâm z 0.70–1.43 m, đáy hộp quanh đường z=0 trên Side; score 0.32–0.93 |
| C | 1.73 | 0.32 | 6 | 1.091 | `output/run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Cả 6 hộp đều `pedestrian` (dài 0.59–1.07 m, cao 1.68–1.79 m); không còn `vehicles`/`two-wheels` |

- **A/B — chỉ đổi delta.** Thay input trước model khác hẳn dịch một hằng số cho output. Nếu chỉ là dịch output, A và B phải có cùng số hộp và z lệch đều 1.73 m. Thực tế A có 1 hộp, B có 13. Hộp duy nhất của A (x 13.15, y −0.45, z 0.33) gần nhất với hộp `vehicles` của B tại (14.77, −1.08, z 0.90): cách 1.74 m theo x/y, z chỉ chênh 0.57 m chứ không phải 1.73 m. Giải thích: khi delta=0, cloud không được hạ về độ cao sensor mà checkpoint KITTI đã học, nên pillar/feature khác và model gần như không nhận ra đối tượng.
- **B/C — chỉ đổi pillar.** Từ 13 hộp xuống 6, mất toàn bộ `vehicles`/`two-wheels`. Đối chiếu tọa độ x/y với B: 3/6 hộp C cách một hộp `vehicles`/`two-wheels` của B ≤ 1.5 m (0.35 m với `two-wheels` tại x 10.3; 1.29 m và 1.47 m với hai `vehicles`), 2/6 cách 1.53 m và 3.02 m, và 1/6 (x 33.5, y −15.6) cách hộp gần nhất của B 8.41 m. Tức phần lớn hộp C nằm ở vùng B gọi là xe nhưng bị gán `pedestrian`. Checkpoint được train với pillar 0.16, nên 0.32 đổi biểu diễn input của mạng. **Không đủ bằng chứng để nói cấu hình nào tốt hơn** chỉ từ số hộp/score; cũng không có ảnh camera hay nhãn đã duyệt để xác nhận.
- **Giới hạn ROI và góc Side.** Chỉ cửa sổ phía trước được chạy, nên vật phía sau/ngoài ROI không phải bằng chứng model bỏ sót. Side là hình chiếu x-z của cả scene: các hộp ở y khác nhau chồng lên nhau (5 hộp của B có tâm x 3.7–10.3 m, chồng nhau trên Side), nên không đọc được yaw và khó tách từng hộp; hộp xa (x ≈ 55.6 m, score 0.50) chỉ có ít điểm thưa. Đường z=0 là đường tham chiếu của plot, không chứng nhận mặt đường cục bộ. Kiểm yaw/kích thước cần góc Trên/Trước và camera.
- **JSON nào chưa đủ cơ sở để import?** Không JSON nào: đây là prediction thô trên PCD KITTI demo, khác frame với job Robotaxi, và các ca `case-*.json` là biến đổi có chủ đích. Với pipeline thật, cần kiểm transform z (không lệch cả batch), rồi từng hộp qua Top/Side/Front + camera, đặc biệt các hộp score thấp (A: 0.32; B: `pedestrian` 0.32/0.34, `two-wheels` 0.38).

## Ca QC có kiểm soát — không import CVAT

Đã đối chiếu từng trường giữa `case-correct` và hai ca lỗi: `case-correct` trùng hoàn toàn `boxes` của lượt B.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | — | Không (giống hệt prediction B) | Không phát hiện lỗi transform; vẫn phải kiểm từng hộp vì đây chỉ là prediction | `output/qc-cases/case-correct.json`, `side-correct.png` |
| case-batch-z | 13/13 | −1.805 m (= delta 1.73 + z_ground 0.075) ở mọi hộp; tâm z còn −1.11 đến −0.38 m | Không; chỉ trường `z` đổi | **Dừng batch**: lệch đồng loạt cùng lượng, cùng chiều, đúng bằng phép chuyển ngược bị quên → báo LC kiểm transform, tạo lại prediction, không sửa tay | `output/qc-cases/case-batch-z.json`, `side-batch-z.png` |
| case-one-box-z | 1/13 (hộp đầu, `vehicles` tại x 8.09, y 1.21) | −1.805 m chỉ hộp đó (z 0.92 → −0.88) | Không; 12 hộp còn lại giống hệt `case-correct` | **Kiểm từng hộp**: chỉ một hộp lệch, các hộp khác bình thường → kiểm hộp đó qua nhiều view, không kết luận lỗi pipeline | `output/qc-cases/case-one-box-z.json`, `side-one-box-z.png` |

Các ca do helper `pipeline-qc-cases.py` tạo bằng biến đổi có chủ đích từ prediction thật của lượt B (`manifest.json`: `training_only: true`), không phải kết quả inference riêng hay nhãn đúng.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

**Đặng Trường Huy (2A202602184)** — vai trò: vận hành + ghi log lượt A và C, kiểm JSON + xem hình học lượt B.
<!-- BẢN NHÁP: Huy tự đọc lại, sửa bằng lời của mình rồi xoá dòng này. -->
- Quan sát: `run-A/boxes-demo-delta-0-voxel-0.16.json` chỉ có 1 hộp `vehicles` score 0.32, trong khi `run-B` trên cùng PCD có 13 hộp, score cao nhất 0.93.
- Phép z: thuận `z_model = z_source − z_ground − delta` đưa cloud về độ cao sensor checkpoint KITTI quen; ngược `z_source = z_model + delta + z_ground` đưa hộp về hệ nguồn. Đổi delta là đổi input nên model ra tập hộp khác, không phải dịch hộp cũ.
- Quyết định lỗi batch: `case-batch-z` lệch 13/13 hộp đúng −1.805 m, các trường khác giữ nguyên → dừng sửa tay, báo LC kiểm phép chuyển ngược.
- Chưa chắc: `z_ground = 0.075 m` chỉ là ước lượng; không có camera nên chưa xác nhận được class của các hộp score thấp.

**Tô Văn Anh Quân (2A202602231)** — vai trò: kiểm JSON + xem hình học lượt A và C, vận hành + ghi log lượt B.
<!-- TODO: Quân tự viết 4 ý: (1) một quan sát A/B/C có dẫn file/hộp/vùng, (2) diễn giải phép z thuận/ngược, (3) một quyết định lỗi batch/từng hộp và hành động, (4) điều chưa chắc. Gợi ý chủ đề khác với Huy: B/C đổi pillar (output/run-C/...) hoặc case-one-box-z. -->
- Quan sát:
- Phép z:
- Quyết định lỗi batch/từng hộp:
- Chưa chắc:

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
