# RDD vs DataFrame vs Dataset

## Resilient Distributed Dataset (RDD)

Resilient Distributed Dataset (RDD) là abstraction cơ bản nhất trong Spark. Mỗi RDD gồm nhiều partitions, mỗi partitions là tập hợp các Scala/Java object cụ thể. Dữ liệu trong RDD được lưu ở JVM heap.

{% hint style="info" %}
## Object Management in Python & Java

**Java (JVM) quản lý heap bằng object graph**: các object trên heap liên kết với nhau qua các reference tạo thành một graph. GC sẽ bắt đầu từ các **GC Roots**, lần theo các liên kết này để xem object nào còn reachable; phần không reachable thì bị thu hồi.

**Python (CPython)** **dùng chủ yếu reference counting:** mỗi object có refcount và khi refcount về 0 thì được giải phóng ngay. Do refcount không xử lý được vòng tham chiếu, Python có thêm _cycle GC_ để phát hiện và dọn các cycle không còn được tham chiếu từ bên ngoài.
{% endhint %}

Điều này dẫn đến một số hệ quả với hệ thống big data như Spark:

* Một job có thể sinh ra hàng triệu object nhỏ: Object Java/Scala có overhead riêng như class pointer, object header, padding rất tốn RAM.
* JVM phải quét toàn bộ graph để tìm object "rác":  Việc này gây GC overhead nghiêm trọng.
* Nếu app sinh nhiều object tạm (intermediate) dễ bị full GC, pause, ảnh hưởng đến hiệu năng. GC trong JVM có thể gây "Stop-the-World"
  * JVM Garbage Collector (GC) sẽ tạm dừng tất cả các thread khi dọn dẹp heap.
  * Dù có loại GC hiện đại như G1GC hay ZGC giảm thời gian dừng, vẫn sẽ có pause (dù ngắn).
* Tổ chức lưu dữ liệu giống row-based nên tính toán vectorized (SIMD[^1]) không hiệu quả
* JVM sinh ra cho ứng dụng long-running, ít churn object. Tuy nhiên Spark, đặc biệt là RDD API, tạo, biến đổi và hủy hàng tỷ object theo từng job. Điều này đi ngược với kiểu workload lý tưởng cho GC dẫn đến hiệu năng kém.

## DataFrame

DataFrame là tập dữ liệu có schema như bảng SQL. Do DataFrame có schema rõ ràng nên Spark có thể áp dụng một số tính năng đặc biệt như Catalyst Optimizer và Tungsten Engine để tối ưu. Dữ liệu trên DataFrame được lưu dưới dạng columnar và off-heap, không phụ thuộc vào JVM.

{% hint style="info" %}
## Row-based vs Column-based

**Row-based:** Các trường trong 1 hàng được lưu cạnh nhau. Khi truy cập sẽ cần load toàn bộ dòng, nhanh và dễ thao tác hơn với từng row, phù hợp với OLTP.

**Column-based:** Các cột sẽ được lưu trên các ô nhớ cạnh nhau. Khi truy cập sẽ load toàn bộ cột, tối ưu cho các tính toán OLAP trên các cột, phù hợp với OLAP.
{% endhint %}

Cụ thể hơn:

* Chính vì lưu dưới dạng column-based nên phải biết trước schema. Biết trước schema nên dễ tối ưu bằng Catalyst Optimizer. Và sử dụng (CPU) cache hiệu quả hơn (chỉ load column cần tính thay vì load toàn bộ, tối ưu hơn cho CPU cache).
* Thực ra Java về cơ bản rất bảo thủ, tuy nhiên vẫn có phương án thoát ra khỏi sự kiểm soát của JVM và Spark đã sử dụng Tungsten để làm điều đó. Về cơ bản, Tungsten sẽ tự kiểm soát cấp phát bộ nhớ như C/C++ thông qua UnsafeAPI, tránh depend vào JVM và overhead GC.
* Bên cạnh đó Tungsten và Catalyst Optimizer là thứ đứng sau Lazy Evolution của Spark. Catalyst Optimizer sẽ tính toán, tối ưu execution plan, Tungsten sẽ tự generate bytecode thành 1 hàm duy nhất, chạy liền mạch, tránh gọi function thông thường (chậm do object reference, GC, function call overhead).

## Dataset

Dataset là một lớp trừu tượng dữ liệu trong Spark, kết hợp được cả type safety của RDD và tối ưu hóa của DataFrame. Tuy nhiên, DataSet hỉ có ở Scala và Java API, không hỗ trợ ở Python (PySpark). Đồng thời, Dataset cũng có khả năng tận dụng Catalyst và Tungsten, nên nó cũng được optimize như DataFrame.

Nguyên nhân thực sự đằng sau là:

* Thực ra Dataset là sự kết hợp của DataFrame và Typed Object, đây cũng chính là lý do mà chỉ support được Scala/Java vì chúng là static type chứ không hẳn là dynamic type như Python.
* DataFrame mặc dù biết schema nhưng vẫn cấp phát các kiểu dữ liệu nguyên thuỷ dưới dạng Object nên có thể support Python, còn Dataset thì sử dụng kiểu của Scala/Java nên không có support được Python.

| DataFrame                                                                      | Dataset                                                                               |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| Mỗi field là Object (vì phải support động cho Python).                         | Dùng Encoder, mỗi field sẽ được giữ nguyên là kiểu nguyên thủy nếu có thể, zero copy. |
| Khi xử lý phải copy reference nếu mutate, và Java GC phải quét nhiều node hơn. | (De)serialization dùng codegen nên rất nhanh.                                         |
| Truy cập field sẽ qua hàm `get(i)`, rồi cast về kiểu cụ thể.                   | Truy cập field trực tiếp, không cần casting nên tối ưu CPU cache và memory access.    |

{% hint style="info" %}
## Dynamic Type vs Static Type in Python & Java

Sự khác nhau ở đây phần lớn chính là sự khác biệt giữa ngôn ngữ biên dịch và thông dịch (của Java/Scala và Python - mặc dù Java/Scala biên dịch qua byte code và tối ưu byte code bằng JIT nhưng vẫn là static type).
{% endhint %}

{% hint style="info" %}
## PySpark

Về cơ bản PySpark vẫn phải chạy trên nền JVM, PySpark chỉ là interface giao tiếp với JVM qua Py4J gateway. Nghĩa là PySpark code vẫn được biên dịch thành byte code (\*.pyc ) và PVM sẽ thực thi từng dòng trên byte code, byte code này sẽ call qua JVM qua Py4J và nhận về kết quả.
{% endhint %}

## So sánh

| RDD                                                                                                     | DataFrame / Dataset                                                                             |
| ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Không yêu cầu biết trước schema                                                                         | Phải biết schema trước để tối ưu                                                                |
| Không có tối ưu gì - user tự kiểm soát logic, memory, serialization                                     | Tối ưu mạnh nhờ Catalyst (query plan) và Tungsten (bytecode + memory mgmt)                      |
| Phù hợp với dữ liệu phi cấu trúc (logs, JSON động, custom parsing...) hoặc khi logic xử lý quá phức tạp | Phù hợp với dữ liệu có cấu trúc, xử lý rõ ràng theo schema (ETL, analytics)                     |
| UDF hoạt động native, dễ dùng                                                                           | UDF vẫn dùng được nhưng không được tối ưu (vì là blackbox với Catalyst)                         |
| Không hỗ trợ nhiều builtin SQL-style ops (filter, groupBy, join...)                                     | Hỗ trợ đầy đủ SQL APIs và biểu thức logic dễ tối ưu hóa                                         |
| Hữu ích khi cần fine-grained control, debug.                                                            | Không linh hoạt bằng khi xử lý cực phức tạp hoặc cần cấu trúc không định nghĩa được bằng schema |

{% hint style="info" %}
**Luôn ưu tiên sử dụng DataFrame/Dataset**

Trong trường hợp bất khả kháng, chúng ta luôn ưu tiên sử dụng DataFrame/Dataset API thay vì RDD API nhằm tận dụng hiệu quả của các optimization engines.
{% endhint %}

[^1]: SIMD (Single Instruction, Multiple Data) là một dạng tối ưu phần cứng trong CPU.\
    Hiểu đơn giản: một lệnh CPU duy nhất có thể xử lý nhiều dữ liệu cùng lúc.
