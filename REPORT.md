# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Đăng Nam Khánh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602741 |
| GitHub | KhanhNamYeh |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 9012 / 1545 (5 seed huấn luyện 1545..1549), tốc độ cruise lưới 10 m/s |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.267 | 20.6 | BLOCK (R2, R3, R4, R6) |
| IDM | 1.504 | 0.03 (CI95 0.0–0.2) | (baseline) |
| EgoMLP | 0.764..0.768 | 21.7..34.4 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.618..0.624 | 0.3..1.7 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.799 (±0.006) | 0.1..0.3 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Dữ liệu thô có 9993 quỹ đạo (992679 dòng). Tôi thấy ba vấn đề:
1. **Cột `Acc_LV` không dùng được.** Nó gần như không tương quan với đạo hàm của `Speed_LV` (hệ số 0.045). Cột gia tốc tương ứng của xe theo sau thì khớp tốt (0.929). Vì vậy tôi bỏ `Acc_LV` và dùng gia tốc xe dẫn suy ra từ `Speed_LV` đã làm mượt (`a_lead`). Với gia tốc này, phân vị p1 là −1.95 m/s² và p0.1 là −2.77 m/s².
2. **Có quỹ đạo trùng.** 383 quỹ đạo trùng hệt nhau nằm ở các id cách xa nhau. Tôi loại chúng, còn lại 9610 quỹ đạo. Nếu giữ lại, cùng một quỹ đạo có thể nằm ở cả train lẫn test, và điểm test sẽ đẹp giả do rò rỉ.
3. **Các khung trong một quỹ đạo gần như giống nhau.** Dữ liệu ghi ở 10 Hz, mỗi quỹ đạo 70–109 bước. Nếu chia theo khung hình, các khung gần như giống hệt nhau của cùng một chuyến đi sẽ rơi vào cả hai tập. Chỉ khi chia theo **quỹ đạo** (seed chia 9012) mới đo được mô hình có tổng quát được sang chuyến đi chưa thấy hay không.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Log gồm 600 đoạn.
- **Chạy lại cùng mã:** 0/600 đoạn khác, và 0 đoạn không giống từng bit.
- **Bản refactor float32** (đổi thứ tự phép tính): cả 600/600 đoạn không giống từng bit, nhưng 0/600 đoạn vượt dung sai. Sai khác lớn nhất là 9.7e-05, nhỏ hơn atol 1e-3 + rtol 1e-3.
  - Phép cộng số thực không có tính kết hợp: (0.1+0.2)+0.3 = 0.6000000000000001, còn 0.1+(0.2+0.3) = 0.6. Vì vậy so từng bit là phép thử sai.
  - Dung sai cỡ 1e-3 m/s² là hợp lý, vì nó nhỏ hơn nhiều so với mức thay đổi lệnh ga/phanh có ý nghĩa.
- **Thêm AEB** (phanh −6 m/s² khi TTC < 1.5 s) làm lệch 16/600 đoạn.
  - Đoạn 47 chỉ lệch 4/106 khung, là các khung 98–101. Ở đó gap giảm từ 4.30 xuống 3.62 m, tốc độ tiến gần 2.4–2.9 m/s, TTC 1.43–1.49 s, và lệnh đổi từ −5.44 thành −6.0.
  - Đoạn 66 chỉ lệch đúng khung 82 (TTC 1.47 s).
  - AEB chỉ kích hoạt ở các khung có TTC < 1.5 s, nên hai bản chỉ khác nhau ở đúng các khung đó.
- **Phát nhanh nhất so với thời gian thật:** phát nhanh nhất có 0 khung trễ và 0 khung rớt, vì ở chế độ đó không có hạn chót. Phát theo thời gian thật lộ ra 582 khung trễ và 576 khung rớt trên 59764 khung, làm lệnh đổi ở 371/600 đoạn. Lỗi trễ hạn chỉ thấy được khi phát lại đúng nhịp thời gian.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

**Lưới cắt vào:** tốc độ cruise 10 m/s, khoảng cách 2.76–59.3 m (p2..p98 của log), tốc độ tiến gần 0–10 m/s, tổng cộng 60 × 45 = 2700 ô.

**Vùng bất khả thi** (không ai tránh được, kể cả phanh hết cỡ 9 m/s²):
- Với trễ 0 s: 1.0 % ô.
- Với trễ 0.6 s: tăng lên 5.1 % ô.

**Kiểm chứng bộ mô phỏng:** phanh hết cỡ đâm đúng 1.0 % và 5.1 % ô, và có 0 ô lệch xa biên ở cả hai độ trễ.

**IDM:** vùng đâm tăng từ 1.0 % lên 5.1 % khi trễ đổi từ 0 sang 0.6 s. Vùng này trùng khít vùng bất khả thi: phần "tránh được mà vẫn đâm" bằng 0.0 %. Nói cách khác, IDM chỉ đâm ở nơi ai cũng đâm. Biên bất khả thi chính là thứ giúp tách "không thể tránh" khỏi "bộ điều khiển thất bại".

**Độ phủ của log:** log có 614 ca xe dẫn phanh (6.4 % quỹ đạo), phanh đỉnh p50 −2.5 m/s² và p99 −4.2 m/s². Người lái trong log phanh mạnh nhất cỡ 3.6 m/s² (p99.9), nhưng 5.2 % lưới đòi phanh mạnh hơn mức đó. Vậy các ca khai thác từ log không phủ vùng khó: phanh gấp, khoảng cách ngắn, tiến gần nhanh.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 (m) | Hạng L2 | Va chạm (%) | Hạng va chạm |
|---|---|---|---|---|
| const-vel | 1.267 | 3 | 20.6 | 3 |
| IDM | 1.504 | 4 | 0.0 | 1 |
| EgoMLP | 0.766 | 2 | 27.5 | 4 |
| PercMLP | 0.622 | 1 | 1.0 | 2 |

**Các cặp đổi thứ tự:** (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP).

**IDM: L2 tệ nhất nhưng lái tốt nhất.** L2 của IDM là 1.504 m, tệ nhất bảng, vì nó không bắt chước người lái: nó giữ thời khoảng riêng 1.2 s, nên dự báo vị trí của nó lệch khỏi quỹ đạo người. Nhưng khi tự lái, nó luôn nhìn khoảng cách và tốc độ tiến gần rồi phanh theo. Vì vậy nó chỉ đâm 0.03 % trên log và 0.5 % trên lát cắt xe dẫn phanh.

**EgoMLP: L2 tốt hơn const-vel nhưng vẫn đâm.** EgoMLP chỉ nhìn tốc độ của chính nó. Trên log, người lái thường đi đều, nên đoán "cứ đi tiếp như cũ" đã cho L2 0.766 m, tốt hơn const-vel (1.267 m). Nhưng nó không biết phía trước có xe, nên khi xe dẫn phanh nó không phản ứng. Kết quả là nó đâm 21.7–34.4 % trên log và 64.0 % trên lát cắt phanh.

**Trung bình che mất tình huống nguy hiểm.** Lát cắt "xe dẫn sắp phanh" chỉ chiếm 4.7 % số cửa sổ, và L2 trên lát cắt cao hơn hẳn (EgoMLP 1.113 m so với 0.766 m toàn bộ). Lấy trung bình trên mọi cửa sổ thì phần lớn sai số ở đúng lát cắt này bị pha loãng.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

**Dao động của EgoMLP qua 5 seed:** tỷ lệ va chạm là 21.7 %, 24.2 %, 28.1 %, 29.0 % và 34.4 %, tức dải rộng 12.7 điểm. Trong khi đó L2 vòng hở gần như không đổi (0.764..0.768 m).

**Khoảng Wilson nói gì:**
- Seed xấu nhất có khoảng tin cậy Wilson 95 % là [32.7, 36.1] %. Cận dưới 32.7 % nằm hẳn trên seed tốt nhất (21.7 %). Vậy khác biệt giữa các seed không phải nhiễu lấy mẫu trên 2883 quỹ đạo test, mà là khác biệt thật giữa các mô hình được huấn luyện.
- Khoảng Wilson chỉ đo sai số do số quỹ đạo hữu hạn. Nó không đo dao động do seed huấn luyện.

**Nếu chỉ chạy một seed:**
- Với EgoMLP, tôi có thể báo 21.7 % hoặc 34.4 %, tuỳ may rủi.
- PercMLP cũng vậy, dao động 0.3..1.7 %. Nếu chỉ chạy seed tốt (0.3 %), dễ kết luận nó "gần bằng IDM". Nhưng seed xấu nhất 1.7 % (CI [1.3, 2.3]) đã rớt R2.

Vì vậy cổng chấm theo seed xấu nhất, và R5 đòi dải qua các seed không quá 1 điểm.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

**Hiệu chỉnh xe sau:** người lái trong log giữ thời khoảng trung vị 1.86 s. Tôi hiệu chỉnh xe sau IDM theo khoảng cách Wasserstein giữa phân bố thời khoảng của nó và của người lái.
- Chọn T = 1.4 s, khoảng cách 0.36. T = 1.2 s cũng cho 0.36, còn T = 2.4 s cho 0.88.
- Một xe không bao giờ phản ứng cho 0.86.

Vậy xe sau phản ứng chân thực hơn rõ rệt so với xe chỉ phát lại.

Bảng với trễ 0.6 s:

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 73.4 % | 73.4 % | 0.0 / 0.0 % |
| IDM | 70.1 % | 5.1 % | 64.9 / 0.0 % |
| EgoMLP | 73.4 % | 73.4 % | 0.0 / 0.0 % |
| PercMLP | 73.2 % | 20.5 % | 53.8 / 0.0 % |

**Va chạm giả với xe sau phát lại:** IDM bị tính va chạm ở 70.1 % lưới, trong đó 64.9 % là va chạm phía sau. IDM phanh đúng khi bị cắt vào, nhưng xe sau phát lại theo log không biết IDM vừa phanh nên đâm vào. Đó là "va chạm giả".

**Kết luận đổi ra sao:**
- Trong thế giới xe sau phát lại, IDM (70.1 %) trông gần như tệ bằng const-vel (73.4 %), một bộ điều khiển không hề phanh.
- Khi xe sau phản ứng, va chạm phía sau về 0.0 % và IDM chỉ còn 5.1 %, đúng bằng vùng bất khả thi.

Thiếu tác nhân phản ứng, phép đo phạt chính hành vi an toàn và dẫn tới kết luận sai về bộ điều khiển.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi R2, R3, R4, R5 và R6; nó chỉ qua R1 và R7. Các quy tắc rớt quy về hai nguyên nhân gốc:
- **(a) Kịch bản mà log không chứa** (thấy ở R4 và R6). Trên lưới cắt vào với trễ 0.6 s, PercMLP "tránh được mà vẫn đâm" ở 15.3 % lưới (baseline 0.0 %), và 20.5 % trong thế giới có xe sau phản ứng (baseline 5.1 %). Log không có ca cắt vào gắt, và người lái chỉ phanh tới 3.6 m/s². Ở tốc độ cruise 20 m/s mà mô hình chưa từng thấy, PercMLP đâm 23.3..32.9 % lưới.
- **(b) Sai số tích luỹ khi tự lái** (thấy ở R2, R3, R5). Ngay trên log, seed xấu nhất có cận trên 95 % là 2.28 %, vượt ngưỡng 1.03 % (R2). Lát cắt phanh đâm 4.3 % (R3), và dải qua các seed là 0.3..1.7 % (R5). Mô hình chỉ học các trạng thái người lái đã đi qua, nên một sai số nhỏ đưa nó vào trạng thái lạ, rồi sai số cứ thế trôi tiếp.

**Cách sửa, mỗi cách nhắm đúng một nguyên nhân:**
- Chữa (a): thêm 197410 cửa sổ huấn luyện từ 20000 ca cắt vào mô phỏng, gắn nhãn bằng hành vi của IDM.
- Chữa (b): chạy 2 vòng DAgger, tức gắn nhãn IDM ở chính các trạng thái mà mô hình tự đi vào khi lái.

**Kết quả sau khi sửa:** PercMLP+ qua cả 7 quy tắc.
- Va chạm trên log 0.1..0.3 % (cận trên 0.55 %).
- Lưới cắt vào 0.3 %.
- Thế giới phản ứng 5.4 %, so với baseline 5.1 %.
- Ở 20 m/s còn 0.7..2.9 %.

**PercMLP+ có tốt hơn IDM không?** Không. Cổng chỉ chứng minh nó "không kém hơn quá biên cho phép". IDM vẫn ít va chạm hơn (0.03 % so với 0.1..0.3 %), và PercMLP+ học từ chính IDM nên khó vượt được thầy của nó. L2 vòng hở còn xấu đi (0.622 → 0.799 m), cho thấy điểm vòng hở không đo đúng thứ ta quan tâm.

**Đề xuất:** chưa thay IDM ngay. Trước hết cho PercMLP+ chạy song song ở chế độ shadow, thử thêm kịch bản ngoài phân phối và dữ liệu thật, rồi mới cân nhắc phát hành.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Chỉ có một chiều dọc, không có cảm biến.** Không có chuyển làn, xe vượt, người đi bộ, và không có nhiễu nhận thức thật; lab chỉ thử nhiễu gap ±1 m và tốc độ ±0.5 m/s. PercMLP+ qua cổng trên 2700 ô lưới, nhưng thế giới thật có nhiều chiều hơn hẳn, nên "qua cổng" không có nghĩa là lái an toàn.
2. **Xe dẫn trong log không phản ứng với xe thử.** Mọi tỷ lệ va chạm trên log (ví dụ 0.1..0.3 %) được tính trong một thế giới mà xe dẫn không đổi hành vi khi ta lái khác người thật. Chỉ xe sau được làm cho phản ứng (T = 1.4 s), và chính tác nhân đó cũng là IDM do ta tự giả định.
3. **Mô phỏng là giả định của chúng ta.** Phanh tối đa 9 m/s², độ trễ 0 hoặc 0.6 s, vật lý lý tưởng. Vùng "bất khả thi" 5.1 % và mọi kết luận "không kém hơn" chỉ đúng bên trong mô hình đó.
4. **Dữ liệu hẹp.** Dữ liệu chỉ từ Argoverse 2: người lái phanh tối đa 3.6 m/s², tốc độ p99 khoảng 17.8 m/s. Ngoài phân phối (20 m/s), PercMLP+ vẫn đâm 0.7..2.9 % lưới trong khi IDM đâm 0.0 %.
5. **Mẫu nhỏ.** Chỉ có 5 seed và 2883 quỹ đạo test, nên không đo được các sự kiện hiếm (cỡ 1/10000).

**Cần làm tiếp:**
- Mô phỏng nhiều làn với tác nhân phản ứng được hiệu chỉnh từ dữ liệu.
- Đưa lỗi cảm biến thật vào.
- Thử ở tốc độ và thời tiết ngoài phân phối.
- Chạy shadow trên xe thật.
- Chạy nhiều seed và nhiều kịch bản hơn để thu hẹp khoảng tin cậy.
