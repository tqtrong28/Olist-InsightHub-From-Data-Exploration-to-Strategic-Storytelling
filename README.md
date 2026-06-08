# Olist-InsightHub-From-Data-Exploration-to-Strategic-Storytelling

## 1. Tổng quan dự án

Đây là dự án bài tập lớn cho môn **IT5425: Quản trị và trực quan hóa dữ liệu**.Link data :  https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce  
Hướng tiếp cận của dự án là **Hybrid Approach**:

- dùng **Python + Pandas + Plotly** để làm sạch dữ liệu và phân tích khám phá
- dùng **Tableau** để xây dựng dashboard tương tác theo hướng giải thích và kể chuyện dữ liệu

Mục tiêu tổng quát của dự án là trả lời ba nhóm câu hỏi:

1. Doanh thu của Olist đến từ đâu?
2. Khâu giao hàng và vận hành đang gặp vấn đề ở đâu?
3. Trải nghiệm khách hàng bị ảnh hưởng như thế nào bởi vận hành và giao hàng?

## 2. Cấu trúc thư mục hiện tại

Các thành phần chính đang được giữ lại trong project:

- `raw_data/`: dữ liệu gốc ban đầu
- `clean_data/`: dữ liệu sạch sau Giai đoạn 1
- `phase1_data_preparation.ipynb`: notebook cho Giai đoạn 1
- `phase2_business_eda.ipynb`: notebook cho Giai đoạn 2
- `eda_outputs/phase2/`: nơi dự kiến lưu chart và summary khi chạy notebook Giai đoạn 2

## 3. Giai đoạn 1 - Chuẩn bị và Làm sạch dữ liệu

### 3.1. Mục tiêu

Giai đoạn 1 tập trung vào việc biến dữ liệu thô thành dữ liệu sạch, ổn định, và an toàn để phân tích.  
Mục tiêu cụ thể:

- đọc và kiểm tra toàn bộ 9 file CSV gốc
- xử lý quan hệ giữa các bảng theo đúng sơ đồ dữ liệu
- tránh đếm trùng khi merge các bảng 1-n
- tạo các bảng phân tích sạch để dùng lại cho EDA và Tableau

### 3.2. Vấn đề cốt lõi của dataset

Dataset Olist không thể merge thẳng tất cả bảng vào một bảng duy nhất, vì:

- `order_items` ở cấp dòng sản phẩm
- `order_payments` ở cấp lần thanh toán
- `order_reviews` ở cấp lần đánh giá
- `orders` ở cấp đơn hàng
- `geolocation` có hơn 1 triệu dòng, rất nặng nếu join trực tiếp

Nếu nối không cẩn thận:

- doanh thu bị đếm trùng
- payment bị nhân bản theo item
- review bị lặp theo item
- Tableau và Plotly bị chậm

### 3.3. Hướng xử lý đã chọn

Giai đoạn 1 được thiết kế theo nguyên tắc:

1. `orders` là trục trung tâm
2. `payments` và `reviews` phải aggregate về cấp `order_id` trước khi join
3. `geolocation` phải được nén về cấp `zip_code_prefix` trước khi nối vào `customers` và `sellers`
4. dữ liệu cuối cùng được tách thành 2 mart riêng, không ép về một bảng duy nhất

### 3.4. Các bước chính đã triển khai trong notebook Phase 1

#### Bước 1 - Đọc dữ liệu gốc

Đọc 9 file CSV từ `raw_data/`, parse các cột thời gian quan trọng như:

- `order_purchase_timestamp`
- `order_approved_at`
- `order_delivered_customer_date`
- `order_estimated_delivery_date`
- `shipping_limit_date`
- `review_creation_date`
- `review_answer_timestamp`

#### Bước 2 - Kiểm tra chất lượng dữ liệu

Thực hiện profile nhanh để kiểm tra:

- số dòng, số cột
- khóa duy nhất
- giá trị thiếu
- phân bố `order_status`
- mức độ lặp của `geolocation`

#### Bước 3 - Tạo bảng địa lý rút gọn

Từ `olist_geolocation_dataset.csv`, dữ liệu được gom theo `zip_code_prefix` để tạo bảng:

- `dim_geo_zip`

Bảng này giữ:

- tọa độ đại diện `lat/lng`
- `city/state`
- số lượng điểm gốc của từng zip

Mục tiêu là giảm dữ liệu từ hơn 1 triệu dòng xuống mức nhỏ hơn rất nhiều để tăng hiệu năng xử lý.

#### Bước 4 - Tạo dimension khách hàng và người bán

Từ bảng địa lý rút gọn, hệ thống tạo:

- `dim_customers_geo`
- `dim_sellers_geo`

Hai bảng này cho phép:

- xác định vị trí khách hàng
- xác định vị trí người bán
- chuẩn bị cho phân tích map và khoảng cách giao hàng

#### Bước 5 - Tạo dimension sản phẩm

Từ bảng sản phẩm và bảng dịch category, hệ thống tạo:

- `dim_products`

Mục tiêu:

- chuẩn hóa category
- hỗ trợ phân tích theo nhóm sản phẩm dễ đọc hơn

#### Bước 6 - Aggregate payment và review

Hai bảng dễ gây lỗi nhất là:

- `agg_payments`
- `agg_reviews`

Ở đây các thông tin được gom về cấp `order_id` để sau này join an toàn với `orders`.

#### Bước 7 - Tạo 2 bảng mart cuối cùng

Hai bảng quan trọng nhất được tạo ra là:

- `mart_order_level.csv`
  - mỗi dòng là 1 đơn hàng
  - dùng cho logistics, delay, review, satisfaction

- `mart_item_level.csv`
  - mỗi dòng là 1 item trong đơn hàng
  - dùng cho sales, category, seller, freight

### 3.5. Các biến dẫn xuất quan trọng của Giai đoạn 1

Một số biến mới được tạo ra để phục vụ phân tích:

- `shipping_days`
- `delivery_delay_days`
- `is_delayed`
- `purchase_year_month`
- `payment_value_total`
- `review_score_mean`
- `freight_ratio`
- `seller_customer_distance_km`

### 3.6. Đầu ra hiện có của Giai đoạn 1

Trong thư mục `clean_data/` hiện có:

- `dim_geo_zip.csv`
- `dim_customers_geo.csv`
- `dim_sellers_geo.csv`
- `dim_products.csv`
- `agg_payments.csv`
- `agg_reviews.csv`
- `mart_order_level.csv`
- `mart_item_level.csv`
- `phase1_validation_summary.csv`

### 3.7. Kết quả của Giai đoạn 1

Sau Giai đoạn 1, dự án đã có một lớp dữ liệu sạch đủ tốt để:

- phân tích kinh doanh bằng Python
- xây dựng dashboard trong Tableau
- hạn chế tối đa lỗi đếm trùng KPI

## 4. Giai đoạn 2 - Phân tích khám phá theo hướng kinh doanh

### 4.1. Mục tiêu

Giai đoạn 2 không tập trung vào lý thuyết môn học ngay, mà đi theo hướng **business-first**.  
Mục tiêu là dùng dữ liệu sạch để tìm insight thật sự có giá trị, làm nền cho dashboard Tableau.

Ba câu hỏi kinh doanh chính của Giai đoạn 2 là:

1. Nhu cầu đến từ đâu?
2. Giao hàng đang có vấn đề ở đâu?
3. Vận hành ảnh hưởng đến review khách hàng như thế nào?

### 4.2. Cách tiếp cận đã chọn

Giai đoạn 2 được tổ chức theo business funnel:

- `Demand`
- `Fulfillment`
- `Satisfaction`

Lý do chọn cách này:

- mạch phân tích rõ ràng
- dễ chuyển thành 3 tab dashboard ở Tableau
- không bị sa đà vào việc vẽ nhiều chart nhưng ít ý nghĩa

### 4.3. Dữ liệu sử dụng

Notebook Phase 2 dùng chủ yếu:

- `clean_data/mart_order_level.csv`
- `clean_data/mart_item_level.csv`

Nguyên tắc sử dụng:

- dùng `mart_order_level` cho KPI cấp đơn, delay, review
- dùng `mart_item_level` cho doanh thu theo category, state, seller
- không dùng `mart_item_level` để tính KPI cấp đơn nếu chưa deduplicate theo `order_id`

### 4.4. Cấu trúc phân tích của Giai đoạn 2

#### Phần A - Health Check và KPI nền

Mục tiêu:

- kiểm tra dữ liệu sạch đã đủ tốt để phân tích chưa
- tạo bức tranh tổng quan ban đầu

KPI nền dự kiến:

- `GMV`
- `Orders`
- `Delivered Orders`
- `AOV`
- `On-time Rate`
- `Delayed Rate`
- `Average Review Score`
- `Median Shipping Days`

#### Phần B - Demand Analysis

Câu hỏi cần trả lời:

- doanh thu tăng giảm theo thời gian như thế nào
- category nào là động cơ doanh thu
- bang nào đóng góp doanh thu lớn nhất

Biểu đồ và summary chính:

- `Monthly Revenue Trend`
- `Top Categories by GMV`
- `Top States by GMV`
- summary theo tháng, category, state

#### Phần C - Fulfillment Analysis

Câu hỏi cần trả lời:

- bang nào có delayed rate cao nhất
- shipping time phân bố ra sao
- khoảng cách seller-customer có liên hệ với shipping time không

Biểu đồ và summary chính:

- `Delayed Rate by State`
- `Shipping Days Distribution`
- `Distance vs Shipping Time`

#### Phần D - Satisfaction Analysis

Câu hỏi cần trả lời:

- giao trễ có kéo review xuống không
- delay bucket nào đáng lo nhất
- nhóm đơn nào dễ có review thấp

Biểu đồ và summary chính:

- `Average Review by Delay Bucket`
- `Delivery Delay by Review Score`

### 4.5. Notebook hiện có cho Giai đoạn 2

File:

- `phase2_business_eda.ipynb`

Notebook này đã được scaffold sẵn với:

- comment tiếng Việt có dấu
- cấu trúc phân tích rõ ràng
- hàm export chart `.html` và `.png`
- hàm export summary `.csv`
- thư mục đích là `eda_outputs/phase2/`

### 4.6. Trạng thái hiện tại của Giai đoạn 2

Giai đoạn 2 đã được chuẩn bị xong về mặt cấu trúc và notebook, nhưng chưa được chạy tự động trong quá trình tạo scaffold.  
Khi chạy notebook thủ công, kết quả dự kiến sẽ bao gồm:

- các bảng summary cho demand, fulfillment, satisfaction
- các biểu đồ Plotly phục vụ chọn insight
- các phát hiện chính để điền lại vào phần kết luận

## 5. Giai đoạn 3 - Xây dựng Dashboard bằng Tableau

Giai đoạn 3 sử dụng Tableau để xây dựng dashboard tương tác gồm **3 tab chính**, đi theo mạch phân tích business:

```text
Sales / Demand -> Logistics / Fulfillment -> Customer Review / Satisfaction
```

Mục tiêu của giai đoạn này không chỉ là vẽ biểu đồ, mà là biến các insight từ EDA thành một hệ thống dashboard dễ đọc, dễ tương tác và có khả năng kể chuyện dữ liệu.

Người xem có thể bắt đầu từ câu hỏi **doanh thu đến từ đâu**, sau đó kiểm tra **vận hành giao hàng**, rồi cuối cùng xem **giao hàng ảnh hưởng như thế nào đến đánh giá của khách hàng**.

---

## 5.1. Tab 1 - Sales / Demand Overview

### Mục tiêu

Tab này trả lời câu hỏi:

> **Doanh thu đến từ đâu?**

Tab này tập trung vào nhu cầu mua hàng và doanh thu. Nó giúp người xem hiểu:

- tổng quan quy mô kinh doanh
- GMV thay đổi như thế nào theo thời gian
- category nào đóng góp nhiều GMV nhất
- bang nào tạo ra nhiều doanh thu nhất

### Ý nghĩa của tab

Tab này phù hợp với người xem thuộc nhóm:

- business
- sales
- category management

Người xem có thể dùng tab này để trả lời các câu hỏi:

- Tổng GMV hiện tại là bao nhiêu?
- Có bao nhiêu đơn hàng?
- Trung bình một đơn hàng mang lại bao nhiêu doanh thu?
- Phí vận chuyển chiếm tỷ lệ bao nhiêu so với doanh thu?
- Doanh thu tăng giảm như thế nào theo thời gian?
- Category nào là nhóm sản phẩm chủ lực?
- Doanh thu tập trung ở bang nào?

### Bảng dữ liệu chính

Dùng:

- `mart_item_level.csv`

Lý do:

- Tab này phân tích theo cấp item/sản phẩm.
- Các thông tin như `price`, `freight_value`, `product_category_name`, GMV và freight ratio phù hợp hơn ở cấp item.

### KPI đã sử dụng

- `GMV`
- `Orders`
- `AOV`
- `Freight Ratio`

Ý nghĩa:

| KPI | Ý nghĩa |
|---|---|
| `GMV` | Tổng doanh thu hàng hóa, tính từ giá sản phẩm |
| `Orders` | Tổng số đơn hàng khác nhau |
| `AOV` | Giá trị trung bình mỗi đơn hàng |
| `Freight Ratio` | Tỷ lệ phí vận chuyển so với doanh thu hàng hóa |

### Biểu đồ chính

- `Monthly GMV Trend`
- `Top Categories by GMV`
- `GMV by Customer State`

Ý nghĩa từng biểu đồ:

| Biểu đồ | Ý nghĩa |
|---|---|
| `Monthly GMV Trend` | Cho biết GMV thay đổi theo thời gian |
| `Top Categories by GMV` | Cho biết category nào tạo ra doanh thu lớn nhất |
| `GMV by Customer State` | Cho biết bang khách hàng nào đóng góp nhiều GMV nhất |

### Filter / tương tác

Tab này sử dụng:

- `Purchase Date`
- `Customer State`

Ý nghĩa filter:

| Filter | Mục đích |
|---|---|
| `Purchase Date` | Cho phép người xem chọn khoảng thời gian cần phân tích. Khi thay đổi khoảng thời gian, toàn bộ KPI và biểu đồ trong tab sẽ được tính lại. |
| `Customer State` | Cho phép người xem lọc theo một hoặc nhiều bang khách hàng. Khi chọn một bang cụ thể, dashboard sẽ cho biết riêng bang đó có GMV, số đơn, AOV, freight ratio, category và xu hướng doanh thu như thế nào. |

### Cách người xem sử dụng tab này

Người xem nên đọc tab này theo thứ tự:

1. Nhìn hàng KPI để hiểu quy mô doanh thu tổng quan.
2. Xem `Monthly GMV Trend` để biết doanh thu thay đổi theo thời gian.
3. Xem `Top Categories by GMV` để biết nhóm sản phẩm nào đóng góp doanh thu chính.
4. Xem `GMV by Customer State` để biết doanh thu tập trung ở bang nào.
5. Sử dụng `Purchase Date` hoặc `Customer State` để phân tích sâu hơn theo thời gian hoặc địa điểm.

### Insight mong muốn

Tab này giúp người xem kết luận được:

> Doanh thu của Olist tập trung vào một số category chủ lực và một số bang lớn. Đây là nền tảng để tiếp tục kiểm tra xem các khu vực có doanh thu cao có đang được phục vụ tốt về mặt logistics hay không.

---

## 5.2. Tab 2 - Logistics / Fulfillment Overview

### Mục tiêu

Tab này trả lời câu hỏi:

> **Giao hàng có vấn đề ở đâu, mức độ chậm như thế nào, và khu vực nào cần chú ý?**

Sau khi Tab 1 cho biết doanh thu đến từ đâu, Tab 2 kiểm tra xem khâu vận hành và giao hàng có ổn định không.

### Ý nghĩa của tab

Tab này phù hợp với người xem thuộc nhóm:

- operations
- logistics
- fulfillment

Người xem có thể dùng tab này để trả lời các câu hỏi:

- Có bao nhiêu đơn hàng được xét?
- Tỷ lệ giao trễ là bao nhiêu?
- Tỷ lệ giao đúng hạn là bao nhiêu?
- Thời gian giao hàng trung vị là bao nhiêu ngày?
- Bang nào có tỷ lệ giao trễ cao?
- Phần lớn đơn hàng mất bao nhiêu ngày để giao?
- Bang nào có mức delay burden cao hơn?

### Bảng dữ liệu chính

Dùng:

- `mart_order_level.csv`

Lý do:

- Tab này phân tích theo cấp đơn hàng.
- Các thông tin như trạng thái giao hàng, thời gian giao, delay, customer state và metric vận hành đều nằm ở cấp order.

### KPI đã sử dụng

- `Total Orders`
- `Delayed Rate`
- `On-time Rate`
- `Median Shipping Days`

Ý nghĩa:

| KPI | Ý nghĩa |
|---|---|
| `Total Orders` | Tổng số đơn hàng trong phạm vi đang lọc |
| `Delayed Rate` | Tỷ lệ đơn giao trễ so với ngày giao dự kiến |
| `On-time Rate` | Tỷ lệ đơn giao đúng hạn hoặc sớm |
| `Median Shipping Days` | Số ngày giao hàng trung vị |

### Biểu đồ chính

- `Delayed Rate by Customer State`
- `Shipping Days Distribution`
- `Average Delay Burden by Customer State`

Ý nghĩa từng biểu đồ:

| Biểu đồ | Ý nghĩa |
|---|---|
| `Delayed Rate by Customer State` | Cho biết bang nào có tỷ lệ giao trễ cao nhất |
| `Shipping Days Distribution` | Cho biết phân phối số ngày giao hàng, tức phần lớn đơn hàng thường mất bao nhiêu ngày để giao |
| `Average Delay Burden by Customer State` | Cho biết mức độ ảnh hưởng trung bình của delay theo từng bang |

### Ghi chú về Delay Burden

`Average Delay Burden by Customer State` được hiểu là mức trễ trung bình theo bang, trong đó:

- đơn không trễ được tính là `0` ngày trễ
- đơn trễ được tính theo số ngày trễ thực tế

Ví dụ:

```text
0 ngày trễ, 0 ngày trễ, 5 ngày trễ, 10 ngày trễ
=> Delay Burden = (0 + 0 + 5 + 10) / 4 = 3.75 ngày
```

Metric này khác với `Delayed Rate`:

| Metric | Câu hỏi trả lời |
|---|---|
| `Delayed Rate` | Bao nhiêu phần trăm đơn bị trễ? |
| `Delay Burden` | Mức độ trễ trung bình nặng đến đâu? |

### Filter / tương tác

Tab này sử dụng:

- `Purchase Date`
- `Customer State`
- `Is Delivered`

Ý nghĩa filter:

| Filter | Mục đích |
|---|---|
| `Purchase Date` | Cho phép người xem kiểm tra logistics performance trong một khoảng thời gian cụ thể |
| `Customer State` | Cho phép người xem tập trung vào một hoặc nhiều bang để xem khu vực đó có vấn đề giao hàng hay không |
| `Is Delivered` | Cho phép người xem chọn xem tất cả đơn hàng hoặc tập trung vào các đơn đã giao |

### Cách người xem sử dụng tab này

Người xem nên đọc tab này theo thứ tự:

1. Nhìn KPI để biết tình hình fulfillment tổng quan.
2. Xem `Delayed Rate by Customer State` để biết bang nào thường bị giao trễ.
3. Xem `Shipping Days Distribution` để biết phần lớn đơn hàng mất bao lâu để giao.
4. Xem `Average Delay Burden by Customer State` để biết bang nào chịu mức trễ trung bình nặng hơn.
5. Sử dụng filter `Customer State` để kiểm tra một bang cụ thể.
6. Sử dụng filter `Is Delivered` để tập trung vào nhóm đơn đã giao nếu cần phân tích shipping days chính xác hơn.

### Insight mong muốn

Tab này giúp người xem kết luận được:

> Một số bang có vấn đề logistics rõ rệt hơn các bang khác. Không chỉ cần biết nơi nào tạo doanh thu lớn, mà còn cần kiểm tra nơi đó có được giao hàng ổn định hay không.

---

## 5.3. Tab 3 - Customer Review / Satisfaction Overview

### Mục tiêu

Tab này trả lời câu hỏi:

> **Giao hàng và vận hành ảnh hưởng đến đánh giá khách hàng như thế nào?**

Đây là tab mang tính storytelling mạnh nhất vì nó nối trực tiếp kết quả từ logistics sang trải nghiệm khách hàng.

### Ý nghĩa của tab

Tab này phù hợp với người xem thuộc nhóm:

- customer experience
- service quality
- management

Người xem có thể dùng tab này để trả lời các câu hỏi:

- Điểm review trung bình là bao nhiêu?
- Tỷ lệ review thấp là bao nhiêu?
- Tỷ lệ review tích cực là bao nhiêu?
- Có bao nhiêu đơn hàng có review?
- Review score phân phối như thế nào?
- Đơn giao trễ có điểm review thấp hơn đơn giao đúng hạn không?
- Trễ càng lâu thì điểm review có giảm không?

### Bảng dữ liệu chính

Dùng:

- `mart_order_level.csv`

Lý do:

- Review score, delay status, delay days và delivery information đều được phân tích ở cấp đơn hàng.
- Dùng order-level giúp tránh việc một order có nhiều item làm sai lệch phân tích review.

### KPI đã sử dụng

- `Average Review Score`
- `Low Review Rate`
- `Positive Review Rate`
- `Reviewed Orders`

Ý nghĩa:

| KPI | Ý nghĩa |
|---|---|
| `Average Review Score` | Điểm review trung bình |
| `Low Review Rate` | Tỷ lệ đơn có review thấp, được định nghĩa là 1-2 sao |
| `Positive Review Rate` | Tỷ lệ đơn có review tích cực, được định nghĩa là 4-5 sao |
| `Reviewed Orders` | Số đơn hàng có thông tin review |

### Biểu đồ chính

- `Review Score Distribution`
- `Average Review Score by Delay Status`
- `Review Score by Delay Bucket`

Ý nghĩa từng biểu đồ:

| Biểu đồ | Ý nghĩa |
|---|---|
| `Review Score Distribution` | Cho biết khách hàng thường đánh giá bao nhiêu sao |
| `Average Review Score by Delay Status` | So sánh điểm review trung bình giữa đơn giao đúng hạn/sớm và đơn giao trễ |
| `Review Score by Delay Bucket` | Cho biết điểm review thay đổi như thế nào khi mức độ trễ tăng lên |

### Delay Bucket

Trong dashboard cuối cùng, `Delay Bucket` được chia thành:

- `On-time / Early`
- `Delay 1-3 days`
- `Delay 4-7 days`
- `Delay > 7 days`

Nhóm `Unknown` đã được bỏ khỏi chart để tập trung vào quan hệ giữa delay thực tế và review score.

### Filter / tương tác

Tab này sử dụng:

- `Purchase Date`
- `Customer State`
- `Is Delivered`

Ý nghĩa filter:

| Filter | Mục đích |
|---|---|
| `Purchase Date` | Cho phép người xem phân tích satisfaction trong một khoảng thời gian cụ thể |
| `Customer State` | Cho phép người xem xem riêng review pattern của một hoặc nhiều bang |
| `Is Delivered` | Cho phép người xem chọn xem tất cả đơn hàng hoặc tập trung vào đơn đã giao khi phân tích quan hệ giữa delivery delay và review score |

### Cách người xem sử dụng tab này

Người xem nên đọc tab này theo thứ tự:

1. Nhìn KPI để biết mức độ hài lòng tổng quan.
2. Xem `Review Score Distribution` để biết review tập trung ở mức mấy sao.
3. Xem `Average Review Score by Delay Status` để kiểm tra xem đơn giao trễ có review thấp hơn không.
4. Xem `Review Score by Delay Bucket` để kiểm tra xem trễ càng lâu thì review có giảm mạnh hơn không.
5. Dùng filter `Customer State` để kiểm tra liệu một bang cụ thể có bị ảnh hưởng mạnh hơn bởi delay hay không.

### Insight mong muốn

Tab này giúp người xem kết luận được:

> Trải nghiệm khách hàng không chỉ phụ thuộc vào sản phẩm, mà còn chịu ảnh hưởng rõ từ giao hàng. Các đơn giao trễ có xu hướng nhận review thấp hơn, và khi mức độ trễ tăng lên thì điểm review trung bình giảm rõ rệt.

---

## 5.4. Tổng kết cấu trúc 3 tab

Ba tab dashboard được thiết kế theo một mạch phân tích thống nhất:

### Tab 1 - Sales / Demand

- **Câu hỏi chính:** doanh thu đến từ đâu?
- **Trọng tâm:** GMV, Orders, AOV, Freight Ratio, Category, State, Time Trend
- **Vai trò:** xác định nguồn doanh thu và nhu cầu mua hàng

### Tab 2 - Logistics / Fulfillment

- **Câu hỏi chính:** giao hàng có vấn đề ở đâu?
- **Trọng tâm:** Delayed Rate, On-time Rate, Shipping Days, Delay Burden, Customer State
- **Vai trò:** kiểm tra năng lực vận hành và các khu vực có vấn đề giao hàng

### Tab 3 - Customer Review / Satisfaction

- **Câu hỏi chính:** giao hàng ảnh hưởng review như thế nào?
- **Trọng tâm:** Average Review Score, Low Review Rate, Positive Review Rate, Delay Status, Delay Bucket
- **Vai trò:** kết nối vận hành với trải nghiệm khách hàng

Tổng thể dashboard đi theo logic:

```text
Doanh thu đến từ đâu?
-> Giao hàng có ổn không?
-> Giao hàng ảnh hưởng review như thế nào?
```

---

## 6. Trạng thái hiện tại và bước tiếp theo

### Trạng thái hiện tại

- Giai đoạn 1 đã hoàn thành phần làm sạch dữ liệu và tạo các bảng sạch trong `clean_data/`.
- Giai đoạn 2 đã hoàn thành phần business EDA bằng Python để tìm insight ban đầu.
- Giai đoạn 3 đã xây dựng xong dashboard Tableau gồm 3 tab:
  - `Sales / Demand Overview`
  - `Logistics / Fulfillment Overview`
  - `Customer Review / Satisfaction Overview`
- Các dashboard đã có KPI, biểu đồ chính và filter tương tác theo đúng mạch phân tích business.
- Tài liệu mô tả chi tiết được gộp lại trong file `README.md` này.

