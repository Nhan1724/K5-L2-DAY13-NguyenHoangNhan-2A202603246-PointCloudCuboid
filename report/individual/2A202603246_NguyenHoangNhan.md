# Báo cáo Day 13 — Nguyễn Hoàng Nhân, 2A202603246

## 1. Kết luận trước
Mã số `2A202603246`, task `D13-2A202603246`, gồm 10 frame: `s01f19`, `s01f45`, `s02f00`, `s02f35`, `s03f00`, `s03f53`, `s04f02`, `s04f36`, `s05f02`, `s05f48`. Hệ điều hành / trình duyệt / phiên bản CVAT: Windows 11 / Google Chrome / CVAT (bản web của lớp). MSSV bạn ngồi cạnh: `2A202603010`.

Tổng cộng tôi gán 78 cuboid (68 Car, 5 Truck, 3 Cyclist, 2 Bus, không có người đi bộ). Frame nhiều nhất là `s04f02` (14 cuboid), ít nhất là `s05f48` (4 cuboid) — vì ở `s05f48` đường vắng, ít phương tiện lưu thông và các cụm điểm ở xa quá thưa nên tôi bỏ qua theo quy tắc. `selfcheck` cờ 6 trên 14 cuboid của frame `s04f02` (7 cảnh báo: 2 chìm, 1 lơ lửng, 4 kích thước ngoài khoảng). Phần tôi tin nhất: góc xoay `yaw` và vị trí tâm `(x, y)` của nhóm xe chạy cùng làn, vì phần lớn xe cùng chiều đều cho góc rất ổn định quanh 266,2°; phần yếu nhất: cao độ đáy cuboid (`z_bottom`) và kích thước `(l, w, h)` ở các xe ở xa hoặc nằm lệch bên hông đường (như nhóm idx 11–14 của `s04f02`), do mặt đường có độ dốc và điểm LiDAR thưa khiến việc ước lượng khung bao dễ bị dư.

Cách dùng CVAT: 
- **Đổi góc nhìn:** Dùng chuột trái để xoay góc nhìn 3D, chuột phải (hoặc giữ Shift + kéo) để dời khung nhìn (pan), cuộn con lăn để phóng to/thu nhỏ, và bấm phím tắt chuyển nhanh sang góc nhìn Top (nhìn từ trên xuống).
- **Vẽ & chỉnh cuboid:** Chọn công cụ **Draw new cuboid** (Shape), kéo khung trên mặt phẳng Top rồi nháy đúp để mở 3 cửa sổ chiếu trực giao **Top / Side / Front** nhằm tinh chỉnh kích thước, vị trí tâm và xoay hướng mũi tên.
- **Kiểm tra chéo:** Bật hiển thị **Cuboid orientation** (mũi tên hướng đầu xe) và đối chiếu khung 3D chiếu ngược lên ảnh camera ở cửa sổ `preview` để chắc chắn khung khớp với hình dáng thật của xe.

---

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ: `s04f02` và `s03f00`. Ảnh: `t1-s04f02-3d`, `t1-s03f00-top`.

| Frame | Phía trước xe ở cạnh nào của ảnh top; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| `s04f02` | Phía trước xe ego hướng xuống cạnh dưới (trùng chiều mũi tên xe chạy cùng làn `yaw ≈ 266,2°`); trục `+y` hướng dọc theo chiều đường (về phía sau xe ego khi `y < 0`), trục `+x` hướng vuông góc sang ngang làn đường | Hơi dốc nhẹ về phía trước/bên trái; mặt đường thấp hơn cảm biến LiDAR khoảng `1,8–2,0 m` ở làn chính (khu vực bên trái `x < -23 m` nền cao hơn, quanh `-1,2 m`) | `~42,0 m` (`idx 11`, Car) | Car (14) | Có nhiều xe phía sau (`y < 0`, như `idx 4–8`) và hai bên: làn ngược chiều bên phải (`x ≈ +13…+17 m`) và bãi/nhánh đường bên trái (`x ≈ -23…-29 m`) | Các cụm điểm rất thưa ở khoảng cách `> 45 m` và dải cây/cột bên vỉa hè không phải đối tượng giao thông |
| `s03f00` | Hướng đầu xe ego trùng với nhóm xe cùng chiều (`yaw ≈ 266°–271°`); trục `+y` chạy dọc theo tim đường, trục `+x` hướng ngang sang hai bên vỉa hè | Tương đối phẳng; mặt đường thấp hơn LiDAR khoảng `1,85–2,05 m` | `~23,6 m` (`idx 7`, Car) | Car (7), Truck (1) | Có xe chạy phía sau (`y < 0`, như `idx 2, 5, 6, 7`) và xe phía trước (`y > 0`, như `idx 1, 3, 4, 8`) trên các làn lân cận | Một số cụm điểm nhỏ sát lề đường ở xa không đủ đặc trưng hình khối để xác định class |

* **3 điều chỉ camera cho biết:** 
  1. Màu sắc, biển số, chữ viết hoặc logo trên thân xe (ví dụ phân biệt xe taxi, xe buýt tuyến hay xe tải chở hàng).
  2. Trạng thái đèn tín hiệu (đèn phanh đang sáng, đèn xi-nhan, đèn pha) và màu đèn giao thông.
  3. Các vật thể phẳng không có độ dày 3D như vạch kẻ đường, biển báo vẽ trên mặt đường hoặc bóng râm.
* **3 điều chỉ LiDAR cho biết:** 
  1. Khoảng cách thực tế và kích thước hình học chính xác của vật thể tính bằng mét `(x, y, z, l, w, h)`.
  2. Các vật thể nằm phía sau lưng hoặc bên hông ngoài góc nhìn (FOV) của camera phía trước.
  3. Hình dạng cấu trúc 3D và độ cao gầm/đáy vật thể so với mặt đường ngay cả trong điều kiện thiếu sáng hoặc chói nắng.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh top | Điều lạ (nếu có) |
|---|---|---|
| `s01f19` | Trùng chiều mũi tên nhóm xe làn chính (`yaw ≈ 266,2°`, hướng dọc trục `y`) | Có 1 xe buýt dài `12,02 m` ở `~23,4 m` phía sau (`idx 1`); làn ngược chiều có 2 xe (`yaw ≈ 93°–96°`) |
| `s01f45` | Dọc theo trục đường chính (`yaw ≈ 266°–270°`) | Có xe tải dài `8,88 m` ở `~36,5 m` phía sau (`idx 7`, `z_bottom = -2,60 m` do đường dốc) |
| `s02f00` | Dọc theo trục đường chính (`yaw ≈ 266,2°`) | Có xe buýt dài `12,69 m` ở `~9,2 m` (`idx 7`); `idx 1` có `z_bottom = -1,02 m` cao hơn mặt đường |
| `s02f35` | Dọc theo trục đường chính (`yaw ≈ 266,2°` cùng chiều, `88°–91°` ngược chiều) | Không có vật bất thường, chủ yếu là 5 xe con phân bố rải rác từ `6,2 m` đến `27,3 m` |
| `s03f53` | Dọc theo trục đường chính (`yaw ≈ 266,2°`) | Có 1 xe hai bánh (`Cyclist`, `idx 1`) ở khoảng cách `11,4 m`, bề rộng hẹp (`0,54 m`) |
| `s04f36` | Dọc theo trục đường chính (`yaw ≈ 266,2°` và `89°–91°`) | Có 2 xe hai bánh (`idx 3, 5`), 1 xe tải (`idx 4`), và xe con ở xa nhất toàn bộ dataset (`idx 11`, `47,27 m`) |
| `s05f02` | Dọc theo trục đường chính (`yaw ≈ 266,2°` và `89°–90°`) | Có xe tải dài `8,28 m` ở `~12,6 m` phía trước (`idx 10`), mật độ xe khá đông (10 cuboid) |
| `s05f48` | Dọc theo trục đường chính (`yaw ≈ 88°–92°` và `270°`) | Chỉ có 4 cuboid; có xe tải ngắn (`5,02 m`) ở rất gần (`~4,75 m`, `idx 4`) |

---

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh: `t2-a-car`, `t2-b-bus-truck`, `t2-c-ped-cyclist`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | `s03f00, 2` (`4,04 m`) | Car | Đúng (`271,7°`, hướng dọc theo làn đường cùng chiều) | Chạm đường (`z_bottom = -2,048 m`, khớp mặt đường xung quanh) | Đã kiểm đủ 3 mặt chiếu Top/Side/Front và khớp khung `preview` trên ảnh | Có (`4,64 × 2,13 × 1,78 m`, nằm gọn trong chuẩn Car `3,0–6,0 × 1,4–2,3 × 1,2–2,3 m`) |
| (b) xe buýt / xe tải | `s02f00, 7` (`9,21 m`) | Bus | Đúng (`270,1°`, đầu xe hướng theo chiều di chuyển của làn) | Chạm đường (`z_bottom = -1,997 m`, bám sát mặt đường) | Đã kiểm Top/Side/Front và đối chiếu ảnh camera | Có (`12,69 × 3,34 × 3,59 m`, đúng tỷ lệ xe buýt cỡ lớn) |
| (c) người / xe hai bánh | `s04f36, 3` (`12,04 m`) | Cyclist | Đúng (`267,6°`, hướng dọc theo chiều chuyển động) | Chạm đường (`z_bottom = -1,437 m`, phù hợp cao độ lề đường) | Đã kiểm Top/Side/Front và xác nhận đúng người đi xe trên ảnh | Có (`1,89 × 0,93 × 1,39 m`, hợp lý cho cả người và xe hai bánh) |

* **Vòng hiệu chuẩn:** Cuboid của tôi lúc đầu hay bị đặt tâm `z` hơi cao hoặc hơi thấp khi nhìn ở góc 3D nghiêng, và có xu hướng kéo dài khung hơi dư ở phần đuôi xe bị khuất tia LiDAR. Sau khi trao đổi, cả lớp chốt cách hiểu guideline: luôn hạ đáy cuboid chạm đúng lớp điểm mặt đường cục bộ quanh bánh xe ở cửa sổ **Side/Front**, và với xe bị khuất một phần phía sau thì giữ kích thước theo chuẩn loại xe thay vì kéo khung ôm cả điểm nhiễu xung quanh.
* **Vật bỏ qua vì không đủ cơ sở:** 
  - Frame `s05f48`, khu vực phía xa cuối đường (`dist > 45 m`), cụm điểm rời rạc chỉ có dưới 15–20 điểm rời, không rõ biên dạng nóc và bánh xe $\rightarrow$ bỏ qua do `LOW_VISIBILITY` / `TOO_SPARSE`.
  - Frame `s04f02`, cụm điểm nhỏ sát dải phân cách/vỉa hè bên phải (`x > 20 m`), không phân biệt được là biển báo, thùng rác hay người đứng $\rightarrow$ bỏ qua do `UNCERTAIN_CLASS`.

---

## 4. Bốn hiện tượng chất lượng (Bước 4)

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che (Occlusion) | `s04f02`, `idx 4` và `idx 5` (tự gán trên task `D13-2A202603246`) | Xe ở gần (`idx 3` tại `4,7 m`) chắn chùm tia LiDAR chiếu tới các xe chạy phía sau (`15,4–16,5 m`) | Chỉ thấy được phần đầu hoặc một góc sườn xe; phía sau thân xe tạo thành vùng trống (vùng bóng tối) không có điểm | Không co nhỏ cuboid theo phần điểm nhìn thấy mà phải kéo đủ chiều dài chuẩn của xe (`~4,64 m`) dựa theo hướng `yaw` và vị trí đầu xe |
| 2. Thưa ở xa (Sparsity) | `s04f02`, `idx 7` (`31,8 m`, 129 điểm) và `idx 11` (`42,0 m`, 130 điểm) | Chùm tia laser phân kỳ theo góc quét nên mật độ tia giảm mạnh theo bình phương khoảng cách | Chỉ còn vài vệt điểm ngang mỏng ở phần thân/kính hậu, mất chi tiết bánh xe và biên dạng góc cạnh | Dễ ước lượng sai chiều dài hoặc cao độ đáy; phải dựa vào `z` mặt đường lân cận và kiểm tra kỹ trên ảnh `preview` để tránh gán lố kích thước |
| 3. Phản xạ thấp / ít điểm bất thường | `s02f00`, `idx 1` (`12,47 m`) hoặc `s04f02`, `idx 5` (`16,45 m`, chỉ có 241 điểm) | Bề mặt thân xe sơn màu tối/đen bóng hoặc kính cửa sổ hấp thụ và phản xạ lệch hướng tia laser | Thân xe bị rỗng ở giữa, số điểm ít hơn hẳn so với các xe cùng khoảng cách (ví dụ `s04f02 idx 4` ở `15,4 m` có tới 701 điểm trong khi `idx 5` ở `16,4 m` chỉ có 241 điểm) | Dễ đặt nhầm tâm `z` khiến cuboid bị "lơ lửng" (`idx 5` cao hơn mặt đường `0,90 m`) do thiếu điểm ở phần gầm và bánh xe |
| 4. Điểm lạ (Noise / Ghost points) | `s04f02`, khu vực quanh `idx 13–14` (`x ≈ -23…-29 m`) | Tia laser phản xạ đa đường (multipath) qua kính xe hoặc lẫn điểm của bụi, cây cối, dải phân cách sát mép xe | Xuất hiện các điểm lơ lửng phía trên nóc hoặc dính liền vào sườn xe khiến cụm điểm phình to bất thường | Nếu kéo cuboid ôm trọn cả điểm lạ sẽ làm kích thước vượt ngưỡng bảng 4.3 (như `idx 13` dài `6,32 m`, cao `2,31 m`); cần loại bỏ điểm nhiễu khỏi khung |

---

## 5. Số từ cuboid của tôi (Bước 5)
Nguồn: `submission/D13-2A202603246-bangA.csv`; số điểm lấy từ cột `points_in` của `-tukiem.csv`. Nhận xét dùng `ground_gap_m` của phiếu tự kiểm (mặt đường cục bộ của frame), ngưỡng `0,3 m`.

| Cuboid (frame, idx, class) | z | h | z đáy = z − h/2 | dist_m | r = √(x²+y²+z²) | yaw_deg | Nhận xét |
|---|---|---|---|---|---|---|---|
| `s04f02, 1, Car` | −0,913 m | 1,840 m | −1,833 m | 24,84 m | 24,85 m | 266,2° | Chạm đường (`ground_gap_m = -0,04 m`, đạt chuẩn) |
| `s04f02, 3, Car` | −0,951 m | 2,148 m | −2,025 m | 4,67 m | 4,77 m | 266,2° | Chìm `0,28 m` (`ground_gap_m = -0,28 m`, gần ngưỡng cảnh báo `0,3 m`) |
| `s04f02, 5, Car` | 0,017 m | 1,781 m | −0,874 m | 16,45 m | 16,45 m | 266,2° | Lơ lửng `0,90 m` (`ground_gap_m = +0,90 m`, vượt ngưỡng cảnh báo) |

* **(a) Khoảng cách gần nhất và xa nhất:** 
  - Gần nhất toàn bộ bảng A: `s03f00 idx 2` (Car) có `dist_m = 4,04 m`, `r = 4,20 m`. 
  - Xa nhất toàn bộ bảng A: `s04f36 idx 11` (Car) có `dist_m = 47,27 m`, `r = 47,28 m`. 
  - Giải thích: Ở xa `r ≈ dist_m` (chênh chỉ `0,01 m`) vì cao độ `z` (`-1,18 m`) rất nhỏ so với khoảng cách mặt phẳng ngang (`47,27 m`); còn ở gần (`4,04 m`), độ cao `z = -1,16 m` chiếm tỷ lệ đáng kể nên `r` lớn hơn `dist_m` rõ rệt (`0,16 m`).

* **(b) Đối chiếu kích thước với bảng 4.3:** 
  - **Xe hai bánh (3 cuboid):** `l = 1,69–1,89 m`, `w = 0,54–0,93 m`, `h = 1,17–1,39 m`. So với bảng 4.3 (chuẩn Cyclist thường `w ≈ 0,6–1,0 m`, `h ≈ 1,4–2,0 m`): chiếc `s03f53 idx 1` có chiều rộng `w = 0,54 m` (hơi hẹp dưới mức 0,6 m, do góc nhìn nghiêng chỉ quét được một mặt sườn) và cả 3 cuboid đều có chiều cao `h` từ `1,17–1,39 m` hơi thấp hơn mức 1,4 m (có thể do người lái cúi người hoặc mất điểm đỉnh đầu).
  - **Xe con (68 cuboid):** Phần lớn có kích thước chuẩn quanh `l ≈ 4,64 m`, `w ≈ 2,13 m`, `h ≈ 1,78 m`, nằm gọn trong khoảng `3,0–6,0 / 1,4–2,3 / 1,2–2,3 m`. Các trường hợp ngoài khoảng tập trung ở `s04f02`: `idx 11` (`l = 6,33 m`), `idx 13` (`l = 6,32 m, h = 2,31 m`) và `idx 14` (`w = 2,51 m`). 
  - **Lý do:** Các xe `idx 11, 13, 14` đều nằm ở khu vực rìa trái (`x ≈ -23…-29 m`), khoảng cách xa (`23–42 m`) và đỗ xoay chéo góc (`yaw` từ `281,5°` đến `301,3°` hoặc `5,1°`), điểm thưa và dính sát lề/vật cản bên cạnh nên khi kéo khung bao đã vô tình gộp thêm điểm nhiễu xung quanh.

* **(c) Góc hướng (yaw):** 
  - Xe cùng chiều phía trước/sau: `s04f02 idx 1` có `yaw = 266,2°` (chỉ lệch `3,8°` so với trục `270°`). 
  - Xe ngược chiều: `s04f02 idx 8` có `yaw = 91,5°` (chỉ lệch `1,5°` so với trục `90°`). 
  - Nhận xét: Hai hướng di chuyển chính ngược chiều nhau lệch nhau gần đúng `180°` (`266,2° - 91,5° = 174,7°`), bám sát hệ trục tọa độ của xe ego.

**Mật độ điểm theo khoảng cách (cuboid trong `s04f02`):**

| Cuboid (frame, idx, class) | dist_m | Số điểm (`points_in`) |
|---|---|---|
| `s04f02, 3, Car` | 4,7 m | 4974 |
| `s04f02, 4, Car` | 15,4 m | 701 |
| `s04f02, 6, Car` | 25,4 m | 203 |
| `s04f02, 7, Car` | 31,8 m | 129 |

* **Quy luật:** Khi khoảng cách tăng gấp ~2 lần thì số điểm giảm còn khoảng 1/4 (từ `4,7 m` ra `31,8 m`, khoảng cách tăng `6,8 lần` thì số điểm giảm `38,6 lần`, tuân theo quy luật tỉ lệ nghịch với bình phương khoảng cách $1/d^2$). So với đường cong chuẩn trên slide (`1968 / 718 / 214 / 72` điểm ở các mốc `0–10 / 10–20 / 20–30 / 30–40 m`): mốc `15,4 m` (`701 điểm`) và `25,4 m` (`203 điểm`) khớp rất sát với `718` và `214`; mốc `4,7 m` (`4974 điểm`) cao hơn mức trung bình `1968` vì xe ở cực gần cảm biến; còn mốc `31,8 m` (`129 điểm`) cao hơn `72 điểm` trên slide do xe này (`idx 7`) nằm thẳng góc quét ngang nên hứng được nhiều tia hơn hoặc cuboid ôm dư một phần điểm mặt đường.

* **Tự kiểm:** 
  - **Hai cuboid đã sửa sau `preview` / `selfcheck`:** 
    1. `s04f02, idx 1`: ban đầu bị lệch hướng mũi tên `yaw` ngược đầu xe, sau khi bật `Cuboid orientation` và xem `preview` đã xoay lại đúng `266,2°` và hạ đáy chạm đường (`ground_gap_m = -0,04 m`).
    2. `s04f02, idx 4`: ban đầu bị co ngắn chiều dài do bóng che từ xe `idx 3`, sau khi kiểm tra cửa sổ Top đã kéo dài lại đúng chuẩn `4,64 m` và chỉnh đáy khớp mặt đường (`ground_gap_m = 0,04 m`).
  - **Cảnh báo giữ nguyên kèm lý do:** 
    - `s04f02, idx 3` (chìm `0,28 m`) và `idx 8` (chìm `0,25 m`): mức lệch nhỏ hơn ngưỡng `0,30 m`, giữ nguyên vì bánh xe có độ lún/phản xạ gầm và mặt đường tại vị trí đó hơi nghiêng cục bộ.
    - `s04f02, idx 5` (lơ lửng `0,90 m`) và nhóm `idx 11, 13, 14` (kích thước ngoài khoảng): đây là các thiếu sót tôi phát hiện sau khi chạy phiếu tự kiểm lúc đã xuất file nộp; tôi giữ nguyên số liệu gốc trung thực trong bảng A và ghi nhận đây là phần cần rút kinh nghiệm để chỉnh lại cao độ `z` và thu gọn khung ở khu vực rìa trái.

* **Đổi kiểm:** 
  - MSSV người kiểm chéo với tôi: `2A202603010` (bạn ngồi cạnh).
  - Hai con số đối chiếu với bảng A: (1) Tổng số cuboid trong bảng A là `78` dòng, khớp 100% với số lượng nhãn xuất ra từ CVAT; (2) Giá trị `z_bottom` của `s04f02 idx 1` tính tay từ `z - h/2 = -0,913 - 1,840/2 = -1,833 m`, khớp tuyệt đối với cột `z_bottom` (`-1,833 m`) trên bảng A.

---

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
* **(a) Ý nghĩa của nhãn 3D:** Mô hình nhận thức 3D của xe tự hành cần dự đoán chính xác tọa độ vị trí, kích thước thật và hướng di chuyển của vật cản trong không gian vật lý tính bằng mét để tính toán quỹ đạo phanh và tránh va chạm, điều mà ảnh 2D bị biến dạng phối cảnh theo khoảng cách không thể cung cấp trực tiếp. Bên cạnh đó, vì mật độ điểm LiDAR giảm cực nhanh ở xa (như xe ở `31,8 m` chỉ còn `129 điểm` so với `4974 điểm` ở `4,7 m`), việc tuân thủ chặt chẽ quy tắc ước lượng kích thước chuẩn cho vật thưa điểm giúp bộ dữ liệu huấn luyện đồng nhất, tránh làm mô hình học sai biên dạng vật thể.
* **(b) Dùng AI:** Công cụ sử dụng: **AI (Gemini / Claude)**
* **(c) Điều còn chưa chắc:** Tôi vẫn còn phân vân ở cách xác định cao độ mặt đường chuẩn tại những đoạn đường dốc hoặc gồ ghề ở rìa xa (như khu vực `x < -23 m` trong frame `s04f02`), vì khi không có điểm mặt đường ngay dưới gầm xe thì rất khó phân biệt giữa một chiếc xe bị gán lơ lửng hay do nền đường tại chỗ đó thực sự cao hơn làn chính. Ngoài ra, với các cụm điểm quá thưa ở khoảng cách trên `40 m`, ranh giới giữa việc "cố gắng ước lượng khung chuẩn" và "bỏ qua vì thiếu cơ sở" đôi khi vẫn mang tính cảm tính.