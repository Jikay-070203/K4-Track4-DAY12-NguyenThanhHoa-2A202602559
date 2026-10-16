# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Nguyễn Thanh Hoa` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602559` |
| GitHub | `Jikay-070203` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `1611 / 7770 (các seed mô hình: 7770–7774)` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5 (mặc định)` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.266 | 20.9 (1 lần chạy, không có seed) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.538 | 0.07 (2/2883) | (baseline) |
| EgoMLP | 0.765 ± 0.004 | 23.4 .. 26.2 (trung bình 24.9) | BLOCK (R2–R6) |
| PercMLP | 0.625 ± 0.003 | 0.45 .. 2.15 (trung bình 1.15) | BLOCK (R2–R6) |
| PercMLP+ (sau khi sửa) | 0.823 ± 0.007 | 0.17 .. 0.24 (trung bình 0.21) | BLOCK (R4, R6) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 cho thấy ba vấn đề. (1) Cột `Acc_LV` không dùng được: tương quan của nó với đạo hàm `Speed_LV` chỉ là **0.045**, trong khi cột gia tốc của xe theo sau là **0.929**. Vì vậy tôi dùng gia tốc suy ra từ `Speed_LV` đã làm mượt (`a_lead`) để khai thác ca phanh. (2) Có **383** quỹ đạo trùng hệt nhau nằm ở các id xa nhau (9993 → 9610 quỹ đạo sạch). Nếu chia theo khung hình hoặc chia ngẫu nhiên không loại trùng, cùng một chuyến đi sẽ nằm ở cả train lẫn test và làm số đo trên test đẹp một cách giả tạo (rò rỉ). Vì thế phải loại trùng trước rồi chia theo **quỹ đạo**: các khung hình của một quỹ đạo rất giống nhau nên chia theo khung hình cũng gây rò rỉ. (3) Dữ liệu gần như không chứa phanh gấp: giảm tốc đã làm mượt của xe dẫn chỉ tới −2.77 m/s² ở phân vị 0.1 % và −1.95 ở phân vị 1 %, còn bộ chấp hành làm được 9 m/s². Ghi chú thêm: `Spatial_Gap` đúng bằng hiệu vị trí trừ chiều dài xe 4.5 m (sai lệch tối đa 3.6e-11 m), nên khoảng cách đáng tin.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: **0/600** đoạn khác nhau và 0 đoạn không giống từng bit. Bản refactor float32 khác bit ở **600/600** đoạn nhưng vượt dung sai (atol = rtol = 1e-3) ở **0** đoạn, độ lệch lớn nhất chỉ **9.7e-05** m/s². Phép cộng số thực dấu phẩy động không có tính kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001 còn 0.1+(0.2+0.3) = 0.6), nên đổi thứ tự phép tính hay độ chính xác là đủ làm khác bit, và so sánh bằng `==` sẽ báo sai. Thay đổi AEB (phanh −6 m/s² khi TTC < 1.5 s) làm lệch **7/600** đoạn (87, 171, 179, 190, 234, 450, 473), độ lệch lớn nhất 1.98 m/s². Mở các đoạn đó ra: lệnh chỉ khác ở **1–4 khung** mỗi đoạn, và ở đúng các khung có TTC 1.42–1.49 s (ví dụ đoạn 87, khung 82: khoảng cách 6.16 m, tiến gần 4.2 m/s, TTC 1.47 s, lệnh −4.56 → −6.0). Vì luật AEB chỉ kích hoạt khi TTC < 1.5 s, các khung còn lại mọi lệnh giống hệt. Phát nhanh nhất cho 0 khung trễ và 0 khung rớt vì không có hạn chót. Phát theo thời gian thật lộ ra **585** khung trễ và **578** khung rớt trên 60 027 khung (2 % khung tốn 140 ms, vượt hạn chót 100 ms, khung kế tiếp bị rớt và lệnh cũ được giữ), làm **376/600** đoạn đổi lệnh vượt dung sai, độ lệch tới 9.38 m/s². Phát nhanh nhất che đi lỗi trễ hạn này.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới gồm **60 khoảng cách × 45 tốc độ tiến gần = 2700 ô**, tốc độ cruise 12 m/s, khoảng cách 2.76 .. 59.31 m (p2..p98 của log), tốc độ tiến gần 0 .. 12 m/s. Với độ trễ 0 s có **2.2 %** ô không ai tránh được; với 0.6 s là **7.4 %** (tăng 5.2 điểm phần trăm, gấp khoảng 3.3 lần). Bộ mô phỏng khớp biên vật lý: 0 ô lệch ngoài biên ở cả hai độ trễ. Vùng đâm của IDM đúng bằng vùng bất khả thi (2.2 % → 7.4 %, diện tích "tránh được mà vẫn đâm" = 0.0), nên IDM không thất bại; còn const-vel đâm 70.2 % ở độ trễ 0.6 s trong đó chỉ 7.4 % là bất khả thi, phần còn lại do bộ điều khiển. Log **không phủ** vùng khó: người lái trong log phanh mạnh nhất chỉ **3.62 m/s²** (phân vị 99.9) trong khi xe làm được 9 m/s², và **8.6 %** lưới đòi phanh mạnh hơn mức đó. Khai thác được **614** ca xe dẫn phanh (6.4 % của 9610 quỹ đạo), giảm tốc đỉnh p50 là −2.53 và p99 là −4.21 m/s².

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Va chạm vòng kín (%) | Hạng va chạm |
|---|---|---|---|---|
| constant-velocity | 1.266 | 3 | 20.9 | 3 |
| IDM | 1.538 | 4 | 0.07 | 1 |
| EgoMLP | 0.765 | 2 | 24.9 | 4 |
| PercMLP | 0.625 | 1 | 1.15 | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất (1.538 m) nhưng đâm ít nhất (0.07 %) vì L2 đo mức giống **người lái trong log**, còn IDM là bộ phản hồi giữ khoảng cách an toàn: nó giữ thời khoảng 1.2 s trong khi người lái giữ trung vị 1.87 s, nên quỹ đạo khác người nhưng vẫn an toàn, và nó sửa lệnh mỗi 0.1 s theo khoảng cách thật. EgoMLP có L2 0.765 tốt hơn const-vel 1.266 vì tốc độ của người lái mượt và dễ ngoại suy từ lịch sử 2 s của chính nó, nhưng nó **không nhìn thấy khoảng cách và tốc độ xe dẫn**, nên khi xe dẫn phanh nó không biết mà phanh theo: trên các quỹ đạo có xe dẫn phanh nó đâm 56.8 % (const-vel 57.3 %), tổng thể 24.9 % cao hơn cả const-vel 20.9 %. Lát cắt "xe dẫn sắp phanh" chỉ chiếm 4.5 % cửa sổ nhưng L2 trên lát cắt lớn hơn trung bình (EgoMLP 1.109 so với 0.765), nên trung bình che lỗi.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm của EgoMLP qua 5 seed là 23.4, 26.2, 25.6, 24.9, 24.3 %, tức dao động **23.4 .. 26.2 %** (2.8 điểm). Khoảng Wilson của seed tốt nhất (7770) là [21.9, 25.0] % và của seed xấu nhất (7771) là [24.6, 27.8] %, hai khoảng **chồng nhau** một phần: chênh lệch giữa các seed gần bằng nhiễu. Với PercMLP dải rộng hơn nhiều: **0.45 .. 2.15 %** (gấp khoảng 4.8 lần), khoảng Wilson của seed tốt nhất (7773) là [0.26, 0.77] % và của seed xấu nhất (7771) là [1.68, 2.75] %, **không chồng nhau**. Nếu tôi chỉ chạy seed 7773 hoặc 7772 (0.62 %, cận trên 0.98 %), PercMLP sẽ có vẻ đạt quy tắc R2 (ngưỡng 1.07 %) và gần bằng IDM; chọn seed 7771 thì kết luận ngược lại (cận trên 2.75 %). Báo một lần chạy là nguy hiểm vì có thể công bố "mô hình đã an toàn" chỉ nhờ seed may mắn; đó cũng là lý do cổng R2 lấy seed xấu nhất và R5 đòi dải qua seed không quá 1 điểm (PercMLP 1.7 điểm nên rớt R5).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng ở độ trễ 0.6 s, tỷ lệ lưới có va chạm bất kỳ (trước hoặc sau):

| Bộ điều khiển | Xe sau phát lại (%) | Xe sau phản ứng (%) | Va chạm phía sau: phát lại / phản ứng (%) |
|---|---|---|---|
| const-vel | 77.6 | 77.6 | 0.0 / 0.0 |
| IDM | 75.7 | 7.4 | 68.3 / 0.0 |
| EgoMLP | 82.0 | 82.0 | 0.0 / 0.0 |
| PercMLP | 77.7 | 24.2 | 56.7 / 0.0 |
| PercMLP+ | 74.8 | 10.5 | 67.2 / 0.0 |

`T` hiệu chỉnh là **1.4 s** (khoảng cách phân bố thời khoảng W1 = 0.357, T = 1.2 gần bằng với 0.360), trung vị thời khoảng của người lái trong log là 1.87 s. Xe sau tự lái như vậy chân thực hơn rất nhiều so với xe không bao giờ phản ứng (W1 = 0.866). Khi xe sau chỉ phát lại, nó giữ tốc độ log bất kể xe thử vừa phanh, nên bất kỳ bộ điều khiển nào phanh đều bị tính "va chạm giả" từ phía sau: IDM bị **68.3 %** ô va chạm sau (và ở độ trễ 0 là 73.0 %), trong khi const-vel và EgoMLP không phanh nên 0.0 %. Với xe phản ứng, IDM chỉ còn 7.4 %, đúng bằng phần bất khả thi, và va chạm sau về 0.0 %. Kết luận đổi hẳn: trong thế giới phát lại IDM (75.7 %) gần như không phân biệt được với const-vel (77.6 %), còn trong thế giới phản ứng IDM tốt hơn const-vel mười lần (7.4 % so với 77.6 %).

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi R2 (2.15 %, cận trên 2.75 % so với ngưỡng 1.07 %), R3 (10.3 % trên lát cắt phanh, cận trên 15.5 % so với 6.1 %), R4 (16.7 % lưới so với 2.0 %), R5 (dải 0.5 .. 2.2 %, rộng hơn 1 điểm) và R6 (24.2 % so với 9.4 %); chỉ qua R1 và R7. Có hai nguyên nhân gốc. Thứ nhất, **kịch bản log không chứa** (xe cắt vào với tốc độ tiến gần lớn, phanh mạnh hơn mức 3.62 m/s² của người lái): thể hiện ở R4 và R6, khi lưới cắt vào cho 16.7 % ô "tránh được mà vẫn đâm" dù trên log chỉ đâm 1–2 %. Thứ hai, **sai số tích luỹ** khi mô hình tự lái, vì huấn luyện chỉ thấy trạng thái của người lái chứ không thấy trạng thái do chính nó dẫn tới: thể hiện ở R2, R3, R5, khi L2 vòng hở tốt nhất (0.625 m) mà vòng kín đâm gấp khoảng 16 lần IDM (1.15 % so với 0.07 %). Tôi sửa bằng hai cách: thêm 197 130 cửa sổ từ 20 000 ca cắt vào mô phỏng (gắn nhãn bằng IDM) và 2 vòng DAgger (tự lái trên log huấn luyện, gắn nhãn bằng dự báo 3 s của IDM ở đúng trạng thái mô hình đã đi vào; tổng khoảng 895 000 cửa sổ). Kết quả cho PercMLP+: R2 qua (0.17 .. 0.24 %, cận trên 0.50 %), R3 qua (1.6 %, cận trên 4.7 %), R5 qua; nhưng **R4 vẫn rớt (3.0 % so với 2.0 %)** và **R6 vẫn rớt (10.5 % so với 9.4 %)**, nên cổng vẫn **BLOCK** (`fix.json`: `decision_PercMLP+ = false`). R4 rớt do một seed ngoại lệ (7774: 3.0 %, bốn seed còn lại 0.07 .. 0.37 %), tức vẫn còn bất ổn qua seed. Ở tốc độ cruise 20 m/s mà huấn luyện chưa từng thấy (≤ 16), PercMLP ngoại suy kém (20.1 .. 32.3 %) còn PercMLP+ chỉ 0.6 .. 1.9 % (IDM 0.0 %). PercMLP+ **không tốt hơn** IDM: trên log nó đâm 0.24 % so với 0.07 % của IDM và khoảng tin cậy chồng nhau; trên lưới cắt vào nó kém hơn IDM; nó học từ nhãn của IDM nên không có lý do gì vượt thầy, tốt nhất là xấp xỉ. Không nên phát hành, cũng không nên thay IDM. Tôi đề xuất điều tra seed 7774, chạy thêm seed và thêm vòng DAgger, thử trên thế giới phản ứng khác, rồi chạy lại cổng với ngưỡng giữ nguyên. Một điểm đáng chú ý: L2 vòng hở của PercMLP+ **tăng** từ 0.625 lên 0.823 m trong khi va chạm vòng kín giảm từ 1.15 % xuống 0.21 %, tức điểm vòng hở phạt đúng hành vi an toàn hơn vì nó không còn bắt chước người lái.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Thế giới 1 chiều, không cảm biến**: bộ điều khiển nhận khoảng cách và tốc độ xe dẫn thật; nhiễu chỉ thử một mức (±1 m, ±0.5 m/s) và PercMLP vẫn dao động 0.4 .. 2.0 % ở mức đó. Không có làn, người đi bộ, che khuất hay lỗi cảm biến thật, nên "qua cổng" không nói gì về hệ thống đầy đủ. (2) **Xe dẫn trong log không phản ứng với xe thử, và xe sau là mô hình IDM hiệu chỉnh** (T = 1.4 s, trễ 0.6 s): kết luận ở Câu 6 phụ thuộc hoàn toàn vào giả định này, vì IDM đổi từ 75.7 % sang 7.4 % chỉ nhờ đổi loại xe sau. Người lái thật có thể phản ứng khác hẳn. (3) **Dữ liệu hẹp và số mẫu nhỏ ở phần quan trọng**: chỉ một bộ dữ liệu (Argoverse), phanh nhẹ (giảm tốc p99 chỉ −4.21 m/s², người lái ≤ 3.62 m/s²), một tốc độ cruise 12 m/s cho lưới, và chỉ 185 quỹ đạo có xe dẫn phanh trong 2883 quỹ đạo test, nên IDM có 2/185 ca đâm với khoảng tin cậy rộng [0.3, 3.9] %. (4) **Một cách chia dữ liệu và 5 seed**: dải qua seed (PercMLP 0.45 .. 2.15 %) cho thấy ngay 5 seed vẫn chưa đủ để chắc về đuôi phân bố. (5) **Ngưỡng của cổng là lựa chọn của đội**, không phải chân lý (ví dụ +1 điểm, +2 điểm); và mô phỏng bỏ qua độ trễ phần cứng, thời gian tính toán thật (đã thấy 585 khung trễ ở chế độ thời gian thật) và độ ma sát. Cần làm tiếp để thu hẹp khoảng cách: thu thập log chủ động có phanh gấp và cắt vào, chạy shadow mode trên xe thật so với IDM, thử với nhiều bộ dữ liệu và cảm biến thật, thêm nhiều seed và nhiều hành vi xe khác, rồi mới quyết định phát hành từng bước với giám sát.
