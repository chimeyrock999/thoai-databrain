# Dimension Hierarchy Techniques

Trong thực tế, dimension luôn có phân cấp ([hierarchy](basic-dimension-techniques.md#multiple-hierarchies-in-dimensions)). Nhìn bề ngoài nó có vẻ rất đơn giản, ví dụ Country - Region - City/Province - District. Nếu ngây thơ, ta nghĩ chỉ cần flatten vào một dimension là xong, nhưng thực tế thì không không sạch sẽ như thế. Phân cấp trong dimension có thể phải đối mặt với các vấn đề như sau:

1. **Ragged / Unbalanced Levels:**  Mỗi entity có số cấp khác nhau, nên không thể flatten vì số cột khác nhau.
2. **Missing Levels:** Một số entity bị thiếu cấp, dẫn đến roll-up bị sai hoặc bỏ sót cấp.
3. **Extra / Irregular Levels:** Một số entity có thêm cấp lạ không nằm trong cấu trúc chuẩn. Do đó không thể tạo được hierarchy chung cho toàn dimension.
4. **Multi-Parent Relationships:** Một cấp có thể có 2 hoặc nhiều cấp cha, gây duplicate rows, snowflake gây fan-out và double-count trong fact.
5. **Recursive / Unknown Depth:** Độ sâu phân cấp không cố định, lkhông thể định nghĩa level 1, level 2,… cố định.
6. **Hierarchy Changes Over Time (SCD):** Cấu trúc cấp thay đổi theo thời gian, dữ liệu lịch sử bị “kéo lệch” nếu không SCD-aware.
7. **Cycles / Loops (Dirty Data):** Dữ liệu nhập sai tạo vòng lặp làm query đệ quy treo, drill-down sai hoặc ETL bị kẹt.
8. **Grain Mismatch:** Fact grain không đủ chi tiết để phân tích ở level thấp hơn, drill-down bất khả thi dù dimension có chứa thêm level mới.

Và nhóm kĩ thuật trong chương này, được tác giả giới thiệu để sống sót trong thực tế phân cấp lộn xộn.

## Fixed Depth Positional Hierarchies

Fixed Depth Positional Hierarchy là kỹ thuật đơn giản nhất, ưu tiên nhất và sạch nhất trong tất cả các loại hierarchy Kimball đề cập. Nó được tin tưởng sử dụng khi:

* Cấp phân cấp cố định (fixed depth).
* Mỗi cấp có tên rõ ràng.
* Ít thay đổi theo thời gian.
* Quan hệ many-to-one sạch, mỗi cấp chỉ có 1 cha.

Cách triển khai loại này tương đối đơn giản, fatten tất cả vào một bảng. Tuy nhiên để đảm bảo hiệu quả thì nên đảm bảo [conformed dimension](integration-via-conformed-dimensions.md), thống nhất tên, số level, tránh null trên toàn bộ business process.

{% hint style="danger" %}
## Tránh sử dụng đặt tên trừu tượng cho Fixed Depth Positional Hierarchies

Kimball có lưu ý rằng không nên sử dụng các tên như `level_1` , `level_2`,... cho fixed depths vì chúng tối nghĩa. Chúng sẽ làm mất đi ý nghĩa cố định của từng level, BI user không thể hiểu, filter, hay drill-down đúng được.
{% endhint %}

## Slightly Ragged/Variable Depth Hierarchies

Slightly ragged - hơi rách rưới :joy:, ám chỉ dimension có phân cấp không cố định số level (variable depth), nhưng cũng không quá loạn, không đẹp đẽ nhưng không quá tệ. Tức là cấp của các entity hơi lệch nhau nhưng không quá lớn, vẫn thuộc cùng một loại hierarchy.

Đối với loại này Kimball lựa chọn Force-fit hierarchy vào positional columns thông qua các bước:

{% stepper %}
{% step %}
**Xác định max depth.**

Xác định entity có depth lớn nhất và lấy nó làm chuẩn.&#x20;

{% hint style="info" %}
Geographic hierarchy có 3 dạng depth như sau:

3 cấp: Country → Region → City

4 cấp: Country → State → County → City

5 cấp: Country → Province → City → District → Ward

Chúng không có đô sâu cấp cố định, tuy nhiên độ lệch nhỏ và chọn max depth là 5 làm chuẩn.
{% endhint %}
{% endstep %}

{% step %}
**Tạo cố định số cột positional tương ứng**

Sử dụng **positional attributes** đại diện cho level, thay vì đại diện cho tên thật của cấp. Ví dụ: `level_1`, `level_2` , `level_3` ,... hoặc `geo_level_1`, `geo_level_2`, `geo_level_3`,... hoặc `hier_level_1`, `hier_level_2`, `hier_level_3`,...

{% hint style="info" %}
Ở ví dụ trên, ta tạo một bảng có 5 cột positional attributes  `geo_level_x` nhằm đại diên cho 5 level.
{% endhint %}
{% endstep %}

{% step %}
**Điền các cấp vào bảng dimension**

Lần lượt map từng cấp từ trên xuống vào đúng vị trí dựa trên business rules, phần thừa dùng `NULL` hoặc placeholder.

{% hint style="info" %}
Với ví dụ trên, ta sẽ thu được bảng dimension như sau:

```
| geo_key | geo_level_1  | geo_level_2  | geo_level_3  | geo_level_4  | geo_level_5  |
|---------|--------------|--------------|--------------|--------------|--------------|
| 101     | CountryA     | RegionA      | CityA        | NULL         | NULL         |
| 102     | CountryB     | StateB       | CountyB      | CityB        | NULL         |
| 103     | CountryC     | ProvinceC    | CityC        | DistrictC    | WardC        |   
```
{% endhint %}
{% endstep %}
{% endstepper %}

## Ragged/Variable Depth Hierarchies with Hierarchy Bridge Tables

Khi phân cấp "jack" thực sự, ở đó:

* Số cấp khác nhau rất lớn giữa các entity (3 cấp, 5 cấp, 7 cấp...)
* Có cấp thiếu, cấp thừa, hoặc cấp lạ
* Một entity có thể thuộc nhiều cấp cha (multi-parent)
* Cấp có thể thay đổi theo thời gian
* Độ sâu hierarchy không biết trước (kiểu parent–child recursive)

Do đó ta không thể flatten positional, không thể dùng fixed-depth, không thể force-fit slightly ragged. Nếu cố flatten sẽ làm mất ý nghĩa của level, drill-down hỏng, fact double-count.

Khi đó, tác giả để xuất sử dụng **Hierarchy Bridge Tables** để giải quyết bài toán này. Cụ thể, Hierarchy Bridge Table là một bảng mô tả mọi quan hệ ancestor–descendant (không phải cha-con/parent-child đơn thuần) trong hierarchy. Hay nói cách khác, bảng này chứa một dòng cho mỗi đường đi có thể có trong cây phân cấp. Bằng cách này,  mọi dạng duyệt phân cấp (hierarchy traversal) đều có thể được thực hiện bằng SQL đơn thuần mà không cần dùng đến các extension hay công cụ đặc biệt.&#x20;

{% hint style="info" %}
## Haha, đọc không hiểu một cái gì hết :hear\_no\_evil:, thế thì qua ví dụ minh hoạ vậy.

Giả sử ta có một công ty với org chart như sau:

```
level_1: CEO
 ├─ level_2: VP Finance
 │   └─ level_3: Finance Manager
 │       └─ level_4: Accountant
 └─ level_2: VP Sales
     ├─ level_3: Regional Lead
     └─ level_3: Sales Rep
```

Nếu chỉ flatten thông tin employee trong dimension thì có kết quả như sau:

<pre><code><strong>employee_dim_positional
</strong>------------
| emp_key | emp_name | title            | level_1 | level_2     | level_3        | level_4     |
|---------|----------|------------------|---------|-------------|----------------|-------------|
|       1 | Alice    | CEO              | CEO     | NULL        | NULL           | NULL        |
|       2 | Bob      | VP Finance       | CEO     | VP Finance  | NULL           | NULL        |
|       3 | Carol    | Finance Manager  | CEO     | VP Finance  | Finance Mgr    | NULL        |
|       4 | David    | Accountant       | CEO     | VP Finance  | Finance Mgr    | Accountant  |
|       5 | Emma     | VP Sales         | CEO     | VP Sales    | NULL           | NULL        |
|       6 | Felix    | Regional Lead    | CEO     | VP Sales    | Regional Lead  | NULL        |
|       7 | Grace    | Sales Rep        | CEO     | VP Sales    | Sales Rep      | NULL        |
</code></pre>

Và ta nhận thấy hàng tá vấn đề như sau:

* **Drill-down / roll-up theo phòng ban rất khó và cồng kềnh:**
  * Muốn phân tích: "Doanh thu theo phòng ban trực thuộc CEO (bao gồm mọi cấp dưới)?" Khi đó phải join theo từng level, hoặc UNION nhiều điều kiện, hoặc self-join N lần.&#x20;
  * Khi số cấp tăng lên query cực kì đau đầu. Hậu quả dẫn đến BI cực kì khó, query dễ sai logic, maintain cực tệ.
* **Mở thêm phòng mới, schema sẽ vỡ trận ngay:**
  * Thêm 1 cấp mới dưới Finance thì phải thêm `level_5`.
  * Thêm team con dưới Sales thì phải ALTER TABLE.
  * Khi công ty tái cấu trúc tổ chức, mô hình sẽ phải "đập đi xây lại".-
* **Không hỗ trợ multi-parent (một phòng thuộc 2 nhóm báo cáo):**
  * Marketing báo cáo cho CEO + COO song song, trong positional không thể biểu diễn được.
* **Lịch sử thay đổi cấu trúc (SCD-hierarchy) không thể track:**
  * Nếu Finance tách thành Finance + Accounting vào năm 2025, nhưng fact cũ join vào level mới dẫn đến audit sai lịch sử.
* **Query chiều ngược (Lấy toàn bộ ancestor) gần như bất khả thi:**
  * "Sales Rep thuộc bộ phận nào, cấp cha nào, cha của cha là ai?" Trong positional thì phải CASE WHEN trên nhiều level, hoặc crawl từng cột.

Nhưng với Bridge Table ta sẽ mô tả tất cả mối quan hệ có thể có trong bảng như sau:

```
employee_hierarchy_bridge_adv
-------------
| ancestor_key | descendant_key | depth | weight | effective_from | effective_to | path_string                                      |
|--------------|----------------|-------|--------|----------------|--------------|--------------------------------------------------|
| 1            | 2              | 1     | 1.00   | 2020-01-01     | NULL         | CEO > VP Finance                                 |
| 1            | 3              | 2     | 1.00   | 2020-01-01     | NULL         | CEO > VP Finance > Finance Manager               |
| 1            | 4              | 3     | 1.00   | 2020-01-01     | NULL         | CEO > VP Finance > Finance Manager > Accountant  |
| 1            | 5              | 1     | 1.00   | 2020-01-01     | NULL         | CEO > VP Sales                                   |
| 1            | 6              | 2     | 1.00   | 2020-01-01     | NULL         | CEO > VP Sales > Regional Lead                   |
| 1            | 7              | 2     | 1.00   | 2020-01-01     | NULL         | CEO > VP Sales > Sales Rep                       |
| 2            | 3              | 1     | 1.00   | 2020-01-01     | NULL         | VP Finance > Finance Manager                     |
| 2            | 4              | 2     | 1.00   | 2020-01-01     | NULL         | VP Finance > Finance Manager > Accountant        |
| 5            | 6              | 1     | 1.00   | 2020-01-01     | NULL         | VP Sales > Regional Lead                         |
| 5            | 7              | 1     | 1.00   | 2020-01-01     | NULL         | VP Sales > Sales Rep                             |
```

Trong đó các cột có ý nghĩa như sau:

* `weight` : Optional - Phân bổ trách nhiệm nếu multi-parent.
* `effective_from`/`effective_to` : Optional - Phục vụ SCD-hierarchy theo thời gian.
* `path_string`: Optional - Phục vụ BI Visualization.

Khi đó để trả lời các câu hỏi, BI sẽ join facts và `employee_hierarchy_bridge_adv` trên các cột `ancestor_key` ,  `descendant_key` để thực hiện truy vấn phân cấp. Ví dụ để tính doanh thu của Alice (CEO) bao gồm toàn bộ cấp dưới ta sẽ join fact với bridge theo `descendant_key`, filter theo `ancestor_key = 1` , và tính `SUM`.

```sql
SELECT SUM(f.amount) AS total_revenue_under_alice
FROM fact_sales f
JOIN employee_hierarchy_bridge_adv b
     ON f.emp_key = b.descendant_key      -- nhân viên thực hiện sale
WHERE b.ancestor_key = 1;                 -- Alice (CEO)
```

Một số câu SQL khác như:

* Nếu muốn xem chi tiết từng cấp dưới của Alice:

```sql
SELECT d.emp_name, SUM(f.amount) AS revenue
FROM fact_sales f
JOIN employee_hierarchy_bridge_adv b
     ON f.emp_key = b.descendant_key
JOIN dim_employee d
     ON d.emp_key = b.descendant_key
WHERE b.ancestor_key = 1
GROUP BY d.emp_name
ORDER BY revenue DESC;
```

* Nếu muốn phân tích theo từng cấp (depth):

```sql
SELECT b.depth, SUM(f.amount) AS revenue_by_level
FROM fact_sales f
JOIN employee_hierarchy_bridge_adv b
     ON f.emp_key = b.descendant_key
WHERE b.ancestor_key = 1
GROUP BY b.depth
ORDER BY b.depth;
```

Nó còn support được nhiều tình huống phức tạp khác như "Alice rời vị trí, Bob lên CEO thì SCD hierarchy xử lý",...
{% endhint %}

Tóm lại Bridge Table rất rất mạnh - kiểu đọc tới đoạn này tớ phải thốt lên “ơ thế hoá ra làm được đến mức này à?”. Nó xử lý được depth biến thiên, drill-down ngược xuôi, thậm chí hỗ trợ cả SCD-hierarchy.\
Nhưng mạnh nào cũng có giá. Nó không hề dễ nhai, khả năng cao là bị nhai lại :smirk:. Ngay cả BI chưa chắc đã hiểu hay Modeler tớ nghĩ đôi khi còn phải gãi đầu, và thú thật tớ cũng chỉ hiểu nôm na là như thế chứ chưa kể đến mấy trick nâng cao phía sâu hơn.

Nếu cậu không để ý thì cái Bridge Table này có khả năng sẽ phình rất to, ETL chưa chắc đã dễ nữa, nói chung là không có gì ngon, bổ, rẻ hết.

{% hint style="info" %}
Với cái ví dụ trên, nếu công ty có `N` nhân viên thì trong trường hợp xấu thì số dòng trong bảng sẽ xấp xỉ `N*(N-1)/2` , chưa kể nếu implement SCD thì nó càng to nữa :joy:
{% endhint %}

## Ragged/Variable Depth Hierarchies with Pathstring Attributes

Đôi khi dùng Bridge Table hơi overkill thật, đâu phải business nào cũng cần phân tích tới mức ancestor–descendant nặng đô như vậy.\
Nếu mục tiêu chỉ là _xác định đường đi của node trong hierarchy_, không yêu cầu roll-up linh hoạt, không multi-parent, không historical lineage thì ta có thể dùng **Pathstring Attribute** thay vì Bridge.

Ý tưởng của Kimball như sau: Mỗi dòng trong dimension lưu một string mô tả toàn bộ tuyến phân cấp **(**&#x50;athstring Attribute)từ root > … > node. Sau đó sử dụng các toán tử `LIKE` / `REGEXP` / `SPLIT` để phân tích.

{% hint style="info" %}
## Quay lại với ví dụ trên

Với kĩ thuật này, ta có thể tạo bảng dimension như sau:

```
dim_employee_with_path
-------------
| emp_key | emp_name         | title            | path_string                    |
|---------|------------------|------------------|--------------------------------|
| 1       | Alice            | CEO              | CEO                            |
| 2       | Bob              | VP Finance       | CEO>Finance                    |
| 3       | Carol            | Finance Manager  | CEO>Finance>Manager            |
| 4       | David            | Accountant       | CEO>Finance>Manager>Accountant |
| 5       | Emma             | VP Sales         | CEO>Sales                      |
| 6       | Felix            | Regional Lead    | CEO>Sales>RegionalLead         |
| 7       | Grace            | Sales Rep        | CEO>Sales>SalesRep             |
```

Và với một số câu query phân tích:

*   **Lấy toàn bộ nhân viên dưới Finance:**

    ```sql
    SELECT emp_name, title
    FROM dim_employee_with_path
    WHERE path_string LIKE 'CEO>Finance%';
    ```
*   **Xác định level của một nhân viên (bằng số “>”):**

    ```sql
    SELECT emp_name,
           LENGTH(path_string) - LENGTH(REPLACE(path_string,'>','')) + 1 AS depth_level
    FROM dim_employee_with_path;
    ```
*   **Lấy doanh thu toàn bộ dưới CEO (JOIN bằng path):**

    ```sql
    SELECT SUM(f.amount) AS total_revenue
    FROM fact_sales f
    JOIN dim_employee_with_path d ON f.emp_key = d.emp_key
    WHERE d.path_string LIKE 'CEO%';
    ```
{% endhint %}

Ưu điểm của Pathstring:

* Không cần Bridge: Không thêm bảng, không tốn storage, ETL dễ hơn rất nhiều.
* Không cần nhiều join: Truy vấn đơn giản, BI dễ dùng.
* Dễ lưu trữ: Chỉ 1 cột string.

Nhưng nó vẫn yếu hơn Bridge khi thiếu sót:

* Không support được multi-parent.
* Không hỗ trợ SCD-hierarchy tốt.
* Không drill-up từ descendant về ancestor hiệu quả.

## My Summary

Có những thứ nhìn thì tưởng quen, tưởng đơn giản, nhưng đi sâu xuống mới thấy trước giờ mình hơi ngây thơ. Mỗi kỹ thuật trong phần này - kể cả chưa chạm tới advanced - đều làm tớ “wooooow” không chỉ một lần. Có cái nhẹ nhàng, có cái xoắn não, nhưng điểm chung là mở mắt rất nhiều.

Đúng là trước giờ nghe nói Kimball GOAT cũng chỉ gật gù cho vui. Đọc đến đoạn này mới thấy ờ… người ta nói không sai. Càng đi xuống càng rộng, càng nhức đầu, càng thấy ổng nghĩ xa và sâu đến mức khó tin. Một cái kỹ thuật tưởng chỉ là “phân cấp thôi mà” mà cũng có cả tá biến thể, trade-off, edge cases và cách giải khác nhau.

Thú thật đến đây tớ mới hiểu tại sao quyển này được gọi là kinh thánh — đọc mấy chương đầu tưởng nhẹ nhàng, tới phần hierarchy là não đúng kiểu _bùng nổ_ 😂&#x20;

Giờ thì không còn “nghe phong thanh” nữa — mà **tớ chính thức công nhận Kimball đúng là GOAT.**
