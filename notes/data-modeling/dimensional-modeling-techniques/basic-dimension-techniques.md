# Basic Dimension Techniques

Dimension tables là các bảng cung cấp bối cảnh để trả lời cho các câu hỏi "who, what, where, when, why, and how” xung quanh business events.

## Dimension Table Structure

Các bảng Dimension luôn có khoá chính là một cột duy nhất. Khoá chính này sẽ được tham chiếu qua khoá ngoại trên các bảng facts nhằm mô tả ngữ cảnh tương ứng với ngữ cảnh của dòng trong bảng facts.

Dimension table nên được thiết kế theo dạng “wide, flat, denormalized”, tức là mô tả càng đầy đủ càng tốt.  Các cột trong bảng thường mang những giá trị dạng low-cardinality text. Mặc dù ta có thể đưa các mã code hoặc trạng thái vận hành vào làm thuộc tính, nhưng các giá trị mô tả chi tiết và dễ hiểu mới là quan trọng nhất.&#x20;

Chính những thuộc tính của bảng này sẽ được dùng để lọc và nhóm dữ liệu trong truy vấn, và cũng chính là các nhãn xuất hiện trên báo cáo hoặc dashboard.

## Dimension Surrogate Keys

Khoá chính của bảng dimension là một surrogate key - thường là số tự nhiên tăng dần -  không mang ý nghĩa nghiệp vụ. Mục đích của nó là giúp kiểm soát sự độc lập, hợp nhất nhiều data source, tránh việc thay đổi logic sau này (triển khai bởi SCD[^1]), bao gồm trùng lặp hoặc tái sử dụng.&#x20;

{% hint style="info" %}
## Ví dụ:

Một khách hàng có natural key C123.\
Ban đầu khách ở HCMC, sau này chuyển sang Hanoi. Vì theo dõi thay đổi bằng SCD Type 2, DW sẽ tạo hai dòng dimension khác nhau, mỗi dòng được gán một surrogate key riêng:

* SK = 1 , NK = C123, city = HCMC
* SK = 2, NK = C123, city = Hanoi

Natural key giống nhau nhưng mỗi phiên bản lịch sử được phân biệt bằng surrogate key. Fact table sẽ join vào đúng SK tương ứng với thời điểm sự kiện xảy ra.
{% endhint %}

Bảng Date dimension là ngoại lệ duy nhất đối với quy tắc surrogate key; vì đây là dimension ổn định và dễ dự đoán, nó có thể sử dụng một khóa chính mang ý nghĩa rõ ràng hơn.

{% hint style="info" %}
## Ví dụ:

Trong Date Dimension, surrogate key không cần dùng số tăng dần. Thay vào đó người ta thường dùng YYYYMMDD làm khóa chính vì nó có ý nghĩa rõ ràng, ổn định và không bao giờ thay đổi theo thời gian.

Ví dụ: 20250101 tương ứng với ngày 01/01/2025, 20250315 tương ứng với ngày 15/03/2025

Khóa chính dạng “YYYYMMDD” giúp việc lọc theo ngày trong fact table dễ dàng và date dimension không không bị ảnh hưởng bởi SCD.
{% endhint %}

## Natural, Durable, and Supernatural Keys

Natural keys được tạo ra bởi source systems phải tuân thủ theo quy tắc nghiệp vụ, nằm ngoài khả năng kiểm soát của hệ thống DW/BI.&#x20;

Ví dụ trong môi trường bệnh viện, một bệnh nhân có thể được cấp một `patientID` mới khi mất hồ sơ, khi đổi hệ thống, hoặc khi đăng ký lại. Nếu DW chỉ dựa vào natural key, các lần khám này sẽ bị tách thành nhiều bệnh nhân khác nhau, làm sai lệch toàn bộ phân tích lịch sử điều trị.

Để tránh điều này, DW tạo ra durable key – một mã nhận diện ổn định, gắn với đúng một bệnh nhân trong suốt cuộc đời. Durable key không phụ thuộc vào cách hệ thống nguồn tạo mã bệnh nhân và thường chỉ là một số nguyên tăng dần do DW tự quản lý. Đôi khi, key này được gọi là durable supernatural key. Khi thông tin hồ sơ của bệnh nhân thay đổi theo thời gian, DW có thể sinh ra nhiều surrogate key để lưu lại từng phiên bản lịch sử (SCD). Tuy nhiên, durable key luôn giữ nguyên, đảm bảo DW nhận diện đúng “một con người thật”, bất kể natural key hay hồ sơ vận hành thay đổi bao nhiêu lần.

## Drilling Down

Drilling down là cách phân tích dữ liệu đơn giản và phổ biến nhất của business users. Nó đơn giản ám chỉ việc đi sâu chi tiết hơn vào dữ liệu hơn bằng cách thêm một thuộc tính dimension vào truy vấn đang có. Trong SQL, hành động “drill down” tương đương với việc thêm một cột dimension vào `SELECT` và thêm luôn cột đó vào `GROUP BY`.

{% hint style="info" %}
## Ví dụ:

Ban đầu xem doanh thu theo tháng, sau đó muốn xem chi tiết hơn theo ngày, hoặc theo sản phẩm, hoặc theo khu vực, thì chỉ cần thêm cột đó vào query. Đó là “drill down”.
{% endhint %}

Drill down không cần phải định nghĩa trước hierarchy hay drill path cố định. Người dùng có thể drill vào bất kỳ thuộc tính nào từ bất kỳ dimension nào nối với fact, miễn là có quan hệ.

## Degenerate Dimensions

**Degenerate Dimensions** là các dimension chứa mã định danh về nghiệp vụ nhưng không có bảng dimension riêng mà được lưu thẳng vào fact table nhằm tối ưu cho các câu truy vấn.

{% hint style="info" %}
## Ví dụ:

Khi một hóa đơn có nhiều dòng (line item), các dòng fact của line item sẽ chứa các khoá ngoại tham chiếu tới các dimension mô tả hoá đơn. Tuy nhiên, thông tin trên bàng dimension mô tả không có gì ngoài `invoice_number`, nhưng vẫn là một khóa dimension hợp lệ cho fact table của line item. Trong trường hợp này, `invoice_number` sẽ lưu trực tiếp trong bảng fact dưới dạng một degenerate dimension, và thừa nhận rõ ràng rằng nó không có bảng dimension tương ứng.&#x20;
{% endhint %}

## Denormalized Flattened Dimensions

Khi thiết kế dimensional model, bạn cần “kiềm chế” thói quen chuẩn hoá vốn quen thuộc ở hệ thống OLTP. Thay vì tách các quan hệ `1-n`  thành nhiều bảng nhỏ, hãy gộp thẳng những quan hệ đó thành nhiều cột trong cùng một dimension. Việc denormalize như vậy giúp mô hình dễ hiểu hơn và cho phép truy vấn phân tích chạy nhanh hơn rất nhiều, đây cũng là hai mục tiêu cốt lõi của dimensional modeling.

{% hint style="info" %}
## Ví dụ:

Trong hệ thống OLTP, thông thường theo relational modeling, ta thường thiết kế các mối quan hệ thành nhiều bảng khác nhau như:

```markdown
category(id, name)
subcategory(id, name, category_id)
product(id, name, subcategory_id)
```

Thay vì đó, dimensional modeling sẽ denormalize toàn bộ vào một dimension duy nhất:

```markdown
dim_product(product_sk, product_name, subcategory_name, category_name, brand_name)
```

Thực chất, tác giả đề cập nên chuyển mindset design theo star schema thay vì snowflake.
{% endhint %}

## Multiple Hierarchies in Dimensions

Một dimension có thể chứa nhiều hierarchy khác nhau cùng tồn tại trong một bảng - và điều này là hoàn toàn bình thường trong dimensional modeling, không cần phải tạo nhiều dimension tách riêng.

{% hint style="info" %}
## Ví dụ:

Location Dimension có thể có nhiều hierarchy song song:

**Hierarchy 1: City → State → Country -** Dùng cho hành chính.

**Hierarchy 2: City → Sales Region → Market Zone** - Dùng cho phân vùng kinh doanh.

**Hierarchy 3: City → Delivery Zone → Shipping Region** - Dùng cho logistics.

Tất cả các hierarchy này có thể cùng nằm trong bảng `dim_location` như:

<pre class="language-markdown" data-line-numbers><code class="lang-markdown"><strong>dim_location
</strong>-----------
city_name
state_name
country_name
sales_region
market_zone
delivery_zone
shipping_region
</code></pre>
{% endhint %}

## Flags and Indicators as Textual Attributes

Những cái viết tắt khó hiểu (`ACCT_TYP_CD`, `CUST_SEG = R1`, `ST = A`), những cột kiểu flags  `true`/`false`, `Y`/`N`, `1`/`0`, và những mã operation kiểu `A1B2C3` có ý nghĩa ngầm bên trong,... nên được bổ sung trong các dimension tables với full text chứa đầy đủ ý nghĩa.

{% hint style="info" %}
## Ví dụ:

Các cột `active_flag (Y/N`, `acct_type (SV/CK/CR)`, `status (A/D/H)` nên được bổ sung thông tin trên bảng dimension:

```markdown
| customer_sk | customer_id | active_flag | active_description    | acct_type | acct_type_description  | status_code | status_description  |
|-------------|-------------|-------------|-----------------------|-----------|------------------------|-------------|---------------------|
| 101         | C123        | Y           | Customer is active    | SV        | Savings Account        | A           | Approved            |
```
{% endhint %}

Nếu một mã code chứa nhiều thông tin, hãy tách ra từng phần:

{% hint style="info" %}
## Ví dụ:

Một mã code vị trí nhân viên `HR-SEA-FT` với ý nghĩa:

* `HR` = Human Resources
* `SEA` = Southeast Asia region
* `FT` = Full-time

Nên được tách thành các cột nhỏ hơn như `dept_code`, `region_code`, `employment_type_code` kèm các cột mô tả

```markdown
| employee_sk | employee_id | employee_position_code_original | dept_code | dept_description    | region_code | region_description         | employment_type_code  | employment_type_description  |
|-------------|-------------|---------------------------------|-----------|---------------------|-------------|----------------------------|-----------------------|------------------------------|
| 101         | E1123       | HR-SEA-FT                       | HR        | Human Resources     | SEA         | Southeast Asia Region      | FT                    | Full-time                    |
```
{% endhint %}

{% hint style="warning" %}
## Lưu ý:

Chỉ nên triển khai thêm các cột bổ sung nếu chính tên của cột flag/indicator không tự nói lên được ý nghĩa của chính nó.
{% endhint %}

## Null Attributes in Dimensions

Giá trị null trên các thuộc tính của bảng dimension có thể xảy ra do thiếu sót trên dữ liệu gốc hoặc thuộc tính đó không áp dụng đối với tất cả các rows. Nhưng dù là bất cứ lý do gì cũng không nên để `NULL` trên các thuộc tính này bởi vì:

1. Mỗi database xử lý `NULL` trong `GROUP BY` và `WHERE` khác nhau.
2. Việc join hoặc filter theo `NULL` dễ sinh bug.
3. Analyst, User nhìn `NULL` rất khó hiểu.

Thay vào đó, hãy mô tả các thuộc tính null này bằng các chuỗi mô tả rõ ràng hơn như `Unknown`, `Not Applicable`, `Missing`, `Unavailable`, `No Value Provided`, etc.

## Calendar Date Dimensions

Hầu hết fact table đều cần phân tích theo ngày, tháng, năm, fiscal period, holiday… nên _Calendar date dimensions_ gần như luôn được gắn vào mọi fact. Hơn nữa các ngày đặc biệt hoặc thuộc tính phức tạp (như Easter, Black Friday, fiscal week…) không nên tính bằng SQL, mà nên lookup trực tiếp từ bảng này vì đã chứa đầy đủ thông tin. Do đó, date dimension có nhiều (rất nhiều) thuộc tính (ví dụ `date`, `day_of_month`, `day_of_week`, `day_of_week_name`, `week_of_year`, `month`, `month_name`, `quarter`, `year`, `fiscal_week`, `fiscal_period`, `fiscal_year`, `is_holiday`, `is_special_event`, `is_weekend`, etc) nhằm giúp phân tích theo nhiều góc độ dễ dàng hơn.

Để dễ partition và filter,  surrogate key của date dimensions có thể dùng smart key dạng `YYYYMMDD` thay vì surrogate key tự tăng. Tuy nhiên dù dùng smart key, các thao tác filter và group-by nên dựa trên thuộc tính mô tả (`month_name`, `year`, `fiscal_period`, …) chứ không dựa trên smart key.

Tương tự như các dimension khác, date dimension tables cũng luôn cần một row đặc biệt cho giá trị “unknown” hoặc “to-be-determined”, thay vì để NULL.

Khi cần mức độ chi tiết hơn (giờ–phút–giây), fact table có thể chứa một timestamp raw, độc lập, không liên kết FK tới bất kỳ dimension nào hoặc analyst cần group theo time-of-day thì cần thêm time dimension như `day_part`, `shift_number`, `is_peak_hour`, ...

## Role-Playing Dimensions

Một bảng dimension vật lý (physical table) — ví dụ `dim_date` — có thể được fact table tham chiếu nhiều lần, mỗi lần với một ý nghĩa khác nhau. Tất cả tham chiếu này đều tớinsion chung một bảng dimension, nhưng mỗi “vai trò” phải được tách thành một view riêng (hoặc alias) với các tên cột khác nhau để không bị lẫn. Những role này chính là role-playing dimensions.

{% hint style="info" %}
## Ví dụ:

Trong `fact_order` có nhiều loại ngày nhưng với role khác nhau như "Order Date", "Ship Date", "Delivery Date", "Return Date". Khi đó trên bảng facts sẽ có các cột riêng `order_date`, `ship_date`, `delivery_date`, `return_date` vơi các tên khác nhau nhưng cùng tham chiếu tới cùng một bảng date dimension.
{% endhint %}

## Junk Dimensions

Trong các hệ thống, thường tồn tại rất nhiều miscellaneous, low-cardinality flags và indicators.  Thay vì tạo một loạt dimension bé tí, mô hình Kimball khuyên gom toàn bộ những flag và indicator này vào một bảng duy nhất, gọi là junk dimension (hoặc _transaction profile dimension_). Bảng này không cần sinh ra mọi tổ hợp giá trị theo kết quả của phép nhân Cartesian, mà chỉ chứa những tổ hợp thực sự xuất hiện trong dữ liệu nguồn.

{% hint style="info" %}
## Ví dụ:

Nếu dữ liệu giao dịch có ba thuộc tính nhỏ

```markdown
is_priority      (Y/N)
fraud_flag       (Y/N)
channel_type     (WEB / APP / STORE)
```

Thay vì tạo 3 bảng dimension riêng, ta tạo một `dim_transaction_profile`:

```markdown
| transaction_profile_sk | is_priority | fraud_flag | channel_type |
|------------------------|-------------|------------|---------------|
| 1                      | N           | N          | WEB           |
| 2                      | Y           | N          | APP           |
| 3                      | N           | Y          | STORE         |
| 4                      | Y           | Y          | WEB           |
```


{% endhint %}

## Snowflaked Dimensions

Khi một dimension có các cấp bậc (hierarchy) - ví dụ Category → Subcategory → Product - mà ta normalize, thì ta sẽ tách những thuộc tính low-cardinality thành các bảng con riêng, nối chúng về bảng chính bằng foreign key. Nếu trong dimension có nhiều hierarchy và ta đều normalize, thì mô hình sẽ tạo thành một cấu trúc nhiều tầng (multi-level). Cấu trúc này là snowflaked dimension.

Snowflake mô tả chính xác mối quan hệ phân cấp, nhưng lại không nên dùng trong dimensional modeling vì:

* Business user khó hiểu và khó dùng (nhiều bảng, nhiều join).
* Query chạy chậm hơn (join nhiều tầng).
* Schema phức tạp không cần thiết.

Trong khi đó, một dimension đã được làm phẳng (flattened, denormalized) chứa đầy đủ thông tin y hệt snowflake, nhưng được trình bày trong một bảng duy nhất nhưng dễ hiểu hơn, nhanh hơn, đúng tinh thần star schema.

{% hint style="info" %}
## Ví dụ:

**Snowflaked (normalized)**

```markdown
dim_product
    product_id
    product_name
    subcategory_id → dim_subcategory
                            subcategory_id
                            subcategory_name
                            category_id → dim_category
                                         category_id
                                         category_name
```

**Flattened dimension (denormalized)**

```markdown
dim_product
-----------
product_id
product_name
subcategory_name
category_name
```
{% endhint %}

<details>

<summary><i class="fa-clipboard-question">:clipboard-question:</i> Có phải lúc nào cũng phải tránh và tránh được Snowflaked Dimensions? Kỹ thuật nào dùng để chuyển từ Snowflaked sang Denormalized?</summary>



</details>

## Outrigger Dimensions

Một dimension có thể chứa reference (FK) sang một dimension khác. Khi điều này xảy ra, dimension đó được gọi là outrigger dimension. Kimball nói rằng outrigger không sai, vẫn _permissible_, nhưng nên hạn chế dùng, vì làm dimension phức tạp hơn, phá vỡ sự đơn giản của star schema và gây nhiều join không cần thiết.&#x20;

Thay vào đó, nếu 2 dimension có liên hệ với nhau, tốt nhất đưa cả 2 dimension vào fact table, mỗi dimension sẽ có FK riêng, để quan hệ thể hiện qua fact, không phải qua dimension bên trong dimension.





[^1]: Slowly Changing Dimension
