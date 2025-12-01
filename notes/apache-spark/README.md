# Apache Spark

## What is Apache Spark?

Apache Spark là unified computing engine và tập hợp các thư viện cho việc parallel data processing (xử lý dữ liệu song song) trên một machine clusters (VMs).

* **Unified**: Spark hỗ trợ nhiều loại tasks khác nhau, từ Data Analytics, Data Loading và SQL Queries đến Machine Learning và Stream Processing.
* **Computing Engine:** Spark chỉ hỗ trợ loading data và xử lý data chứ không hỗ trợ data storage, nhưng Spark có thể thao tác với nhiều loại storage khác nhau, như HDFS, Cloud Storage (Amazon S3, Azure Storage, Google Cloud Storage), Apache Cassandra, etc.
* **Libraries:** Spark hỗ trợ thư viện cho nhiều loại ngôn ngữ như SQL, Java, Scala, Python, etc; cho những mục đích như Machine Learning, Stream Processing, etc.

## Overview Architecture

Spark là một framework điều phối hoạt động của một nhóm machines (quản lý và điều phối quá trình thực thi các tasks trong toàn bộ cluster). Lưu lý rằng Spark chỉ quản lý task trên cluster, không quản lý cluster, nhiệm vụ này được tổ chức bởi Cluster Manager (Spark standalone cluster manager, Hadoop YARN, Mesos hoặc Kubernetes).<br>

<figure><img src="../.gitbook/assets/spark_architecture.png" alt=""><figcaption><p>Apache Spark Overview Architecture</p></figcaption></figure>

### **Spark Application**

Code của người dùng sử dụng Spark API (Python, Java, Scala, hoặc Spark SQL) được đóng gói thành `SparkApplication`. Chúng ta submit application này tới `ClusterManager` thông qua câu lệnh `spark-submit`. `SparkApplication` bao gồm một `SparkDriver` chịu trách nhiệm điều phối các hoạt động song song trên Spark cluster. Driver này truy cập các thành phần phân tán (`SparkExecutor` và `ClusterManager`) thông qua `SparkSession`.

### **Cluster Manager**

`ClusterManager` chịu trách nhiệm phân phối tài nguyên để thực thi các `SparkApplication`.

### **Spark Driver**

`SparkDriver` chịu trách nhiệm khởi tạo `SparkSession`; giao tiếp với `ClusterManager` để yêu cầu tài nguyên (CPU, memory) cho `SparkExecutor` ; chuyển đổi các Spark operations thành DAG computations, lập lịch và phân phối thành các tasks cho các Spark Executors.

### **Spark Session**

`SparkSession` là entry point cho các Spark Application kể từ Spark 2.0. Nó gần như là bản nâng cấp của `SparkContext`, bằng việc kết hợp `SparkContext`, `SQLContext`, `HiveContext`, và `StreamingContext`, trong khi với Spark 1.x, ta phải tạo từng Conext một.

### **Spark Executor**

`SparkExecutor` là JVMs process chạy trên các `WorkerNode` (đối với Spark in K8S thì executor chạy trong các `Pod`), chịu trách nhiệm thực thi các tasks. Trong hầu hết deployments modes, thì chỉ có 1 executor trên 1 node.

### Deployment Mode

| Mode           | Spark Driver                                     | Spark Executor                                          | Cluster Manager                                                                                                      |
| -------------- | ------------------------------------------------ | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Local          | Chạy trên một JVM                                | Chạy trên cùng JVM với driver                           | Chạy trên cùng host                                                                                                  |
| Standard Alone | Có thể chạy trên bất cứ node nào của cụm         | Mỗi node đều chạy 1 JVM và launch executor của chính nó | Một host sẽ chịu trách nhiệm làm master → subm                                                                       |
| YARN (client)  | Chạy trên Client (không phải thành phần của cụm) | YARN’s NodeManager’s container                          | YARN’s Resource Manager kết hợp với YARN’s Application Master để chỉ định containers trên NodeManagers cho executors |
| YARN (cluster) | Chạy cùng với YARN Applcation Master             | YARN’s NodeManager’s container                          | YARN’s Resource Manager kết hợp với YARN’s Application Master để chỉ định containers trên NodeManagers cho executors |
| Kubernetes     | Chạy trên một K8S Pod                            | Mỗi workers chạy trong 1 K8S Pod                        | Kubernetes Master                                                                                                    |

### **Distributed data and partitions**

Trong thực tế, dữ liêu phân bố trong các storage dưới dạng các partitions. Tuy nhiên, Spark sẽ đối xử với từng partition này như một **in-memory** **high-level logical data abstraction - DataFrame**. Mặc dù không phải lúc nào cũng có thể, các executor sẽ được ưu tiên phân bố các task yêu cầu đọc partition gần với nó nhất và chỉ xử lý dữ liệu trên các data partition được assign này.
