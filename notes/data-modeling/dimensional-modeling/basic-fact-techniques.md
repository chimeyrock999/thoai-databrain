# Basic Fact Techniques

## Fact Table Structure

Một Fact Table cần chứa _kết quả định lượng/numeric measures_ ra bởi một operational measurement event.

{% hint style="info" %}
## Ví dụ:

Khi **khách hàng mua hàng**, ta có các con số như _số lượng, giá tiền, chiết khấu, thuế,…_ Đây là các **measures**, và mỗi lần mua hàng (transaction) chính là **một event thực tế**.
{% endhint %}

Ở cấp độ chi tiết nhất (lowest grain), mỗi dòng (row) trong bảng fact tương ứng với một sự kiện đo lường thực tế — và ngược lại.

Việc thiết kế bảng fact phải dựa trên hoạt động thực tế trong hệ thống vận hành (operational activity), chứ không dựa vào các báo cáo mà sau này ta muốn tạo.

{% hint style="danger" %}
## Lưu ý:

Bạn không nên thiết kế bảng fact chỉ để “phục vụ báo cáo doanh thu theo tháng”. Mà thay vào đó, hãy ghi nhận dữ liệu theo từng giao dịch thực tế (transaction grain). Vì khi có dữ liệu chi tiết, ta có thể tổng hợp (aggregate) lên mức tháng, quý, năm sau này rất dễ dàng.
{% endhint %}

Ngoài các measures (số liệu), bảng fact còn chứa:

* **Foreign keys:** trỏ đến các bảng dimension (ví dụ: `customer_key`, `product_key`, `store_key`, …)
* **Optional** [_**degenerate dimensions**_](#user-content-fn-1)[^1]: là các giá trị “dimension” nhưng không có bảng riêng, ví dụ như `invoice_number`, `order_id`, …(được lưu ngay trong fact table luôn)
* **Date/time stamps**: các cột ghi nhận thời gian của event, ví dụ `order_date_key`, `shipped_date_key`, …

Khi người dùng chạy truy vấn (query) hoặc tạo báo cáo, thì phần tính toán và tổng hợp (aggregation) chủ yếu diễn ra trên bảng fact.

{% hint style="info" %}
## Ví dụ:

Tính tổng doanh thu (`SUM(sales_amount)`), Đếm số đơn hàng (`COUNT(order_id)`), Trung bình số lượng bán ra (`AVG(quantity)`),… Tất cả đều bắt đầu từ bảng fact.
{% endhint %}

## Additive, Semi-Additive, Non-Additive Facts

Một fact table có thể chia thành 3 loại:

* **Fully-Additive**: Các đại lượng đo lường có thể cộng dồn trên tất cả các dimension. Ví dụ: `sales_amount`, `quantity_sold`, `revenue`, `cost`, … có thể cộng dồn theo bất cứ chiều nào (thời gian, sản phẩm, khu vực, khách hàng,…)
* **Semi-Additive:** Các đại lượng đo lường có thể cộng dồn trên một số chiều nhưng không phải tất cả. Ví dụ: `account_balance` cộng được theo khách hàng, chi nhanh, tài khoản nhưng không cộng được theo thời gian.
* **Non-Additive:** Các đại lượng không thể cộng dồn trên bất cứ chiều nào. Ví dụ: `discount_rate`, `profit_margin`, `conversion_rate`, `percent_growth`, …

Trong đó, kloại fact dễ sử dụng và linh hoạt là Fully-Additive fact.

Phương án tiếp cận tốt cho non-additive fact là bất cứ khi nào có thể, thay vì lưu trữ non-additive fact, hãy lưu trữ các thành phần additive fact tạo ra nó.

{% hint style="info" %}
## Ví dụ:

Thay vì lưu `profit_margin`, lưu `profit_amount` và `revenue_amount` và sau này BI layer sẽ tính ra `profit_margin = SUM(profit_amount)/SUM(revenue_amount)`.
{% endhint %}

## Null in Fact Tables

Các giá trị định lương trong bảng facts có thể chấp nhận các giá trị null. Do các aggregate functions (`SUM`, `COUNT`, `MIN`, `MAX`, `AVG`) có thể xử lý các giá trị null này một cách chính xác.

Tuy nhiên cần tránh các giá trị null trong các khoá ngoại của bảng fact, bởi vì những giá trị null này có thể dẫn tới vi phạm tính toàn vẹn về tham chiếu (referential integrity).

Khi fact có dimension chưa xác định (unknown, not applicable), ta không dùng `NULL`, mà dùng một surrogate key, trỏ tới một dòng “default” trong dimension table (ví dụ: `Unknown Customer`, `Not Applicable Product`, `Missing Date` ). Như vậy vừa giữ được referential integrity, vừa phân biệt được loại “unknown” trong phân tích.

## Conformed Facts

Trong data warehouse có thể tồn tại nhiều bảng fact khác nhau, và trong đó có một số đại lượng định lượng (measures) trùng tên hoặc tương tự nhau. Khi đó, cần đảm bảo rằng technical definition (định nghĩa kỹ thuật, công thức tính) của các đại lượng này giống hệt nhau nếu muốn so sánh hoặc tổng hợp chúng trong các báo cáo sau này. Ngược lại, nếu các định nghĩa khác nhau, nên đặt tên khác nhau để tránh nhầm lẫn và để cảnh báo người dùng rằng chúng không thể so sánh trực tiếp.

{% hint style="info" %}
## Ví dụ:

Trong các bảng `fact_sales_online`, `fact_sales_store`, `fact_returns`, … có thể tồn tại các measure như `sales_amount`, `revenue`, `profit`, `discount`, …

* Nếu các cột này có định nghĩa giống nhau, nên đặt cùng tên:
  * `fact_sales_online.revenue` và `fact_sales_store.revenue` đều tính theo công thức `quantity * unit_price`, do đó có thể `UNION` hoặc `SUM` chung. Khi đó, `revenue` được xem là một conformed fact - nghĩa là một measure có định nghĩa nhất quán trên nhiều fact table khác nhau.
* Ngược lại, nếu các cột có định nghĩa khác nhau:
  * Chẳng hạn `fact_sales.revenue` là doanh thu gross, `fact_finance.revenue_net` là doanh thu net (đã trừ thuế hoặc chiết khấu), hoặc `fact_returns.return_revenue = refund_amount * -1`, thì nên đặt tên khác nhau để phản ánh sự khác biệt và tránh người dùng hiểu sai khi so sánh
{% endhint %}

## Transaction Fact Tables

Mỗi dòng (row) trong _transaction fact table_ tương ứng với một sự kiện đo lường (measurement event) xảy ra tại một điểm xác định về không gian và thời gian.

Transaction fact table có grain ở mức thấp nhất (atomic grain) — tức là mỗi dòng lưu lại transaction chi tiết nhất — là loại fact table nhiều chiều (most dimensional) và biểu đạt được nhiều khía cạnh nhất (most expressive).

Mỗi transaction fact table luôn có:

* Foreign keys trỏ đến các dimension liên quan (customer, product, date, store, …)
* Và có thể có thêm:
  * Timestamp chi tiết (đến giây, phút)
  * Degenerate dimension key như `order_number`, `transaction_id`

## Periodic Snapshot Fact Tables

Mỗi dòng trong periodic snapshot fact table đại diện cho một khoảng thời gian cố định (period) - ví dụ: một ngày, một tuần, hoặc một tháng - và mỗi dòng chứa tổng hợp (summary) của nhiều transaction hoặc nhiều sự kiện xảy ra trong khoảng thời gian đó. Khác với transaction fact, ở đây grain (độ chi tiết) là thời kỳ (period), chứ không phải từng transaction.

{% hint style="info" %}
## Ví dụ:

`fact_sales_daily` → 1 dòng = doanh thu tổng hợp của một ngày cho mỗi cửa hàng

`fact_inventory_monthly` → 1 dòng = tổng hàng tồn trong tháng đó
{% endhint %}

Các bảng periodic snapshot thường có rất nhiều measures, vì mọi phép đo lường nào có ý nghĩa ở cấp độ thời kỳ đó đều có thể đưa vào.

{% hint style="info" %}
## Ví dụ:

`fact_account_balance_daily` có thể chứa:

* `opening_balance` (số dư đầu ngày)
* `closing_balance` (số dư cuối ngày)
* `deposits` (tổng nạp trong ngày)
* `withdrawals` (tổng rút trong ngày)
* `interest_accrued` (tiền lãi phát sinh) \</aside>
{% endhint %}

Periodic snapshot fact tables thường “dense” về foreign key, nghĩa là với mọi combination dimension hợp lệ, vẫn có 1 dòng cho mỗi period. Ngay cả khi không có hoạt động nào trong kỳ đó, vẫn insert một dòng với các measure = 0 hoặc NULL.

<details>

<summary><i class="fa-clipboard-question">:clipboard-question:</i> Tại sao cần Periodic Snapshot Fact Tables trong khi có thể tính mọi thứ dựa trên Transaction Fact Tables?</summary>

1. **Về chi phí (Cost efficiency & performance):** Vì Periodic Snapshot Fact thường được tổng hợp (aggregated) nên nó có thể chi phí lưu trữ và xử lý

* Bảng snapshot có ít dòng hơn nhiều (1 dòng/period thay vì 1 dòng/event).
* Các truy vấn báo cáo định kỳ (daily, weekly, monthly) chỉ cần đọc snapshot, tránh phải scan toàn bộ transaction fact (rất nặng và đắt).
* Ngoài ra, snapshot còn giúp tách workload OLAP (report) khỏi OLTP hoặc raw transaction zone.

2. **Về tính khả thi (Feasibility of data collection):** Trong nhiều trường hợp, không thể thu thập được transaction-level data nên snapshot là cách duy nhất để ghi nhận số liệu định kỳ. Ví dụ:

* Hệ thống nguồn không log chi tiết từng event (chỉ lưu số dư cuối ngày).
* Hoặc chỉ có thể poll API mỗi ngày để lấy tổng số liệu — không có stream realtime.

3. **Về độ tin cậy nghiệp vụ (Business truth & reconciliation):** Periodic Snapshot Fact cung cấp một “ground truth” đã được chốt nghiệp vụ độc lập với transaction log, giúp đảm bảo tính nhất quán và khả năng đối soát (reconciliation)

* Transaction Fact có thể thay đổi do:
  * Late-arriving events, backfill, correction, manual adjustments
  * Event bị ghi sai hoặc mất log
* Snapshot được “chốt” tại thời điểm chuẩn (end-of-day, end-of-month), phản ánh trạng thái mà business thực sự công nhận.
* Việc lưu song song cả Transaction Fact và Periodic Snapshot Fact cho phép đối chiếu sai lệch (reconciliation) để phát hiện bất thường hoặc thiếu log.

</details>

<details>

<summary><i class="fa-note-sticky">:note-sticky:</i> Không phải tất cả facts trong Periodic Snapshot Fact Table đều là semi-additive nhưng <strong>phần lớn mang tính semi-additive theo thời gian.</strong></summary>

| `opening_balance`   | Số dư đầu kỳ          | ❌ |
| ------------------- | --------------------- | - |
| `closing_balance`   | Số dư cuối kỳ         | ❌ |
| `total_deposits`    | Tổng nạp trong kỳ     | ✅ |
| `total_withdrawals` | Tổng rút trong kỳ     | ✅ |
| `interest_accrued`  | Lãi tích luỹ trong kỳ | ✅ |

</details>

## Accumulating Snapshot Fact Tables

Accumulating facts table (accumulation fact table) là loại bảng fact được dùng để theo dõi các quá trình nghiệp vụ có điểm bắt đầu, điểm kết thúc và các cột mốc (milestone) xác định trong quá trình đó. Mỗi dòng trong bảng biểu diễn một tiến trình công việc hay một đối tượng được theo dõi, đồng thời được cập nhật nhiều lần khi tiến trình đó đi qua các mốc thời gian khác nhau.

Cấu trúc của bảng accumulation fact table điển hình sẽ có:

* Một dòng dữ liệu cho mỗi đối tượng hoặc tiến trình nghiệp vụ (ví dụ một đơn hàng, một lô hàng...)
* Nhiều cột ngày giờ (date key) đánh dấu các mốc quan trọng trong tiến trình (ví dụ ngày tạo đơn, ngày giao hàng, ngày thanh toán)
* Các chỉ số/đo lường (facts) thể hiện trạng thái hiện tại hoặc giá trị tại từng mốc (ví dụ số lượng nhận, số lượng giao, số lượng còn lại)
* Các khóa ngoại liên kết đến các bảng dimension mô tả ngữ cảnh (ví dụ sản phẩm, khách hàng, kho bãi...)
* Dữ liệu được cập nhật theo thời gian (update row), không phải thêm dòng mới như fact bảng giao dịch thông thường.

<details>

<summary><i class="fa-clipboard-question">:clipboard-question:</i> Again, tại sao có thể lưu các state của một entity vào một <em>transaction fact table</em> mà vẫn cần <em>accumulating snapshot fact table</em>?</summary>

**Transaction fact** không giữ được toàn bộ process trong 1 dòng. Ví dụ order đi qua 5 step:

| order\_id | step      | date\_id |
| --------- | --------- | -------- |
| 1001      | created   | 20250101 |
| 1001      | confirmed | 20250102 |
| 1001      | shipped   | 20250104 |
| 1001      | delivered | 20250106 |
| 1001      | closed    | 20250107 |

Transaction fact lưu thành 5 dòng, khi đó muốn phân tích SLA phải join & pivot. Tác vụ này thường nặng và khó, đồng thời phải đối mặt với một số vấn đề như:&#x20;

* Late arriving events
* Step bị skip
* Process thay đổi
* Event đến không đúng order
* Khó reconstruct lifecycle 100% chính xác
* Dễ gây lỗi analytic (order bị double-count, missing event)

**Accumulating snapshot** lưu thành 1 dòng:

| order\_id | created\_date | confirmed\_date | shipped\_date | delivered\_date | closed\_date |
| --------- | ------------- | --------------- | ------------- | --------------- | ------------ |
| 1001      | 20250101      | 20250102        | 20250104      | 20250106        | 20250107     |

Việc tính SLA giữa các step cực nhanh, cực trực quan.

</details>

<details>

<summary><i class="fa-clipboard-question">:clipboard-question:</i> Việc lưu theo “wide/pivoted” style này có vi phạm principle “grateful extension” mà dimensional modeling đề ra không? Ví dụ khi business thay đổi, thêm một step trong quy trình thì bảng fact đó phải cập nhật thêm một cột?</summary>

1. Việc design nên bắt đầu bằng atomic facts là transaction facts, sau đó tuỳ vào độ phức tạp của báo cáo mà mình nên build thêm các bảng facts khác như snapshot/aggregate dựa trên bảng facts này. Thông thường sẽ phải tồn tại cả 2 loại bảng này (theo [nick\_white](https://kimballgroup.forumotion.net/t3015-model-not-fixed-step-for-process-in-a-wide-datawarehouse-extend-fact-table) và [Kimball Group's design tips](https://www.kimballgroup.com/2012/05/design-tip-145-time-stamping-accumulating-snapshot-fact-tables/)).
2. Việc thêm process là thay đổi về mặt business nên việc thay đổi cột trên bảng facts là điều dễ hiểu, tương tự như khi business có thêm metrics trên các bảng facts khác. Và việc thêm cột này cũng không hẳn là sẽ ảnh hưởng đến các báo cáo, câu query hiện có nên vẫn nằm trong "grateful extension principle" của dimensional modeling.

</details>

## Factless Fact Tables

Thông thường, bảng fact lưu các _measure numeric_ (doanh thu, số lượng, chi phí, …). Nhưng có những sự kiện nghiệp vụ chỉ cần ghi nhận việc “xảy ra” mà không có số đo cụ thể. Những trường hợp đó, ta vẫn tạo bảng fact, nhưng chỉ gồm các foreign key đến dimension, không có cột measure numeric nào. Đó chính là Factless Fact Table.

{% hint style="info" %}
## Ví dụ:

Mỗi khi một email được gửi tới khách hàng trong chiến dịch, ta có một **sự kiện (event)**: “Customer A received Campaign X on Day Y via Channel Z.” Vì ví dụ chỉ quan tâm **việc gửi / mở / click** chứ không cần số đo phức tạp, ta lưu dưới dạng factless, mỗi dòng tương ứng 1 email đã **được gửi**.

```markdown
| customer_key | campaign_key | template_key | date_key  | channel_key |
|--------------|--------------|--------------|-----------|-------------|
| 1001         | 101          | 21           | 20251109  | 1           |
| 1002         | 101          | 22           | 20251109  | 2           |
| 1003         | 102          | 20           | 20251110  | 1           |
```

Khi phân tích, ta có thể dùng `COUNT(*)` để trả lời: “hôm nay gửi bao nhiêu email?”, “campaign nào gửi nhiều nhất?”, “kênh nào hiệu quả?”.
{% endhint %}

Khi làm việc với factless fact tables, ta nên (phải) kiểm soát 2 bảng factless fact table song song. Và khi lấy subtract 2 bảng này ta sẽ thu được những sự kiện đã không xảy ra.

* Coverage Factless: Mô tả mọi sự kiện có thể xảy ra
* Activity Factless: Mô tả mọi sự kiện đã xảy ra

{% hint style="info" %}
## Ví dụ:

Trong ví dụ trên, ta luôn phải kiểm soát 2 table:

* `fact_email_targe` - coverage: danh sách những khách hàng cần được gửi email.
* `fact_email_sent` - activity: danh sách những khách hàng đã được gửi email.

Khi lấy subtract, ta sẽ thu được những khách hàng target mà chưa được gửi email. Hơn nữa có thể mở rộng `fact_email_open`, `fact_email_click` để đo được độ hiệu quả trên từng channel, từng email template, etc
{% endhint %}

## **Aggregate Fact Tables**

Là bảng fact tổng hợp được tạo ra từ atomic fact table nhằm tăng tốc độ truy vấn và giảm chi phí tính toán trên BI, không có ý nghĩa nghiệp vụ mới, không thay thế fact gốc. Aggregate fact tables hoạt động như “index”, chúng làm query nhanh hơn nhưng người dùng không cần và không nên chạm trực tiếp tới những bảng này.

Aggregate fact và atomic fact phải xuất hiện đồng thời ở BI và để BI tự động chọn bảng phù hợp dựa theo câu query. Cơ chế này gọi là _aggregate navigation_ và việc chọn bảng phải tự động, không bắt người dùng quyết định.

Aggregate fact dùng [_shrunken conformed dimensions_](#user-content-fn-2)[^2]_._ Aggregate facts vẫn cần join với dimension tuy nhiên sử dụng một shrunken version đã được rút gọn để nhẹ hơn và nhanh hơn.

<details>

<summary><i class="fa-note">:note:</i> <strong>Shrunken Conformed Dimensions</strong></summary>

Là một tập các dimension của một dimensions gốc, tuy nhiên được rút gọn nhằm tối ưu khi truy vấn.

</details>

<details>

<summary><i class="fa-clipboard-question">:clipboard-question:</i> Làm thế nào để control <strong>aggregate navigation</strong> nhất là khi phải hidden logic này với users?</summary>

Có một số phương án như sau:

1. Thông qua [semantic layer](#user-content-fn-3)[^3]: [dbt Semantic Layer](https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl), [cube.js](https://cube.dev/docs/product/introduction), ...
2. Tận dụng cơ chế _Materialized View Rewrite_ của một số loại database/storage: [ClickHouse’s Projections](https://clickhouse.com/docs/sql-reference/statements/alter/projection), [BigQuery’s Materialize view - smart tuning](https://docs.cloud.google.com/bigquery/docs/materialized-views-use#smart_tuning), ...
3. Routing logic bằng SQL logic:
   *   Tạo ra view hoặc query thực hiện logic query aggregate fact tables trước khi tính toán dựa trên atomic facts tables.

       Ví dụ:

       ```sql
       # Create view
       CREATE VIEW vw_sales AS
       SELECT * FROM fact_sales_daily_agg
       WHERE date_key < current_date - INTERVAL '2' DAY
       UNION ALL
       SELECT * FROM fact_sales_atomic
       WHERE date_key >= current_date - INTERVAL '2' DAY;

       # Select
       SELECT SUM(total_sales)
       FROM vw_sales
       WHERE date_key BETWEEN '2025-01-01' AND '2025-01-03';
       ```

       Một số engine sẽ tự động filter và pruning đọc data từ các files không liên quan
   *   Tạo một routing table để handle những logic phức tạp hơn.

       Ví dụ:

       | **metric**   | **grain\_level** | **valid\_from** | **valid\_to** | **target\_table**         |
       | ------------ | ---------------- | --------------- | ------------- | ------------------------- |
       | total\_sales | day              | 1900            | 2024-11-10    | fact\_sales\_daily\_agg   |
       | total\_sales | hour             | 2024-11-11      | 9999          | fact\_sales\_atomic       |
       | total\_sales | month            | 1900            | 9999          | fact\_sales\_monthly\_agg |
       | order\_count | day              | 1900            | now           | fact\_orders\_daily\_agg  |

       Sau đó truy vấn thông qua logic marco

       ```sql
       {% macro route_sales(date_from, date_to, grain) %}

       SELECT target_table
       FROM routing_table
       WHERE metric = 'total_sales'
         AND grain_level = {{ grain }}
         AND valid_from <= {{ date_from }}
         AND valid_to >= {{ date_to }}

       {% endmacro %}

       SELECT *
       FROM {{ route_sales('2025-01-01', '2025-01-03', 'day') }}
       WHERE date_key BETWEEN '2025-01-01' AND '2025-01-03';
       ```

Semantic Layer mang lại khả năng điều hướng aggregate thông minh và nhất quán nhất, nhưng đổi lại làm data platform phức tạp hơn và cần thêm hạ tầng để vận hành.

Materialized View Rewrite là cách nhẹ nhàng nhất vì tận dụng khả năng tối ưu của query engine, nhưng độ tin cậy phụ thuộc từng hệ và không phải engine nào cũng hỗ trợ, đôi khi việc match đúng pattern cũng là một thách thức.

Logical Views và Routing Table thì gọn và linh hoạt, DE kiểm soát được mọi thứ và không lộ logic cho user, nhưng đòi hỏi custom khá nhiều và khó đạt mức dynamic tự động như semantic layer.

</details>

## Consolidated Fact Tables

Consolidated fact là bảng fact hợp nhất được tạo bằng cách gộp dữ liệu từ nhiều quy trình khác nhau (multiple business processes) vào một fact table duy nhất, miễn là tất cả có thể biểu diễn ở cùng một grain.

Mục đích là đơn giản hóa phân tích cross-process, ví dụ như so sánh Actual vs Forecast hoặc Plan vs Actual, giúp BI truy vấn nhanh, trực tiếp, không cần drill-across giữa nhiều fact tables.

Việc hợp nhất làm ETL phức tạp hơn vì cần chuẩn hóa grain, keys và logic cập nhật; nhưng đổi lại, nó giảm đáng kể gánh nặng phân tích cho BI và tránh risk mismatch khi join nhiều fact khác grain.

Consolidated fact tables nên được dùng cho các chỉ số cross-process thường xuyên được sử dụng cùng nhau và cần được so sánh trực tiếp.

## My Summary

Fact tables trong dimensional modeling đều nhằm mô hình hóa hoạt động của business ở nhiều mức chi tiết khác nhau.&#x20;

Thông thường sẽ có các atomic facts để làm "ground truth" và transaction fact là một trong những ứng cử viên bởi sự chi tiết nếu hệ thống có ghi nhận explicit events. Tuy nhiên không phải domain nào cũng có transaction rõ ràng, khi đó snapshot facts sẽ thay thế vai trò chi tiết này.&#x20;

Việc thiết kế nên luôn bắt đầu bằng ground truth, sau đó tuỳ vào độ phức tạp của các báo cáo, cách tính toán metrics, KPIs mà ta xây dựng thêm các bảng phái sinh như snapshot, aggregate hoặc consolidated fact nhằm tối ưu hiệu năng tính toán, bộ nhớ.

Các fact techniques trong dimensional modeling chính là những “design patterns” giúp thiết kế và tổ chức các fact sao cho dễ mở rộng, linh hoạt trước thay đổi của business. Bạn có thể không áp dụng chúng, nhưng giống như việc bỏ qua design pattern trong code, bạn sẽ phải đối mặt với mô hình khó mở rộng, khó bảo trì và dễ gãy khi business tăng trưởn&#x67;**.**

[^1]: Là các dimension chứa mã định danh về nghiệp vụ nhưng không có bảng dimension riêng mà được lưu thẳng vào fact table nhằm tối ưu cho các câu truy vấn.

[^2]: là một tập các dimension của một dimensions gốc, tuy nhiên được rút gọn nhằm tối ưu khi truy vấn.

[^3]: Semantic Layer là lớp trung gian giữa BI và warehouse, nơi định nghĩa thống nhất toàn bộ logic nghiệp vụ như metrics, dimensions và grain, giúp người dùng chỉ làm việc với các khái niệm business thay vì SQL kỹ thuật. Layer này tự sinh truy vấn, tự chọn giữa atomic fact và aggregate fact (aggregate navigation), và đảm bảo mọi dashboard dùng chung một nguồn sự thật duy nhất.
