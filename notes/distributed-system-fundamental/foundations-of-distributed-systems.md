# Foundations of Distributed Systems

Theo Robert Morris ([MIT 6.824](https://youtube.com/playlist?list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB\&si=-6gZ2rtaKKy00c1O)), **Distributed System** là tập hợp các máy tính độc lập phối hợp với nhau thông qua network để hoàn thành một nhiệm vụ chung một cách nhất quán. Các máy này không chia sẻ memory, không có global clock, và có thể hoạt động bình thường mặc dù xảy ra partial failure.

## Why Distributed System?

Mọi system sinh ra đều có mục tiêu giải quyết vấn đề hoặc bài toán cụ thể. Robert Morris cũng nhấn mạnh rằng distributed systems không nên được xây dựng nếu không thực sự cần thiết. Nếu vấn đề đó có thể giải quyết trên một single machine, thì luôn nên ưu tiên phương án đó vì nó đơn giản hơn, dễ debug hơn và ít failure hơn. Trước khi lựa chọn distributed system, cần cân nhắc và thử các giải pháp thay thế đơn giản hơn.

Martin Kleppmann (DDIA) cũng có quan điểm tương tự: distributed systems thường không được xây dựng vì chúng “tốt” hay “đẹp”, mà vì **không còn lựa chọn nào khác** để đáp ứng yêu cầu của bài toán.

Nói cách khác, distributed systems thường chỉ xuất hiện khi các yêu cầu của hệ thống **vượt quá khả năng giải quyết của một máy đơn lẻ**, và việc phân tán trở thành điều không thể tránh khỏi. Những yêu cầu phổ biến dẫn đến quyết định này bao gồm:

* **Scale**: workload vượt quá giới hạn CPU, memory, disk hoặc I/O của một máy.
* **Availability / Fault tolerance**: hệ thống cần tiếp tục phục vụ khi một số thành phần bị lỗi.
* **Physical or organizational constraints**: dữ liệu hoặc compute nằm ở nhiều nơi khác nhau.
* **Isolation and security**: yêu cầu tách biệt workload hoặc dữ liệu.

## Challenges

Distributed systems khó không phải vì có nhiều máy, mà vì chúng buộc ta phải làm việc trong một môi trường đầy bất định. Các giả định quen thuộc trong single-machine systems - như thứ tự thực thi rõ ràng, thành phần luôn phản hồi, hay network đáng tin cậy - đều không còn đúng. Thay vào đó, hệ thống phải được thiết kế để vẫn hoạt động đúng ngay cả khi các giả định này liên tục bị phá vỡ. Những khó khăn cốt lõi thường đến từ ba đặc điểm nền tảng sau.

### Concurrency

Trong distributed system, nhiều thành phần có thể hoạt động cùng lúc, nhưng không tồn tại một thứ tự thực thi hay mốc thời gian chung cho toàn hệ thống. Vì vậy, ta không thể biết chắc sự kiện nào xảy ra trước, sự kiện nào xảy ra sau. Một số tác vụ có thể phụ thuộc lẫn nhau, trong khi những tác vụ khác lại hoàn toàn độc lập. Nếu không phân biệt rõ hai trường hợp này, hệ thống rất dễ hoạt động sai. Do đó, điều khó không nằm ở việc “chạy song song”, mà ở việc phải suy luận và thiết kế đúng khi không có thứ tự rõ ràng.

### Partial Failure

Không giống như single-machine systems, nơi hệ thống thường “sống hoặc chết” một cách rõ ràng, distributed systems có thể rơi vào trạng thái chỉ **một phần** bị lỗi. Một số computer hoặc network connection có thể gặp sự cố trong khi các thành phần khác vẫn tiếp tục chạy bình thường. Điều này dẫn đến các trạng thái không đồng nhất: mỗi computer có thể nhìn thấy một state khác nhau của hệ thống. Thách thức nằm ở chỗ hệ thống vẫn phải đưa ra quyết định đúng trong khi không biết chắc thành phần nào đang hoạt động, thành phần nào đã fail, hay liệu một computer chỉ đang chậm phản hồi hay đã thực sự gặp lỗi.

### Unreliable Network

Giao tiếp giữa các computer luôn phải đi qua network, và network không phải là một thành phần đáng tin cậy. Gói tin có thể bị delay, bị duplicate, hoặc bị mất hoàn toàn. Quan trọng hơn, việc không nhận được phản hồi không đồng nghĩa với việc phía bên kia đã fail. Theo DDIA, distributed systems phải được thiết kế với giả định rằng mọi communication đều có thể thất bại, và không nên dựa vào các tín hiệu network để suy luận trạng thái hệ thống một cách chắc chắn. Chính sự mơ hồ này khiến việc xử lý timeout, retry và coordination trở nên đặc biệt khó khăn.

#### Abstraction and System Building Blocks

Một mục tiêu quan trọng của distributed systems là xây dựng các abstraction nhằm che giấu sự phức tạp của phân tán, để application có thể tương tác với hệ thống theo một cách đơn giản và nhất quán hơn. Thay vì phải đối mặt trực tiếp với network, failure hay concurrency, application được làm việc thông qua các interface ổn định, phản ánh hành vi mong muốn của hệ thống.

Tuy nhiên, abstraction trong distributed systems không chỉ là “che đi chi tiết”, mà còn là đặt ra ranh giới rõ ràng về những gì hệ thống đảm bảo và những gì không. Trong thực tế, failure, latency và inconsistency thường sẽ lộ ra ngoài abstraction ở một mức độ nào đó. Vì vậy, thiết kế abstraction luôn là bài toán cân bằng giữa sự đơn giản và khả năng phản ánh đúng hành vi của hệ thống bên dưới.

Ở mức nền tảng, distributed systems thường cần abstract ba nhóm vấn đề chính. Những abstraction này không loại bỏ hoàn toàn bản chất phân tán của hệ thống, nhưng giúp thu hẹp phạm vi những gì application cần quan tâm, và tạo nền tảng để xây dựng các cơ chế cao hơn như fault tolerance, consistency và coordination.

**Storage**\
Thay vì làm việc trực tiếp với nhiều bản sao dữ liệu, hệ thống cung cấp abstraction của một kho dữ liệu thống nhất. Abstraction này che giấu việc replication và failure của các node lưu trữ, nhưng đồng thời phải xác định rõ mức độ consistency mà hệ thống đảm bảo khi có concurrent updates hoặc sự cố xảy ra.

**Communication**\
Giao tiếp qua network được abstract thành các cơ chế như RPC hoặc message passing, giúp application không phải xử lý trực tiếp các chi tiết như packet loss hay retransmission. Tuy nhiên, abstraction này vẫn cần làm rõ semantics của communication, bao gồm việc request có thể được retry, bị duplicate hoặc không được xử lý đúng một lần.

**Computation**\
Việc thực thi được abstract theo cách không giả định rằng code chỉ chạy đúng một lần. Abstraction ở mức computation cần cho phép hệ thống chịu được retry, re-execution và crash, đồng thời đảm bảo rằng các thao tác không làm hệ thống rơi vào trạng thái sai khi được thực thi nhiều lần.

## Performance, Scalability, and Fault Tolerance

Scalability không chỉ là khả năng thêm máy, mà là khả năng duy trì performance khi load tăng. Trong nhiều trường hợp, bottleneck nằm ở coordination và design, chứ không phải ở tài nguyên phần cứng.

Fault tolerance thường được nhìn qua hai khía cạnh:

* **Availability**: hệ thống tiếp tục phục vụ request dưới một số failure nhất định.
* **Recoverability**: hệ thống có thể phục hồi sau failure và tiếp tục hoạt động như thể chưa từng có lỗi.

Phần dưới đây đi sâu hơn vào hai khía cạnh quan trọng của hệ thống phân tán: **reliability** và **scalability** thông qua DDIA. Thay vì nhìn chúng như các khái niệm trừu tượng, mục tiêu là hiểu rõ chúng ảnh hưởng như thế nào đến hành vi thực tế của hệ thống, và vì sao chúng luôn gắn chặt với workload, design và trade-off trong vận hành.

### Reliability

Reliability là khả năng hệ thống tiếp tục hoạt động đúng khi có sự cố.\
&#xNAN;_&#x46;ault_ là lỗi của một thành phần (disk, network, bug, config…), còn _failure_ là khi lỗi đó ảnh hưởng đến người dùng.

Không thể tránh hoàn toàn fault, nên mục tiêu là ngăn fault trở thành failure bằng thiết kế fault-tolerant.

Ba nguồn lỗi chính:

* Hardware faults: disk, network, máy chủ
* Software errors: bug mang tính hệ thống, cascading failure
* Human errors: config/deploy sai,...

Reliability không phải là không có lỗi, mà là chịu lỗi có kiểm soát: bảo vệ tính đúng đắn của dữ liệu, recover nhanh (rollback, restart, recompute), và monitoring rõ ràng.\
Trong một số trường hợp (prototype, chi phí thấp), có thể hy sinh reliability nhưng cần ý thức rõ trade-off.

### Scalability

Scalability là khả năng hệ thống duy trì performance ở mức chấp nhận được khi load tăng lên. Một hệ thống chỉ được coi là scalable khi sự tăng trưởng về load không làm suy giảm nghiêm trọng trải nghiệm người dùng hoặc hiệu năng tổng thể.

Scalability không phải là một mục tiêu độc lập, mà là hệ quả của việc hiểu đúng workload và cách workload đó thay đổi theo thời gian. Nếu chưa hiểu rõ hệ thống đang chịu loại load nào và vì sao load đó tăng, mọi nỗ lực “scale trước” đều có nguy cơ đi sai hướng hoặc trở nên lãng phí.

Scalability không thể đánh giá bằng cảm tính hay gán nhãn chung chung như “scale tốt” hay “không scale”. Để nói về scalability một cách có ý nghĩa, luôn cần xác định rõ hai yếu tố đo lường cốt lõi: load (hệ thống đang phải gánh bao nhiêu công việc) và performance (hệ thống phản ứng thế nào với load đó). Về bản chất, scalability chính là mối quan hệ giữa load tăng và performance thay đổi.

Load không có một định nghĩa chung cho mọi hệ thống, mà phụ thuộc vào cách hệ thống vận hành và workload cụ thể. Load cần được mô tả bằng các tham số phù hợp như QPS/RPS, read–write ratio, cache hit/miss rate, fan-out, hoặc skew/hot key. Quan trọng hơn, load không chỉ là số request bề mặt, mà là tổng lượng công việc thực sự phát sinh phía sau mỗi request, bao gồm database calls, cache lookups, service-to-service calls, hay queue operations. Nếu chỉ nhìn số lượng đọc/ghi đơn thuần, việc đánh giá scalability rất dễ bị sai lệch.

Performance cũng phụ thuộc vào loại hệ thống. Với batch systems, performance thường được đo bằng throughput hoặc tổng thời gian hoàn thành job; trong khi với online systems, thước đo quan trọng nhất là response time. Do các số liệu đo lường performance chịu ảnh hưởng của nhiều yếu tố bên ngoài như network, scheduling, garbage collection hay contention, performance cần được nhìn dưới dạng phân phối thống kê, thay vì chỉ dựa vào giá trị trung bình. Các hiện tượng như queueing delay hoặc một request nặng có thể làm nhiều request phía sau bị chậm, khiến số liệu trung bình không phản ánh đúng trải nghiệm thực tế của người dùng.

Sau khi đã xác định rõ load và performance, scalability được hiểu là cách hệ thống duy trì performance khi load tăng lên. Các hướng tiếp cận phổ biến gồm scale up (sử dụng phần cứng mạnh hơn) và scale out (phân tán workload ra nhiều node). Trong đó, các thành phần stateless thường scale dễ hơn các stateful components, do không cần đồng bộ trạng thái phức tạp và ít ràng buộc hơn khi phân tán.

## Replication and Consistency

Replication giúp tăng availability và performance, nhưng đồng thời làm phát sinh vấn đề consistency giữa các bản sao. Việc định nghĩa và đảm bảo consistency model phù hợp là nền tảng cho các cơ chế coordination và consensus được trình bày trong các phần tiếp theo của môn học.

## Tài liệu tham khảo

[MIT 6.824 Distributed System - Lecture 1: Introduction](https://youtu.be/cQP8WApzIQQ?si=Y-FF5pNqTyD1Lzc6)

[Distributed Systems 1.1: Introduction](https://youtu.be/UEAMfLPZZhE?si=cxMwm-LDcIJmfQqH)

[Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/)&#x20;







