# Retail Sales

## Bối cảnh

Hãy tưởng tượng bạn làm việc tại headquaters của một chuỗi cửa hàng tiện lợi lớn.Doanh nghiệp này có 100 cửa hàng khắp năm tiểu bang. Mỗi cửa hàng có đầy đủ các gian hàng, bao gồm hàng tạp hóa, thực phẩm đông lạnh, sữa, thịt, nông sản, bánh mì, đồ uống và các sản phẩm hỗ trợ sức khỏe/sắc đẹp. Mỗi cửa hàng có khoảng 60.000 sản phẩm riêng lẻ, được gọi là _stock keeping units_ (SKU), trên kệ.

Dữ liệu được thu thập chủ yếu tại hai điểm: quầy thu ngân, nơi hệ thống POS quét mã vạch từng sản phẩm để ghi nhận giao dịch bán hàng; và cửa sau, nơi nhà cung cấp giao hàng vào kho.

<figure><img src="../../.gitbook/assets/image.png" alt="" width="267"><figcaption><p>Mẫu biên lai thu ngân</p></figcaption></figure>

Mục tiêu kinh doanh xoay quanh việc dự báo nhu cầu, tối ưu tồn kho, bán được nhiều hàng nhất và tối đa hóa lợi nhuận. Pricing và promotion là hai yếu tố tác động mạnh đến doanh số: giảm giá tạm thời, quảng cáo, trưng bày hàng, hoặc coupon đều có thể đẩy volume bán tăng mạnh, nhưng cũng dễ khiến biên lợi nhuận giảm. Vì vậy, khả năng phân tích tác động của giá và các hình thức khuyến mãi là trọng tâm với cả cửa hàng lẫn trụ sở.

## Quy trình 4 bước

### 1. Lựa chọn Business Process

Trong case study này, ban quản lý muốn hiểu rõ hơn về hoạt động mua hàng của khách hàng, được ghi nhận bởi hệ thống máy POS. Vì vậy business process cần modeling là **POS retail sales transactions**. Dữ liệu này cho phép business users phân tích **sản phẩm nào** đang bán ở **cửa hàng nào**, vào **ngày nào**, điều kiện **khuyến mại nào**, **giao dịch nào**.

### 2. Xác định Grain

Sau khi xác định business process, việc quan trọng nhất là quyết định grain - mức độ chi tiết của dữ liệu trong fact table.

Kimball khuyến nghị luôn dùng **atomic grain**, tức mức chi tiết nhỏ nhất mà hệ thống ghi nhận, vì nó mở ra nhiều dimension và cho khả năng phân tích linh hoạt tối đa. Lý do như sau:

* Atomic data có thể dễ dàng tổng hợp lên các mức cao hơn, nhưng dữ liệu tổng hợp thì không thể tái tạo lại chi tiết; chọn grain cao quá sẽ khiến người dùng bị hạn chế khi cần drill-down.
* Mô hình dùng aggregated grain có thể hỗ trợ performance, nhưng không bao giờ thay thế được atomic fact vì nó mất thông tin, mất dimension và dễ không đáp ứng yêu cầu phân tích bất ngờ.
* Một hiểu lầm phổ biến là dimensional modeling chỉ phù hợp với dữ liệu summary; thực tế, mô hình hoạt động tốt nhất khi chứa dữ liệu chi tiết nhất.

Trong case này, dữ liệu chi tiết nhất là **một sản phẩm riêng lẻ trong một POS transaction**, giả định rằng POS transaction sẽ tổng hợp tất cả doanh số của một sản phẩm trên một item line. Cho dù nhu cầu phân tích của business user không tới mức đó, nhưng bạn cũng không thể dự đoán được tương lai người ta có mong muốn phân tích theo chiều này hay không.

### 3. Nhận dạng Dimensions

Khi grain đã chốt rõ ràng, việc xác định dimension trở nên khá tự nhiên. Từ grain “POS line item”, các dimension chính như **product**, **transaction**, **date**, **store**, **promotion**, **cashier**, và **payment method** xuất hiện gần như ngay lập tức vì chúng mô tả trực tiếp sự kiện bán hàng.  Các dimension này đại diện cho các góc mô tả quen thuộc của một event: **what** (product), **where** (store), **when** (date), **who** (cashier/customer), **why** (promotion), và **how** (payment method).

Một nguyên tắc quan trọng: grain quy định “độ phân giải” của fact table, và chỉ những  dimension nào **mang duy nhất một giá trị** cho mỗi sự kiện ở grain đó mới được phép gắn vào fact. Nếu một dimension tiềm năng gây ra việc phải nhân đôi fact row để gắn dữ liệu, nghĩa là dimension đó vi phạm grain.  Trong trường hợp như vậy, có hai lựa chọn: (1) loại bỏ dimension đó khỏi fact, hoặc (2) quay lại xem lại grain vì có thể grain ban đầu chưa phản ánh đúng sự kiện thực tế.

{% hint style="info" %}
## Ví dụ: Promotion dimension gây nhân đôi fact row.

Một order có 2 dòng:

```
| order_id | line_no | product | qty |
| -------- | ------- | ------- | --- |
| 1001     | 1       | A       | 1   |
| 1001     | 2       | B       | 2   |
```

Với grain là một item line trong hóa đơn thì những dimension chỉ mang 01 giá trị duy nhất cho mỗi line như:

* Product: mỗi line có 1 product
* Date: mỗi line có 1 sale date
* Store: mỗi line thuộc 1 store
* Cashier: mỗi line là do 1 cashier xử lý&#x20;

&#x20;Chúng là các dimension phù hợp với grain.

Ngược lại, giả sử dòng sản phẩm này áp dụng 2 promotion đồng thời là Black Friday và Coupon giảm 10%. Khi đó việc đưa promotion dimension vào đây sẽ làm duplicate facts row:

```
| order_id | line_no | product | promotion |
| -------- | ------- | ------- | --------- |
| 1001     | 1       | A       | Promo 1   |
| 1001     | 1       | A       | Promo 2   |
```

Do đó promotion dimension trong trường hợp này là không phù hợp với grain, nên được loại bỏ khỏi facts. Và phương án có thể là  xử lý bằng Bridge table, hoặc Fact phụ về promotion, hoặc tách promotion thành attribute khác phù hợp với grain,... hoặc quay lại xem lại grain đã chọn đúng hay chưa.
{% endhint %}

Sau khi đã chốt danh sách dimension hợp lệ, mới bắt đầu bước “fleshing out” - liệt kê toàn bộ các thuộc tính mô tả bên trong từng dimension. Nhưng trước khi làm điều này, cần hoàn tất bước chọn fact để không bị sa đà vào chi tiết mà quên tổng thể mô hình.

{% hint style="info" %}
## &#x20;`transaction_number` là [degenerate dimension](../dimensional-modeling-techniques/basic-dimension-techniques.md#degenerate-dimensions).

Trong các dimension tương ứng lúc này, `transaction_number` là dimension chỉ lưu thông tin của order mà không có attribute đi kèm. Nó sẽ được lưu trực tiếp vào bảng fact dưới dạng degenerate dimension.
{% endhint %}

### 4. Xác định Facts









<br>





<br>
