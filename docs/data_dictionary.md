# Data Dictionary – KPIM Mart | Phân tích khách hàng theo RFM

Tài liệu mô tả mô hình dữ liệu của file Power BI **Capstone Project PBI Cơ bản**, được trích trực tiếp từ semantic model (tên bảng, tên cột, kiểu dữ liệu, công thức cột tính toán, quan hệ).

- Nguồn: cơ sở dữ liệu SQL Server `kpim_mart` (dữ liệu của khóa học) và một file Excel kế hoạch doanh thu.
- Phạm vi giao dịch: 01/01/2020 – 31/12/2022. Ngày chốt RFM lấy theo ngày mua lớn nhất trong dữ liệu (31/12/2022).
- Chế độ lưu trữ: Import.

> Cột có dấu ✱ ở phần mô tả nghĩa là mô tả được suy ra từ tên cột, chưa có tài liệu nguồn xác nhận.

---

## 1. Tổng quan mô hình

Mô hình dạng sao (star schema) với `fact_sales` ở trung tâm, cộng thêm hai bảng tính toán phục vụ phân khúc RFM.

| Bảng | Loại | Số cột | Vai trò |
| --- | --- | --- | --- |
| `fact_sales` | Fact | 15 | Dòng giao dịch bán hàng (mức dòng hóa đơn) |
| `fact_sales_plan` | Fact | 4 | Kế hoạch doanh thu theo cửa hàng và tháng |
| `dim_customer` | Dimension | 26 | Hồ sơ khách hàng + các cột tính RFM |
| `dim_product` | Dimension | 11 | Sản phẩm |
| `dim_store` | Dimension | 10 | Cửa hàng |
| `dim_promotion` | Dimension | 8 | Chương trình khuyến mãi |
| `dim_date` | Dimension | 11 | Lịch (đánh dấu là bảng ngày) |
| `map_rfm_code` | Bảng tính toán (DAX) | 6 | Tra cứu 125 tổ hợp R-F-M → phân khúc |
| `dim_rfm_segment` | Bảng tính toán (DAX) | 4 | Mô tả và gợi ý hành động cho từng phân khúc |
| `_Measures` | Bảng chứa measure | 0 cột, 24 measure | Xem `dax_measures.md` |

Ngoài ra mô hình có các bảng `LocalDateTable_*` và `DateTableTemplate_*` do Power BI tự tạo (ẩn), không thuộc thiết kế phân tích.

### Quan hệ giữa các bảng

Tất cả quan hệ là nhiều-một (*→1), lọc một chiều.

| Từ (nhiều) | Đến (một) | Trạng thái |
| --- | --- | --- |
| `fact_sales[customer_key]` | `dim_customer[customer_key]` | Active |
| `fact_sales[product_key]` | `dim_product[product_key]` | Active |
| `fact_sales[store_key]` | `dim_store[store_key]` | Active |
| `fact_sales[promotion_key]` | `dim_promotion[promotion_key]` | Active |
| `fact_sales[order_date]` | `dim_date[date]` | Active |
| `fact_sales[ship_date]` | `dim_date[date]` | Inactive |
| `fact_sales[payment_date]` | `dim_date[date]` | Inactive |
| `fact_sales_plan[store]` | `dim_store[store]` | Active |
| `fact_sales_plan[date]` | `dim_date[date]` | Active |
| `dim_customer[score r-f-m]` | `map_rfm_code[R-F-M]` | Active |
| `map_rfm_code[segment]` | `dim_rfm_segment[segment]` | Active |

Ghi chú: hai quan hệ ngày giao hàng và ngày thanh toán không hoạt động; muốn dùng phải kích hoạt bằng `USERELATIONSHIP` trong measure.

---

## 2. `dim_customer` – Khách hàng

18.484 khách hàng, trong đó 18.364 khách có giao dịch và 120 khách chưa phát sinh giao dịch.

### 2.1 Cột lấy từ nguồn / Power Query

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `customer_key` | Số nguyên | Khóa khách hàng, liên kết với `fact_sales` |
| `loyal_group` | Văn bản | Hạng thẻ thành viên, đã chuyển chữ hoa. Giá trị trong dữ liệu: WHITE, BRONZE, SILVER, GOLD, BLACK, PLATINUM |
| `birthday` | Ngày | Ngày sinh |
| `gender` | Văn bản | Giới tính: Nam, Nữ |
| `marital_status` | Văn bản | Tình trạng hôn nhân: Đã kết hôn, Độc thân |
| `education` | Văn bản | Trình độ học vấn: Trung Học, Cao Đẳng, Đại Học, Thạc Sĩ, Tiến Sĩ |
| `occupation` | Văn bản | Nghề nghiệp: Chuyên viên tư vấn, Quản Lý, Kỹ thuật viên, Nhân viên văn phòng, Nghề nghiệp tự do |
| `yearly_income` | Số thực | Thu nhập hàng năm ✱ (các ngưỡng nhóm thu nhập tính theo đô la, xem `yearly_income_group`) |
| `total_children` | Số nguyên | Tổng số con ✱ |
| `is_active` | Đúng/Sai | Cờ khách hàng đang hoạt động ✱ |
| `full_name` | Văn bản | Họ tên, ghép từ `last_name` và `first_name` bằng dấu cách |
| `Age` | Số nguyên | Tuổi, tính từ `birthday` đến **ngày làm mới dữ liệu** (xem lưu ý ở mục 2.3) |
| `education_level` | Số nguyên | Mã hóa học vấn: Trung Học = 1, Cao Đẳng = 2, Đại Học = 3, Thạc Sĩ = 4, còn lại (Tiến Sĩ) = 5 |
| `age_group` | Văn bản | Nhóm tuổi: Dưới 18 tuổi (<18), Từ 18 - 25, Từ 26 - 35, Từ 36 - 50, Trên 50 tuổi (>50) |
| `yearly_income_group` | Văn bản | Nhóm thu nhập: $0 - $20k (≤20.000), $20k - $50k (≤50.000), $50k - $75k (≤75.000), $75k - $100k (≤100.000), Over $100k |

### 2.2 Cột tính toán (DAX) – nền tảng của mô hình RFM

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `lastdatepurchase` | Ngày | Ngày mua gần nhất của khách (`MAX(fact_sales[order_date])`) |
| `date_first_purchase` | Ngày | Ngày mua đầu tiên của khách (`MIN(fact_sales[order_date])`) |
| `recency` | Số nguyên | Số ngày từ lần mua gần nhất đến ngày mua lớn nhất trong dữ liệu |
| `frequency` | Số nguyên | Số đơn hàng riêng biệt của khách |
| `monetary` | Số thực | Tổng doanh thu (`sales_amount`) của khách |
| `score_recency` | Số nguyên 1–5 | Điểm R theo ngũ phân vị; mua càng gần đây điểm càng cao |
| `score_frequency` | Số nguyên 1–5 | Điểm F = số đơn hàng, giới hạn tối đa 5 (không dùng phân vị) |
| `score_monetary` | Số nguyên 1–5 | Điểm M theo ngũ phân vị; chi tiêu càng nhiều điểm càng cao |
| `score r-f-m` | Văn bản | Ghép điểm, ví dụ `5-1-3`. Khách chưa mua hàng có giá trị `--` |
| `customer_segment` | Văn bản | Một trong 11 phân khúc RFM chi tiết, hoặc `Chưa mua hàng` |
| `segment(group)` | Văn bản | 6 nhóm quản trị gộp từ 11 phân khúc (xem `dax_measures.md`); khách chưa mua hàng thuộc `Không Phân Loại` |

Công thức đầy đủ nằm ở `dax_measures.md`, mục "Cột tính toán RFM".

### 2.3 Lưu ý khi dùng dữ liệu

- `Age` dùng ngày hiện tại của máy khi làm mới dữ liệu nên giá trị thay đổi theo thời điểm refresh. Để kết quả lặp lại được, nên cố định một ngày tham chiếu (ví dụ 31/12/2022).
- Bảng này chứa họ tên và ngày sinh. Dữ liệu thuộc bộ dữ liệu khóa học, nhưng nên che cột `full_name` khi chụp ảnh bảng chi tiết khách hàng nếu chưa chắc dữ liệu là giả lập.
- Định dạng của `yearly_income` trong mô hình hiển thị ký hiệu ₫ trong khi các nhóm thu nhập dùng ký hiệu $. Cần thống nhất đơn vị trước khi diễn giải.

---

## 3. `fact_sales` – Giao dịch bán hàng

58.414 dòng; 27.345 đơn hàng; tổng doanh thu 17.912.095.075 ₫.

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `order_number` | Văn bản | Mã đơn hàng (một đơn có nhiều dòng) |
| `order_line_number` | Số nguyên | Số thứ tự dòng trong đơn |
| `customer_key` | Số nguyên | Khóa khách hàng |
| `product_key` | Số nguyên | Khóa sản phẩm |
| `store_key` | Số nguyên | Khóa cửa hàng |
| `promotion_key` | Số nguyên | Khóa khuyến mãi |
| `order_quantity` | Số nguyên | Số lượng mua |
| `unit_price` | Số thực | Đơn giá ✱ |
| `unit_cost` | Số thực | Giá vốn đơn vị ✱ |
| `unit_discount` | Số thực | Mức giảm giá trên đơn vị ✱ |
| `sales_amount` | Số thực | Thành tiền của dòng, là cơ sở tính doanh thu và Monetary |
| `order_date` | Ngày | Ngày đặt hàng, quan hệ active với `dim_date` |
| `ship_date` | Ngày | Ngày giao hàng, quan hệ inactive |
| `payment_date` | Ngày | Ngày thanh toán, quan hệ inactive |
| `order_date_key` | Số nguyên | Khóa ngày dạng số ✱ |

---

## 4. `fact_sales_plan` – Kế hoạch doanh thu

Nguồn: file Excel kế hoạch doanh thu, gộp ba năm 2020, 2021, 2022.

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `district` | Văn bản | Quận |
| `store` | Văn bản | Tên cửa hàng, khớp với `dim_store[store]` |
| `date` | Ngày | Tháng của kế hoạch |
| `sales_plan` | Số thập phân | Doanh thu kế hoạch. Định dạng hiện tại là `$` trong khi doanh thu thực tế là ₫; cần kiểm tra đơn vị trước khi so sánh thực tế với kế hoạch |

---

## 5. `dim_product` – Sản phẩm (353 sản phẩm)

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `product_key` | Số nguyên | Khóa sản phẩm |
| `product_name` | Văn bản | Tên sản phẩm |
| `product_category` | Văn bản | Nhóm sản phẩm: Thực phẩm đóng gói, Đồ gia dụng, Gia vị - nguyên liệu nấu ăn, Thực phẩm đông lạnh, Thực phẩm tươi sống, Thức uống, Thực phẩm khô |
| `product_subcategory` | Văn bản | Nhóm con của sản phẩm |
| `price` | Số thực | Giá bán ✱ |
| `cost` | Số thực | Giá vốn ✱ |
| `country ` | Văn bản | Quốc gia xuất xứ ✱ (tên cột trong nguồn có dấu cách ở cuối) |
| `brand` | Văn bản | Thương hiệu |
| `suplier_name` | Văn bản | Nhà cung cấp (tên cột giữ nguyên chính tả của nguồn) |
| `uom_size` | Văn bản | Quy cách / đơn vị tính ✱ |
| `link` | Văn bản | Liên kết sản phẩm ✱ |

---

## 6. `dim_store` – Cửa hàng (29 cửa hàng)

Cột `store` đã bỏ tiền tố "KPIM Mart" và cắt khoảng trắng.

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `store_key` | Số nguyên | Khóa cửa hàng |
| `store` | Văn bản | Tên cửa hàng |
| `manager` | Văn bản | Quản lý cửa hàng |
| `manager_image` | Văn bản | Đường dẫn ảnh quản lý ✱ |
| `city` | Văn bản | Thành phố |
| `district` | Văn bản | Quận |
| `ward` | Văn bản | Phường |
| `address` | Văn bản | Địa chỉ |
| `latitude` | Số thực | Vĩ độ |
| `longitude` | Số thực | Kinh độ |

---

## 7. `dim_promotion` – Khuyến mãi (16 chương trình)

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `promotion_key` | Số nguyên | Khóa khuyến mãi |
| `promotion` | Văn bản | Tên chương trình |
| `discount_percent` | Số thực (%) | Tỷ lệ giảm giá |
| `type` | Văn bản | Loại: Không giảm giá, Giảm giá theo số lượng, Xả hàng cũ, Giảm giá theo mùa, Xả hàng tồn, Sản phẩm mới |
| `start_date` | Ngày | Ngày bắt đầu |
| `end_date` | Ngày | Ngày kết thúc |
| `min_qty` | Số nguyên | Số lượng tối thiểu để áp dụng ✱ |
| `max_qty` | Số nguyên | Số lượng tối đa áp dụng ✱ |

---

## 8. `dim_date` – Lịch

Được đánh dấu là bảng ngày (`dataCategory: Time`), khóa là cột `date`.

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `date` | Ngày | Ngày (khóa) |
| `day` | Số nguyên | Ngày trong tháng |
| `month` | Số nguyên | Tháng |
| `quarter` | Số nguyên | Quý |
| `year` | Số nguyên | Năm |
| `Day of Week` | Số nguyên | Thứ trong tuần (0 = Chủ nhật, theo `Date.DayOfWeek` mặc định của Power Query) |
| `Day Name` | Văn bản | Tên thứ viết tắt 3 ký tự |
| `Month Name` | Văn bản | Tên tháng viết tắt 3 ký tự |
| `End of Month` | Ngày | Ngày cuối tháng |
| `Quarter Name` | Văn bản | `Q1`–`Q4` |
| `working_day` | Văn bản | `true` nếu không phải Thứ bảy/Chủ nhật, ngược lại `false` |

---

## 9. `map_rfm_code` – Bảng tra 125 tổ hợp R-F-M

Bảng tính toán (DAX): `CROSSJOIN` của R, F, M từ 1 đến 5 (125 dòng), thêm một dòng cho khách chưa mua hàng.

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `R-F-M` | Văn bản | Mã tổ hợp, ví dụ `5-5-5`; dòng đặc biệt `--` cho khách chưa mua |
| `R` | Số | Điểm Recency (1–5) |
| `F` | Số | Điểm Frequency (1–5) |
| `M` | Số | Điểm Monetary (1–5) |
| `FM_avg` | Số | Trung bình của F và M |
| `segment` | Văn bản | Phân khúc RFM (11 phân khúc + `Chưa mua hàng`) |

---

## 10. `dim_rfm_segment` – Mô tả và hành động theo phân khúc

Bảng tính toán (`DATATABLE`), 12 dòng (11 phân khúc + `Chưa mua hàng`).

| Cột | Kiểu | Mô tả |
| --- | --- | --- |
| `segment` | Văn bản | Tên phân khúc |
| `action_group` | Văn bản | Nhóm hành động (ví dụ Giữ chân và tăng CLTV, Kích hoạt lại chi phí thấp) |
| `description` | Văn bản | Mô tả đặc điểm hành vi của phân khúc |
| `recommended_action` | Văn bản | Gợi ý hành động marketing / chăm sóc |

---

## 11. Quy ước đặt tên

- Bảng: tiền tố `dim_` (chiều), `fact_` (sự kiện), `map_` (bảng tra cứu).
- Cột: chữ thường, ngăn cách bằng dấu gạch dưới; một số cột tính toán dùng tên riêng (`Age`, `score r-f-m`, `segment(group)`).
- Measure: tên tiếng Việt, nhóm theo thư mục hiển thị: Doanh thu, Khách hàng, RFM, Kế hoạch, Tooltips.
