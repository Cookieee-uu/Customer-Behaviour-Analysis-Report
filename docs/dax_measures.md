# DAX – Cột tính toán và Measure | KPIM Mart RFM

Toàn bộ công thức dưới đây được trích từ semantic model của file Power BI **Capstone Project PBI Cơ bản**. Các giá trị kiểm chứng (ghi ở cột "Kết quả") được chạy trực tiếp trên mô hình với toàn bộ bộ lọc ở trạng thái mặc định.

Mục lục:
1. Logic RFM tổng quan
2. Cột tính toán RFM (trong `dim_customer`)
3. Bảng tra phân khúc (`map_rfm_code`)
4. Measure theo thư mục
5. Lưu ý kỹ thuật

---

## 1. Logic RFM tổng quan

| Bước | Nội dung |
| --- | --- |
| 1 | Với mỗi khách hàng tính Recency (số ngày kể từ lần mua gần nhất), Frequency (số đơn riêng biệt), Monetary (tổng doanh thu) |
| 2 | Chấm điểm 1–5 cho từng tiêu chí (R và M theo ngũ phân vị, F theo số đơn, tối đa 5) |
| 3 | Ghép thành mã `R-F-M` rồi tra bảng 125 tổ hợp để ra 1 trong 11 phân khúc |
| 4 | Gộp 11 phân khúc thành 6 nhóm quản trị; khách chưa mua hàng tách riêng |
| 5 | Các measure RFM lấy trung bình điểm R, F, M theo ngữ cảnh lọc rồi tính Average Score |

Ngày chốt: ngày mua lớn nhất trong `fact_sales` (31/12/2022).

---

## 2. Cột tính toán RFM (trong `dim_customer`)

### Recency, Frequency, Monetary

```dax
lastdatepurchase = CALCULATE(MAX(fact_sales[order_date]))

recency = DATEDIFF(dim_customer[lastdatepurchase].[Date], MAX(fact_sales[order_date]), DAY)

frequency = CALCULATE(DISTINCTCOUNT(fact_sales[order_number]))

monetary = CALCULATE(SUM(fact_sales[sales_amount]))

date_first_purchase = CALCULATE(MIN(fact_sales[order_date]))
```

- Trong cột tính toán, `CALCULATE` chuyển ngữ cảnh dòng của khách hàng thành bộ lọc lên `fact_sales`, nên mỗi khách có giá trị riêng.
- Trong `recency`, `MAX(fact_sales[order_date])` không có `CALCULATE` nên lấy ngày lớn nhất của toàn bộ dữ liệu, đóng vai trò ngày chốt.
- Khách chưa có giao dịch có `recency`, `frequency`, `monetary` rỗng (BLANK).

### Điểm R – ngũ phân vị, đảo chiều

```dax
score_recency =
VAR p20 = PERCENTILE.EXC(dim_customer[recency], 0.20)
VAR p40 = PERCENTILE.EXC(dim_customer[recency], 0.40)
VAR p60 = PERCENTILE.EXC(dim_customer[recency], 0.60)
VAR p80 = PERCENTILE.EXC(dim_customer[recency], 0.80)
RETURN
IF(ISBLANK(dim_customer[recency]), BLANK(),
IF(dim_customer[recency] >= p80, 1,
IF(dim_customer[recency] >= p60, 2,
IF(dim_customer[recency] >= p40, 3,
IF(dim_customer[recency] >= p20, 4, 5)))))
```

Số ngày càng lớn (lâu không mua) thì điểm càng thấp.

### Điểm F – theo số đơn, giới hạn 5

```dax
score_frequency =
IF(ISBLANK(dim_customer[frequency]), BLANK(), MIN(dim_customer[frequency], 5))
```

Khách có từ 5 đơn trở lên đều nhận 5 điểm (số đơn lớn nhất trong dữ liệu là 28). Cách chấm này **khác** mô tả "ngũ phân vị cho cả ba tiêu chí" trong tài liệu gốc.

### Điểm M – ngũ phân vị

```dax
score_monetary =
VAR p20 = PERCENTILE.EXC(dim_customer[monetary], 0.20)
VAR p40 = PERCENTILE.EXC(dim_customer[monetary], 0.40)
VAR p60 = PERCENTILE.EXC(dim_customer[monetary], 0.60)
VAR p80 = PERCENTILE.EXC(dim_customer[monetary], 0.80)
RETURN
IF(ISBLANK(dim_customer[monetary]), BLANK(),
IF(dim_customer[monetary] >= p80, 5,
IF(dim_customer[monetary] >= p60, 4,
IF(dim_customer[monetary] >= p40, 3,
IF(dim_customer[monetary] >= p20, 2, 1)))))
```

### Mã R-F-M và phân khúc

```dax
score r-f-m = dim_customer[score_recency] & "-" & dim_customer[score_frequency] & "-" & dim_customer[score_monetary]

customer_segment = COALESCE(RELATED(map_rfm_code[segment]), "Chưa phân loại")
```

Khách chưa mua hàng có mã `--`, khớp dòng đặc biệt trong `map_rfm_code` và nhận phân khúc `Chưa mua hàng`. Giá trị dự phòng `Chưa phân loại` hiện không xuất hiện trong dữ liệu.

### Gộp 6 nhóm quản trị

```dax
segment(group) =
IF(dim_customer[customer_segment] IN {"Loyal Customers","Champions"}, "KH có giá trị",
IF(dim_customer[customer_segment] IN {"Promising","Potential Loyalists","New Customers"}, "KH Tiềm Năng",
IF(dim_customer[customer_segment] IN {"Need Attention"}, "KH thông thường",
IF(dim_customer[customer_segment] IN {"At Risk","Can't Lose"}, "KH cũ có giá trị cao",
IF(dim_customer[customer_segment] IN {"About To Sleep"}, "KH không hoạt động",
IF(dim_customer[customer_segment] IN {"Hibernating","Lost"}, "KH ngủ đông",
"Không Phân Loại"))))))
```

| Nhóm quản trị | Phân khúc RFM chi tiết |
| --- | --- |
| KH có giá trị | Champions, Loyal Customers |
| KH Tiềm Năng | Promising, Potential Loyalists, New Customers |
| KH thông thường | Need Attention |
| KH cũ có giá trị cao | At Risk, Can't Lose |
| KH không hoạt động | About To Sleep |
| KH ngủ đông | Hibernating, Lost |
| Không Phân Loại | Chưa mua hàng (120 khách) |

---

## 3. Bảng tra phân khúc (`map_rfm_code`)

Bảng tính toán: `CROSSJOIN` ba dãy `GENERATESERIES(1, 5, 1)` tạo 125 tổ hợp, mỗi tổ hợp được gán phân khúc bằng `SWITCH(TRUE(), ...)`, cộng thêm một dòng `--` → `Chưa mua hàng`.

Danh sách mã (viết dạng RFM liền nhau) của từng phân khúc:

| Phân khúc | Các mã R-F-M |
| --- | --- |
| Champions | 555, 554, 544, 545, 454, 455, 445 |
| Loyal Customers | 543, 444, 435, 355, 354, 345, 344, 335 |
| Potential Loyalists | 553, 551, 552, 541, 542, 533, 532, 531, 452, 451, 442, 441, 431, 453, 433, 432, 423, 353, 352, 351, 342, 341, 333, 323 |
| New Customers | 512, 511, 422, 421, 412, 411, 311 |
| Promising | 525, 524, 523, 522, 521, 515, 514, 513, 425, 424, 413, 414, 415, 315, 314, 313 |
| Need Attention | 535, 534, 443, 434, 343, 334, 325, 324 |
| About To Sleep | 331, 321, 312, 221, 213 |
| At Risk | 255, 254, 245, 244, 253, 252, 243, 242, 235, 234, 225, 224, 153, 152, 145, 143, 142, 135, 134, 133, 125, 124 |
| Can't Lose | 155, 154, 144, 214, 215, 115, 114, 113 |
| Hibernating | 332, 322, 231, 241, 251, 233, 232, 223, 222, 132, 123, 122, 212, 211 |
| Lost | 111, 112, 121, 131, 141, 151 |

Mô tả đặc điểm và gợi ý hành động của từng phân khúc nằm trong bảng `dim_rfm_segment`.

---

## 4. Measure

Tất cả measure nằm trong bảng `_Measures`.

### 4.1 Thư mục "Doanh thu"

| Measure | Công thức DAX | Ý nghĩa |
| --- | --- | --- |
| Doanh thu | `SUM(fact_sales[sales_amount])` | Tổng doanh thu. Kết quả toàn mô hình: 17.912.095.075 ₫ |
| Doanh thu cùng kỳ năm trước | `CALCULATE([Doanh thu], SAMEPERIODLASTYEAR(dim_date[date]))` | Doanh thu cùng kỳ năm trước, dùng để so sánh |
| Tăng trưởng doanh thu | `IF(ISBLANK([Doanh thu cùng kỳ năm trước]), BLANK(), [Doanh thu] - [Doanh thu cùng kỳ năm trước])` | Chênh lệch tuyệt đối so với cùng kỳ; rỗng khi không có kỳ so sánh |
| Tỷ lệ tăng trưởng doanh thu | `DIVIDE([Tăng trưởng doanh thu], [Doanh thu cùng kỳ năm trước])` | Tăng trưởng % so với cùng kỳ |
| Giá trị trung bình mỗi đơn | `DIVIDE([Doanh thu], [Số đơn hàng])` | Giá trị hóa đơn trung bình |

### 4.2 Thư mục "Khách hàng"

| Measure | Công thức DAX | Ý nghĩa |
| --- | --- | --- |
| Số khách hàng | `DISTINCTCOUNT(dim_customer[customer_key])` | Số khách trong danh sách (kể cả chưa mua). Toàn mô hình: 18.484 |
| Khách hàng có giao dịch | `DISTINCTCOUNT(fact_sales[customer_key])` | Số khách đã phát sinh giao dịch. Toàn mô hình: 18.364 |
| Số đơn hàng | `DISTINCTCOUNT(fact_sales[order_number])` | Số đơn hàng riêng biệt. Toàn mô hình: 27.345 |
| Số đơn hàng bình quân mỗi khách mua | `DIVIDE([Số đơn hàng], [Khách hàng có giao dịch])` | Tần suất mua trung bình |
| Doanh thu bình quân mỗi khách mua | `DIVIDE([Doanh thu], [Khách hàng có giao dịch])` | Giá trị trung bình mỗi khách có mua |
| Doanh thu bình quân mỗi đơn hàng | `DIVIDE([Doanh thu], [Số đơn hàng])` | Tương đương "Giá trị trung bình mỗi đơn"; hai measure có công thức trùng nhau |

### 4.3 Thư mục "RFM" (nguồn của 4 Gauge)

| Measure | Công thức DAX | Kết quả |
| --- | --- | --- |
| RECENCY SCORE | `AVERAGE(dim_customer[score_recency])` | 2,99 |
| FREQUENCY SCORE | `AVERAGE(dim_customer[score_frequency])` | 1,46 |
| MONETARY SCORE | `AVERAGE(dim_customer[score_monetary])` | 3,00 |
| AVERAGE SCORE | `DIVIDE([RECENCY SCORE] + [FREQUENCY SCORE] + [MONETARY SCORE], 3)` | 2,48 |
| TARGET SCORE | `4` | Mốc tham chiếu 4,0 trên thang 1–5 |

Các measure điểm tính trung bình trên các khách hàng đang nằm trong bộ lọc (thẻ, thu nhập, tuổi, nghề nghiệp, thời gian) nên thay đổi theo slicer.

### 4.4 Thư mục "Kế hoạch"

| Measure | Công thức DAX | Ý nghĩa |
| --- | --- | --- |
| Doanh thu kế hoạch | `IF(ISFILTERED(dim_product), BLANK(), SUM(fact_sales_plan[sales_plan]))` | Kế hoạch doanh thu. Trả về rỗng khi báo cáo đang lọc theo sản phẩm vì kế hoạch không có chiều sản phẩm |
| Chênh lệch so với kế hoạch | `[Doanh thu] - [Doanh thu kế hoạch]` | Thực tế trừ kế hoạch |
| Tỷ lệ đạt kế hoạch | `DIVIDE([Doanh thu], [Doanh thu kế hoạch])` | Thực tế chia kế hoạch |

### 4.5 Thư mục "Tooltips" (ẩn)

| Measure | Công thức DAX | Ý nghĩa |
| --- | --- | --- |
| % Doanh thu theo nhóm | `DIVIDE([Doanh thu], CALCULATE([Doanh thu], REMOVEFILTERS(dim_customer[segment(group)])))` | Tỷ trọng doanh thu của nhóm trên tổng |
| % KH theo nhóm | `DIVIDE([Số khách hàng], CALCULATE([Số khách hàng], REMOVEFILTERS(dim_customer[segment(group)]), dim_customer[customer_segment] <> "Chưa mua hàng"))` | Tỷ trọng số khách của nhóm trên tổng khách **đã mua hàng** |
| Nhận xét nhóm | `"Chiếm " & FORMAT([% KH theo nhóm], "0.0%") & " số khách, đóng góp " & FORMAT([% Doanh thu theo nhóm], "0.0%") & " doanh thu"` | Câu mô tả hiển thị trên tooltip |
| Gồm phân khúc | `CONCATENATEX(VALUES(dim_customer[customer_segment]), dim_customer[customer_segment], ", ", dim_customer[customer_segment], ASC)` | Liệt kê các phân khúc chi tiết thuộc nhóm |
| Subtitle | `"As of" & " " & MAX(dim_date[date])` | Dòng "As of" hiển thị ngày dữ liệu mới nhất trên dashboard |

---

## 5. Lưu ý kỹ thuật

1. **Điểm F không dùng phân vị.** `score_frequency = MIN(frequency, 5)`; trong khi tài liệu gốc mô tả cả ba tiêu chí dùng phân vị. README nên mô tả theo đúng mô hình.
2. **Mô tả của `TARGET SCORE`** trong mô hình ghi "thang 1–4", trong khi điểm R, F, M chạy 1–5 và gauge dùng mốc 4,0 trên thang 1–5. Nên sửa lại mô tả cho khớp.
3. **Hai measure trùng công thức:** `Giá trị trung bình mỗi đơn` và `Doanh thu bình quân mỗi đơn hàng` cùng bằng `DIVIDE([Doanh thu], [Số đơn hàng])`. Có thể giữ một.
4. **`Tăng trưởng doanh thu` và `SAMEPERIODLASTYEAR`** cần cột ngày liên tục trên trục thời gian (dùng `dim_date[date]`) để cho kết quả đúng.
5. **Tuổi và nhóm tuổi** tính từ ngày làm mới dữ liệu, không cố định theo ngày chốt RFM.
6. **Giá trị PERCENTILE.EXC** được tính trên toàn bộ khách hàng, không đổi theo slicer vì là cột tính toán (tính lúc làm mới dữ liệu). Chỉ các measure điểm trung bình thay đổi theo bộ lọc.
