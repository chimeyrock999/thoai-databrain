---
description: >-
  Tham khảo từ cuốn sách The Data Warehouse Toolkit, 3rd Edition
  -https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/data-warehouse-dw-toolkit/
---

# Dimensional Modeling Techniques

## Thế nào là một Data Model tốt?

### 1. **Tính Dễ Hiểu (Usability)**

* **Mục tiêu:** Người dùng BI có thể tự do khám phá dữ liệu mà không cần IT hỗ trợ.
* **Kỹ thuật đáp ứng:**
  * **Star Schema:** Sử dụng Star Schema với 1 fact table ở trung tâm + nhiều dimension tables xung quanh. Ví dụ: `fact_sales` kết nối với `dim_product`, `dim_time`.
  * **Denormalization:** Gom các thuộc tính phân cấp (ví dụ: category → subcategory → product) vào cùng dimension table, tránh snowflake schema phức tạp.
  * **Đặt tên rõ ràng:** Sử dụng tên cột dễ hiểu (vd: `product_name` thay vì `prod_nm`).

### 2. **Hiệu Suất Truy Vấn (Performance)**

* **Mục tiêu:** Truy vấn trả kết quả nhanh ngay cả với dataset lớn.
* **Kỹ thuật đáp ứng:**
  * **Partitioning:** Chia fact table theo time\_key (vd: partition theo tháng).
  * **Indexing:** Tạo index trên khóa ngoại (foreign keys) và cột thường dùng để filter/group.
  * **Aggregate Tables:** Xây dựng bảng tổng hợp (ví dụ: tổng doanh thu theo tháng) song song với atomic data.

### 3. **Tính Nhất Quán (Consistency)**

* **Mục tiêu:** Dữ liệu từ các quy trình nghiệp vụ khác nhau có thể tích hợp chính xác.
* **Kỹ thuật đáp ứng:**
  * **Conformed Dimensions:** Sử dụng dimension chuẩn hóa (vd: `dim_date` dùng chung cho sales và inventory).
  * **Enterprise Bus Architecture:** Thiết kế matrix liên kết các business processes với dimensions chung.

### 4. **Khả Năng Mở Rộng (Scalability)**

* **Mục tiêu:** Dễ dàng thêm metrics/dimensions mới mà không phá vỡ cấu trúc hiện có.
* **Kỹ thuật đáp ứng:**
  * **Atomic Grain:** Lưu dữ liệu ở mức chi tiết nhất (vd: mỗi dòng `fact_sales` tương ứng 1 sản phẩm trong 1 giao dịch).
  * **Surrogate Keys:** Dùng khóa tự tăng (integer) thay vì natural keys để xử lý SCD và tích hợp đa nguồn.

### 5. **Xử Lý Thay Đổi (Change Management)**

* **Mục tiêu:** Theo dõi lịch sử thay đổi của dimension attributes (vd: địa chỉ khách hàng).
* **Kỹ thuật đáp ứng:**
  * Slowly Changing Dimension (SCD)

### 6. **Tối Ưu Storage**

* **Mục tiêu:** Giảm chi phí lưu trữ mà vẫn đảm bảo hiệu suất.
* **Kỹ thuật đáp ứng:**
  * **Sparse Storage:** Dùng SparseVector cho one-hot encoded columns (chỉ lưu indices/values khác 0).
  * **Columnar Format (Parquet/ORC):** Nén dữ liệu hiệu quả, tối ưu cho analytical workloads.

## Các bước để xây dựng một Dimension Model:

### 1. Xác định Business Process

Business process là một hoạt động cấp thấp (low-level activity) thực hiện bởi một tổ chức, ví dụ nhận đơn đặt hàng, lập hoá đơn, xử lý thanh toán, thực hiện thủ tục y tê, xử lý khiếu nại, xử lý yêu cầu bồi thường,... Để xác định được business process, bạn cầu hiểu được một số đặc điểm thường thấy của nó như sau:

* **Business process thường được diễn tả bằng động từ:** Hiểu đơn giản, business process là một hành động mà doanh nghiệp làm. Và một cách tự nhiên, nó được mô tả bằng động từ. Trong đó các dimensions tương ứng sẽ mô tả ngữ cảnh của hành động đó.
* **Business process thường được hỗ trợ bởi hệ thống vận hành:** Một process trong doanh nghiệp thường chạy ở 1 hệ thống vận hành và đó là nơi sinh ra các events.
* **Business process tạo ra hoặc ghi nhận các metrics:** Mỗi khi một process chạy, nó sinh ra số liệu mà business muốn phân tích. Đôi khi metrics là sẵn _có_ trong transaction, đôi khi metrics phải derive từ data. Và điều quan trọng, Analyst luôn muốn slice & dice số liệu theo vô hạn cách do đó việc xác định và tổ chức dimension là cực kì quan trọng.
* **Business process có input và output:** Một process A được kích hoạt bởi input, chạy, rồi sinh output. Output đó có thể trở thành input cho process B. Tất cả tạo thành một chuỗi bảng facts, không và không thể gượng ép tất cả vào một bảng fact duy nhất.

{% hint style="info" %}
## **Ví dụ**

* Các business process Place Order (đặt hàng), Ship Product (giao hàng), Issue Invoice (xuất hóa đơn) Receive Payment (nhận thanh toán) đều được mô tả bằng động từ. Với Place Order sẽ có các dimension đồng hành mô tả context như:
  * Khách nào? (Customer dimension)
  * Sản phẩm nào? (Product dimension)
  * Kênh nào? (Channel dimension)
  * Salesperson nào? (Employee dimension).
* Process billing được vận hành bởi hệ thống lập hóa đơn, purchasing bởi hệ thống mua hàng, Logistics bởi hệ thống giao vận, CRM bởi hệ thống chăm sóc khách hàng. Và những hệ thống này sẽ là nơi sản sinh transactional data và được xem là các data sources.
* Khi vận hành order process thì sẽ tạo ra các measure facts order amount, order quantity;  payment process tạo ra payment amount; shipment process tạo ra shipping cost, shipping time,... Metric order amount có sẵn tuy nhiên profit = revenue – cost phải tính toán từ data.&#x20;
* Khi customer places order sẽ sinh ra order facts, warehouse ships products sẽ  sinh shipping facts, Accounting tạo khoá đơn sinh invoice facts, Finance nhận thanh toán tạo ra payment facts.
{% endhint %}

#### Quy trình xác định Business Process

{% stepper %}
{% step %}
**Thu thập yêu cầu từ stakeholders (Business Requirements)**

Lắng nghe pain points từ user: muốn theo dõi gì? chỉ số nào? mục tiêu gì?&#x20;

Ghi nhận các key metrics và câu hỏi kinh doanh thường gặp.
{% endstep %}

{% step %}
**Xác định các hành động nghiệp vụ cốt lõi (Operational Business Events)**

Tập trung vào các hành động thực sự xảy ra trong hệ thống. Ví dụ: đặt hàng, thanh toán, click vào quảng cáo, gửi email...&#x20;

Tránh mô hình hóa khái niệm trừu tượng hoặc báo cáo tĩnh.
{% endstep %}

{% step %}
**Đối chiếu với nguồn dữ liệu hiện có (Data Realities)**

Kiểm tra xem quy trình đó có được ghi lại trong hệ thống không? Nếu có log/event/data, có thể tiếp tục. Nếu chưa có, cần chuẩn bị pipeline.
{% endstep %}

{% step %}
**Lập** [**Bus Matrix**](#user-content-fn-1)[^1]

Tiến hành lập [Bus Matrix](integration-via-conformed-dimensions.md#enterprise-data-warehouse-bus-matrix), nó giúp bạn xác định:

* Mỗi business process có fact table nào?
* Mỗi fact table dùng những dimension nào?
{% endstep %}

{% step %}
**Lựa chọn ưu tiên các Business Process**

Từ bus matrix, ưu tiên chọn quy trình quan trọng nhất hoặc dễ làm trước. Các quy trình có giá trị phân tích cao hơn, có dữ liệu đầy đủ, dễ triển khai và ít thay đổi trong thời gian ngắn nên được ưu tiên hơn.
{% endstep %}
{% endstepper %}

#### Khi Business Process thay đổi?

Business process có thể thay đổi bởi một số lý do như do nhu cầu mới của business, do điều chỉnh hệ thống nghiệp vụ (workflow, hệ thống ERP thay đổi...), do yêu cầu mở rộng phân tích (theo dõi thêm hành vi mới),...

Với dimensional modeling, có nhiều kỹ thuật để thích ứng: tạo fact table mới khi process mới xuất hiện, tạo version fact khi cách đo lường thay đổi, dùng snapshot fact khi logic trở thành state-based, áp dụng bridge table khi quan hệ phức tạp hơn, hoặc thêm mini-dimension/junk dimension nếu thuộc tính mô tả thay đổi. Đây chính là lý do cần sử dụng dimensional modeling thay vì freestyle.\
Tuy nhiên, kĩ thuật thì kĩ thuật nhưng việc business thay đổi đột ngột thì cũng khó lòng đáp ứng, lúc đó chỉ còn phương án là tổ chức lại mô hình thôi.

### 2. Xác định độ chi tiết (Declare the Grain).

Grain là tuyên bố rõ ràng và cụ thể, là convention rằng mỗi record trong bảng fact đại diện cho điều gì. Xác định grain là việc chỉ định chính xá một hàng trong bảng fact này tương ứng với event nào, ở độ chi tiết nào.

{% hint style="info" %}
## Ví dụ

* “Một row là một lần quét barcode sản phẩm”
* “Một row là một dòng trên hóa đơn”
* “Một row là một hóa đơn”
* “Một row là một giao dịch thanh toán”
* “Một row là lượng hàng tồn kho ghi nhận cuối mỗi ngày”
{% endhint %}

Grain là quy định của thiết kế mà mọi operation khác đều phải tuân thủ theo grain này, không được quá trừu tượng hoặc quá chi tiết. Khi đó, mọi dimension và measure sau đó phải phù hợp với grain đã định nghĩa.

Việc chọn grain là trade-off, grain càng chi tiết, khả năng ít bị thay đổi và mở rộng sau này dễ dàng hơn. Bên cạnh đó, grain chi tiết cũng giúp tính toán và phân tích linh hoạt hơn. Tuy nhiên khi grain quá chi tiết sẽ khiến Data Volume tăng cao, đồng thời tốn nhiều effort và resource để ingest và transform data. Kimball khuyên rằng: "You should develop dimensional models representing the most detailed, atomic information captured by a business process". Phần nhiều lý do là chi phí lưu trữ càng ngày càng rẻ, nó không đủ để đánh đổi khả năng mở rộng và phân tích.&#x20;

Khi chọn grain cũng cần xem xét mức độ phù hợp vào hiện trạng thực tế của dữ liệu bằng cách đặt các câu hỏi như:

* Dữ liệu hiện tại có đủ detail để ghi nhận grain này không?&#x20;
* Nếu không sẽ cần phải làm gì? Thu thập thêm event hay điều chỉnh Grain?

#### Quy trình xác định Grain

{% stepper %}
{% step %}
**Xác định business event gốc trong thực tế**&#x20;

Dựa trên business process đã được chọn, cần xác định các event xảy ra trong hệ thống. Grain sẽ được ưu tiên là event nhỏ nhất xảy ra trong hệ thống.
{% endstep %}

{% step %}
**Đọc dữ liệu source để xem hệ thống ghi ở mức nào**

Grain bắt buộc tuân theo “data realities”, nghĩa là log/transaction ghi đến đâu thì grain chỉ đến đó.
{% endstep %}

{% step %}
**Viết grain statement bằng 1 câu mô tả business**

Một câu duy nhất, rõ nghĩa mô tả Grain. Vì grain là tuyên bố quan trọng nhất trong hệ thống.&#x20;

{% hint style="info" %}
## Ví dụ:

* “One row per order line”
* “One row per payment transaction”
* “One row per daily inventory snapshot per product per warehouse”
{% endhint %}
{% endstep %}

{% step %}
**Kiểm tra độ phù hợp của grain với nhu cầu phân tích**

So sánh grain đã chọn có phù hợp với nhu cầu phân tích hay không. Nếu có, tiếp tục. Nếu không, chọn lại grain.

{% hint style="info" %}
## Ví dụ:

Khi chọn grain là từng đơn đặt hàng (order header), trong khi business yêu cầu phân tích đến từng sản phầm (order item), ta nên xem xét chọn lại grain cho phù hợp&#x20;
{% endhint %}
{% endstep %}

{% step %}
**Kiểm tra lại mức độ phù hợp của các metrics**

Xác nhận lại những metrics nào còn phù hợp với grain đã chọn, nếu không hãy loại bỏ chúng.

{% hint style="info" %}
## Ví dụ:

Sau khi chọn grain là order items, thì metrics `total_revenue` (lợi nhuận của đơn hàng) là không hợp lý, cần loại bỏ và tính toán sau.
{% endhint %}
{% endstep %}

{% step %}
**Kiểm tra xem dimension nào khớp grain**

Tương tự metrics, các dimension cũng cần xem xét lại có phù hợp với grain nữa hay không.
{% endstep %}
{% endstepper %}

#### Khi grain thay đổi?

Về cơ bản, grain thay đổi có tác động rất lớn, chỉ đứng sau thay đổi business process. Nguyên nhân có thể đến từ nhiều lý do như business hiểu sai hoặc mô hình hiểu sai ngay từ đầu, business process bổ sung chi tiết mới mà source thật sự ghi nhận, nhu cầu phân tích thay đổi theo chiều ngược lại

Khi grain thay đổi, mọi thứ trong fact table gần như phải xem xét lại từ đầu. Grain là “độ phân giải” của fact table, nên chỉ cần đổi grain thì dimension, fact, và cả mục đích phân tích đều bị ảnh hưởng.

### 3. Xác định Dimensions

Khi đã chốt grain, bước tiếp theo là xác định các dimension bằng cách hỏi: “Business mô tả sự kiện này bằng những thông tin nào?” Những dimension này thường trả lời các câu hỏi quen thuộc như who, what, where, when, why, và how để bao quát toàn bộ bối cảnh của event. Mỗi dimension gồm các thuộc tính mô tả dạng text giúp fact table trở nên có ý nghĩa và phân tích được. Điều quan trọng là dimension phải phù hợp với grain; nếu một dimension không gắn với mức chi tiết của event thì cần loại bỏ. Sau khi chọn dimension, hãy liệt kê đầy đủ các attribute để mô tả sự kiện rõ và giàu ngữ cảnh nhất.

{% content-ref url="basic-dimension-techniques.md" %}
[basic-dimension-techniques.md](basic-dimension-techniques.md)
{% endcontent-ref %}

### 4. Xác định Facts

Khi đã rõ grain, fact chính là những số liệu mà quy trình đang đo lường, chẳng hạn như quantity, amount hay cost. Các fact phải hoàn toàn phù hợp với grain; nếu metric thuộc mức chi tiết khác thì phải đặt vào fact table riêng. Fact thường là số liệu có khả năng tổng hợp để phục vụ phân tích. Việc chọn fact cần dựa cả vào nhu cầu phân tích của business và dữ liệu thực tế đang có, tránh mô hình hóa chỉ dựa trên source data. Dimensional model cuối cùng là kết quả của việc cân bằng hai yếu tố này.

{% content-ref url="basic-fact-techniques.md" %}
[basic-fact-techniques.md](basic-fact-techniques.md)
{% endcontent-ref %}

## My Summary

Quy trình bốn bước của Kimball thực chất là một vòng lặp có tính điều chỉnh liên tục, chứ không hẳn là một chuỗi tuyến tính đi từ điểm A đến B rồi dừng lại. Mặc dù thứ tự chính thức là: chọn business process, khai báo grain, chọn dimension rồi xác định fact, nhưng trong thực tế các bước này ảnh hưởng qua lại và buộc phải được xem xét nhiều lần trước khi mô hình thật sự ổn. Chỉ cần một bước không khớp - grain quá thô, dimension không ăn khớp với event, hoặc fact không thuộc đúng mức chi tiết - thì phải quay lại bước trước để chỉnh lại. Điều này là bình thường và là một phần quan trọng của phương pháp luận dimensional modeling.

Mặc dù Kimball đặt Bus Matrix ở bước chọn dimension, việc chủ động phác thảo Bus Matrix sớm hơn - thậm chí ngay từ lúc gom business process - giúp ta dễ nhìn toàn cảnh hơn: quy trình nào đang tồn tại, dimension nào có thể dùng chung, mối liên hệ giữa các process ra sao, và đâu là dòng dữ liệu cần ưu tiên. Việc này không chỉ giúp xác định business process chính xác mà còn giúp grain và dimension hiện ra rõ ràng hơn khi ta cần ra quyết định.

Toàn bộ quy trình thiết kế dimensional model luôn đòi hỏi sự cân bằng giữa hai yếu tố: nhu cầu phân tích của business và thực tế dữ liệu mà hệ thống có thể cung cấp. Business yêu cầu có thể rất rộng hoặc rất chi tiết, nhưng hệ thống nguồn chỉ ghi log đến một mức độ nhất định; ngược lại, hệ thống có thể có quá nhiều dữ liệu chi tiết nhưng business chỉ cần phân tích ở một mức thích hợp. Mỗi quyết định về grain, dimension, và fact đều phải xét song song hai yếu tố này để mô hình vừa hữu dụng, vừa khả thi. Chính vì vậy, quá trình thiết kế thường không tuân theo một đường thẳng mà là một chu kỳ tinh chỉnh: hiểu nhu cầu → xem dữ liệu → đề xuất mô hình → quay lại hỏi business → điều chỉnh cho phù hợp → lặp lại đến khi mọi phần ăn khớp tự nhiên.

[^1]: Bus Matrix trong Kimball là _một bảng (matrix) thể hiện mối quan hệ giữa business processes và dimensions_.

    Nó là tài liệu thiết kế cốt lõi của kiến trúc Data Warehouse theo Kimball.
