# Foundations of Data Systems

## Reliable, Scalable, and Maintainable

Ở phần này, tớ không đi sâu vào từng khái niệm vì nội dung khá khô khan. Thay vào đó, tớ chỉ tóm lược lại những ý chính sau khi đọc. Nếu cậu muốn tìm hiểu chi tiết hơn, có thể tham khảo thêm các nguồn mà tớ đã đính kèm.

### Reliable

**Reliability** là khả năng hệ thống **tiếp tục hoạt động đúng khi có sự cố**.\
&#xNAN;_&#x46;ault_ là lỗi của một thành phần (disk, network, bug, config…), còn _failure_ là khi lỗi đó **ảnh hưởng đến người dùng**.

Không thể tránh hoàn toàn fault, nên mục tiêu là **ngăn fault trở thành failure** bằng thiết kế **fault-tolerant**.

Ba nguồn lỗi chính:

* **Hardware faults**: disk, network, máy chủ
* **Software errors**: bug mang tính hệ thống, cascading failure
* **Human errors**: config/deploy sai,...

Reliability không phải là không có lỗi, mà là **chịu lỗi có kiểm soát**: bảo vệ tính đúng đắn của dữ liệu, **recover nhanh** (rollback, restart, recompute), và **monitoring rõ ràng**.\
Trong một số trường hợp (prototype, chi phí thấp), có thể hy sinh reliability nhưng cần **ý thức rõ trade-off**.

### Scalable

Scalability là khả năng hệ thống **duy trì performance ở mức chấp nhận được khi load tăng lên**. Một hệ thống chỉ được coi là scalable khi sự tăng trưởng về load **không làm suy giảm nghiêm trọng trải nghiệm người dùng hoặc hiệu năng tổng thể**.

Scalability **không phải là một mục tiêu độc lập**, mà là **hệ quả của việc hiểu đúng workload** và cách workload đó thay đổi theo thời gian. Nếu chưa hiểu rõ hệ thống đang chịu loại load nào và vì sao load đó tăng, mọi nỗ lực “scale trước” đều có nguy cơ đi sai hướng hoặc trở nên lãng phí.

Scalability không thể đánh giá bằng cảm tính hay gán nhãn chung chung như “scale tốt” hay “không scale”. Để nói về scalability một cách có ý nghĩa, luôn cần xác định rõ **hai yếu tố đo lường cốt lõi**: **load** (hệ thống đang phải gánh bao nhiêu công việc) và **performance** (hệ thống phản ứng thế nào với load đó). Về bản chất, scalability chính là **mối quan hệ giữa load tăng và performance thay đổi**.

Load không có một định nghĩa chung cho mọi hệ thống, mà **phụ thuộc vào cách hệ thống vận hành và workload cụ thể**. Load cần được mô tả bằng các tham số phù hợp như QPS/RPS, read–write ratio, cache hit/miss rate, fan-out, hoặc skew/hot key. Quan trọng hơn, load **không chỉ là số request bề mặt**, mà là **tổng lượng công việc thực sự phát sinh phía sau mỗi request**, bao gồm database calls, cache lookups, service-to-service calls, hay queue operations. Nếu chỉ nhìn số lượng đọc/ghi đơn thuần, việc đánh giá scalability rất dễ bị sai lệch.

Performance cũng phụ thuộc vào loại hệ thống. Với **batch systems**, performance thường được đo bằng throughput hoặc tổng thời gian hoàn thành job; trong khi với **online systems**, thước đo quan trọng nhất là **response time**. Do các số liệu đo lường performance chịu ảnh hưởng của nhiều yếu tố bên ngoài như network, scheduling, garbage collection hay contention, performance cần được nhìn dưới dạng **phân phối thống kê**, thay vì chỉ dựa vào giá trị trung bình. Các hiện tượng như **queueing delay** hoặc một **request nặng** có thể làm nhiều request phía sau bị chậm, khiến số liệu trung bình không phản ánh đúng trải nghiệm thực tế của người dùng.

Sau khi đã xác định rõ load và performance, scalability được hiểu là **cách hệ thống duy trì performance khi load tăng lên**. Các hướng tiếp cận phổ biến gồm **scale up** (sử dụng phần cứng mạnh hơn) và **scale out** (phân tán workload ra nhiều node). Trong đó, các thành phần **stateless** thường scale dễ hơn các **stateful components**, do không cần đồng bộ trạng thái phức tạp và ít ràng buộc hơn khi phân tán.

### Maintainable





