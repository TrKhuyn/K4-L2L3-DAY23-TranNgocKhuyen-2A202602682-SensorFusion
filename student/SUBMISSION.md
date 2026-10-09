# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Trần Ngọc Khuyến
- MSSV: 2A202602682
- Email: trankhuyen.work@gmail.com
- Link repo (fork): https://github.com/TrKhuyn/K4-L2L3-DAY23-TranNgocKhuyen-2A202602682-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`): Lấy hash của commit CP6 bằng `git rev-parse HEAD` sau khi commit báo cáo, rồi nộp hash đó trên VLearn.

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: `compare`, `[0, 198]` (199 frame/mode), `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`, `0`.
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: `0.9700934579439252`, `0.7004048582995951`, `519 / 16 / 222`.
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: `0.15032268781360134 m`, `502`, `11.343649056695735 m²`, `0`, `239`, `2.522613065326633`.
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: `0.17109183025633565 m`, `502`, `14.694752018992311 m²`, `0`, `239`, `2.522613065326633`.
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: Fused có RMSE cao hơn LiDAR `0.02076914244273431 m`, nhưng vẫn trong mức chênh lệch cho phép `0.05 m`. Cả hai có 502 matches, 0 ghost và 239 missed GT frames, cùng trung bình 2.5226 confirmed tracks/frame. Vì số cặp ghép không giảm, RMSE khác nhau phản ánh sai số vị trí của những cặp được đánh giá, không phải giảm số matches để có RMSE thấp. Chỉ số tổng hợp bằng nhau không chứng minh ID hoặc từng cặp ghép giống nhau ở mọi frame.

| Chỉ số | LiDAR | Fused | Ngưỡng đầy đủ hệ số chất lượng |
|---|---:|---:|---:|
| RMSE vị trí 3D (m) | 0.150323 | 0.171092 | ≤ 0.45 |
| Matches | 502 | 502 | Đọc cùng RMSE |
| Ghost track frames | 0 | 0 | Đọc cùng RMSE |
| Missed GT frames | 239 | 239 | Đọc cùng RMSE |
| Precision track | 1.0 | 1.0 | ≥ 0.75 |
| Coverage (`matches / det_tp`) | 0.9672447013487476 | 0.9672447013487476 | ≥ 0.70 |

`precision_track = 502 / (502 + 0) = 1`, `coverage = 502 / 519`. Hai mode đều có hệ số chất lượng `q = 1`; phần tracking tự động theo công thức rubric đạt mức 45/45 nếu bộ test gốc khi chấm cũng đạt đủ E–H. Đây là đối chiếu công thức, không thay cho điểm giảng viên chấm thủ công/báo cáo.

Tổng có 741 GT xe hợp lệ qua các frame (`519 + 222`); 239 missed GT frames không đồng nghĩa 239 xe riêng biệt. Detector có 222 FN, còn confirmed tracker ghép được 502 GT. Coverage rubric dùng 519 detection TP làm mẫu số, không dùng toàn bộ 741 GT. Track mới cần tích lũy hit trước khi confirmed, nên các frame đầu có thể thiếu confirmed tracks.

Ví dụ đối chiếu trực tiếp tại frame 198 trong [grade_run_lidar.log](artifacts/grade_run_lidar.log) và [grade_run_fused.log](artifacts/grade_run_fused.log): cả hai ghi `confirmed=3`, `matches=3`, `ghosts=0`, `misses=6`, nhưng `sum_sq_err` lần lượt là `0.06297798579477262` và `0.06856220409445983 m²`. Điều này cho thấy camera update có thể tăng sai số trên frame cụ thể. Nhãn 2D camera biểu diễn tâm hộp ảnh; phép chiếu tâm 3D không nhất thiết trùng tâm hộp 2D, ngoài ra còn có nhiễu và tuyến tính hóa EKF. Đây là các khả năng giải thích, chưa được tách riêng bằng thí nghiệm calibration.

Nguồn số liệu: [metrics.json](artifacts/metrics.json), [grade_run.log](artifacts/grade_run.log) và bốn file per-mode. Đã dùng `validate_metrics_records` đối chiếu cả compare (398 record) và mỗi mode (199 record): đủ các frame 0–198, không trùng/thiếu `(mode, frame)`, invariant đúng và tổng/trung bình khớp metrics. Detection nhất quán giữa hai mode. Artifacts được runner sinh sau thay đổi code cuối cùng, không sửa tay.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. Khác biệt đo lidar 3D và camera 2D trong EKF (`z`, `R`)?
2. Vì sao cần gating Mahalanobis trước khi gán?
3. Pipeline là track-then-fuse hay fuse-then-track? Chỉ ra trên log `fusion-run-lab`.
4. Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?
5. Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?
   Giải thích vì sao lidar quyết định score/init/delete còn camera chỉ EKF update.
6. Nêu điều kiện xác nhận, giữ confirmed sau miss, và điều kiện xóa track.

### Trả lời

1. LiDAR đo tâm hộp 3D trong hệ cảm biến: `z` có shape 3×1, `R` có shape 3×3 với phương sai vị trí (m²). Mô hình đo tuyến tính dùng phép đổi tọa độ vehicle → sensor, `H` có shape 3×6. Camera đo `(u, v)` có shape 2×1, `R` có shape 2×2 theo pixel²; trong `camera_fusion.build_camera_measurement`, phương sai là `sigma_cam_i²`, `sigma_cam_j²` (mặc định 25 pixel²). Camera dùng `h(x)` pinhole phi tuyến và Jacobian 2×6 do `Sensor.get_H` của platform cung cấp. Cả hai update cùng state 6D bằng `kalman.ekf_update`; không trộn tọa độ mét với pixel.

2. Gating dùng `d² = gamma.T @ solve(S, gamma)`, với `gamma = z - h(x)` và `S = H P H.T + R`, để xét residual tương ứng với độ bất định. Khoảng cách Euclid không phản ánh covariance hoặc khác biệt đơn vị giữa hai sensor. `association.chi2_gate` dùng phân vị χ² ở `gating_threshold = 0.995`, bậc tự do bằng số chiều đo (LiDAR 3, camera 2). Cặp ngoài FOV bị loại trước khi tính Mahalanobis, tránh phép chiếu điểm sau camera hoặc có độ sâu không hợp lệ. Trong các cặp qua gate, `pick_next_pair` chọn cost hữu hạn nhỏ nhất rồi bỏ hàng/cột để ghép một-một.

3. Đây là **track-then-fuse**: chỉ một danh sách track trong mỗi mode. Trong `platform/fusion_lab/scripts/run_lab.py`, vòng frame gọi `KF.predict` một lần cho mỗi track, sau đó `associate_and_update` với LiDAR, rồi camera nếu mode fused có dữ liệu FRONT. Không predict lại giữa hai sensor: Waymo Frame gom các đo cùng bước thời gian logic `t = frame_index * dt`. Log `grade_run.log` ghi kết quả sau các bước này theo `(mode, frame)`; log không ghi riêng từng predict/update, nên thứ tự được xác nhận từ code runner, còn log dùng để đối chiếu kết quả hai mode. Detection là hộp 3D từng frame; tracking duy trì ID, vị trí, vận tốc và covariance qua nhiều frame.

4. Extrinsic/intrinsic camera bị lệch làm pixel dự báo `h(x)` lệch có hệ thống so với đo, dẫn đến innovation không còn dao động quanh 0; độ lệch có thể phụ thuộc khoảng cách và vị trí xe trong ảnh. Residual lớn tăng Mahalanobis và có thể bị χ² gate chặn. Nếu bias vẫn nằm trong gate, camera update có thể kéo state sai và làm RMSE fused tăng. Bài bắt buộc này không thực hiện thí nghiệm calibration; đây là giải thích cơ chế, không phải số liệu thực nghiệm.

5. Khi `meas_list` rỗng, không thể suy sensor từ measurement. `associate_and_update(..., sensor)` vẫn gọi `manager.manage_tracks` với sensor tường minh: lượt LiDAR rỗng phải xử lý miss trong FOV và xóa track nếu cần; lượt camera rỗng không tác động lifecycle. LiDAR tạo state 3D và quyết định tồn tại. `TrackManager.handle_updated_track` chỉ cộng score cho hit LiDAR; `manage_tracks` trả về ngay khi sensor không phải LiDAR. Camera chỉ `filter_obj.update` để tinh chỉnh state/covariance, không tạo, xác nhận hoặc xóa track, tránh tính hit/miss hai lần trong một frame.

6. Trong `track_management.init_track_state_from_meas`, tâm đo được đổi về hệ xe bằng `sens_to_veh`; vận tốc ban đầu bằng 0. Covariance vị trí là `R_rot @ meas.R @ R_rot.T`, covariance vận tốc là bình phương `sigma_p44/55/66`. Track mới có state `initialized`, score `1/window`. Mỗi hit LiDAR cộng `1/window` và chặn tối đa 1; miss LiDAR trong FOV trừ `1/window`. Khi score **>** `confirmed_threshold` (0.8), track thành `confirmed`; hit chưa đạt ngưỡng tạo `tentative`. Track đã confirmed giữ trạng thái sau miss, nhưng vẫn bị xóa nếu score **<** `delete_threshold` (0.6). Track chưa confirmed bị xóa khi score **≤ 0**; mọi track bị xóa nếu `P[0,0]` hoặc `P[1,1]` **>** `max_P` (9 m²). Tất cả tham số đọc qua `get_tracking_params()`.

### Kiểm thử và môi trường

- Python 3.12.13, môi trường `.venv`, cài editable Waymo reader, platform và student bằng uv.
- `fusion-run-lab --help` chạy thành công, không lỗi import.
- Part E: 4 test passed; nhóm camera CP2: 20 passed; nhóm association CP3: 9 passed.
- Toàn bộ `pytest student/tests -q`: **128 passed**, không failed/xfailed. Trên Windows cần `PYTHONUTF8=1` để test/báo cáo tiếng Việt dùng UTF-8.
- Part A–D được đọc hiểu và giữ nguyên; `platform/` và test giữ nguyên.
- Cấu hình local dùng dữ liệu/weights trong `data/Day23-Fusion/`; `.env`, `paths.yaml`, dữ liệu và weights không commit.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không thực hiện bonus.

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): OpenAI Codex (AI hỗ trợ).
- Dùng cho phần nào (hàm, câu hỏi, debug): Đọc yêu cầu lab; cài môi trường; triển khai 17 hàm E–H; chạy test, Waymo; tổng hợp số liệu và soạn phần giải thích trong báo cáo.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): Codex đã đối chiếu hợp đồng stub, công thức EKF, API sensor/lifecycle; chạy bộ test gốc trong repo (128 passed), pipeline compare đủ frame 0–198 với seed 0; kiểm tra compare và từng mode bằng `validate_metrics_records`. Học viên cần tự đọc lại code, câu trả lời và kết quả để giải thích được khi vấn đáp; báo cáo không khẳng định học viên đã tự thực hiện kiểm tra nếu chưa làm.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [x] Part A–D: giữ nguyên code cung cấp
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [x] Đã điền đủ file này, gồm khai báo AI
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [x] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
