---
hidden: true
---

# Apache Spark

## Trước khi bắt đầu

Khi bắt đầu viết những dòng ghi chú này, tớ đã làm việc với Spark khoảng hai năm - “làm việc” chứ không hẳn là hai năm kinh nghiệm. Tớ từng đọc sơ qua kiến trúc Spark, từng viết Spark Application để xử lý các tác vụ cơ bản, và cũng từng tự xây dựng một framework cho Data Engineer định nghĩa logic của luồng stateless streaming từ Kafka sang TiDB/HDFS bằng DSL, kèm theo automation deploy & monitoring những pipelines này.

Tớ cũng đã vận hành Spark Standalone trên bare-metal, triển khai Spark trên K8S master, và cả chạy Spark bằng Spark Operator (thậm chí tớ còn viết Airflow plugin để đóng gói logic tạo Custom Resources qua UI, dynamic configure environment với PyPi/Maven/Packaged Env/Jars,... ). Ngoài ra, việc đóng gói virtual environment, dependencies gửi kèm vào Spark jobs, chỉ định Python cũng không còn xa lạ với tớ.

Nhưng dù đã chạm vào khá nhiều thứ, tớ vẫn thấy “**mình chưa thực sự hiểu Spark**”. Những buổi phỏng vấn hỏi kiểu “Lazy Evaluation là gì?” hay “Shuffle hoạt động như thế nào?” khiến tớ nhận ra mình thiếu những mảnh ghép quan trọng. Khi gặp sự cố do job tối ưu chưa tốt, tớ biết rằng cần đọc Spark UI và Execution Plan để phân tích vấn đề, nhưng lại không thật sự hiểu phải đọc _như thế nào_.

Sau nhiều lần nhờ AI lập roadmap, thử đi thử lại hàng chục lần rồi lại bỏ giữa chừng, tớ quyết định quay lại học từ đầu với cái notes này. Tớ coi đây như một cách để buộc bản thân kiên trì hơn với hành trình dài hơi này. Nhưng mới đọc vài phần thôi là tớ đã thấy “ngợp” - và cũng nhờ vậy mà nhận ra nền tảng của mình chưa đủ vững. Tớ chưa nắm chắc các khái niệm về distributed processing và distributed systems. Thế nên, trước khi đi xa hơn với Spark, tớ muốn quay lại củng cố nền tảng trước đã - xây cho chắc cái móng rồi mới dựng được phần còn lại.

{% content-ref url="../distributed-system-fundamental/" %}
[distributed-system-fundamental](../distributed-system-fundamental/)
{% endcontent-ref %}

## What is Apache Spark?

Apache Spark là unified computing engine và tập hợp các thư viện cho việc parallel data processing (xử lý dữ liệu song song) trên một machine clusters (VMs).

* **Unified**: Spark hỗ trợ nhiều loại tasks khác nhau, từ Data Analytics, Data Loading và SQL Queries đến Machine Learning và Stream Processing.
* **Computing Engine:** Spark chỉ hỗ trợ loading data và xử lý data chứ không hỗ trợ data storage, nhưng Spark có thể thao tác với nhiều loại storage khác nhau, như HDFS, Cloud Storage (Amazon S3, Azure Storage, Google Cloud Storage), Apache Cassandra, etc.
* **Libraries:** Spark hỗ trợ thư viện cho nhiều loại ngôn ngữ như SQL, Java, Scala, Python, etc; cho những mục đích như Machine Learning, Stream Processing, etc.

## Overview Architecture

Spark là một framework điều phối hoạt động của một nhóm machines (quản lý và điều phối quá trình thực thi các tasks trong toàn bộ cluster). Lưu lý rằng Spark chỉ quản lý task trên cluster, không quản lý cluster, nhiệm vụ này được tổ chức bởi Cluster Manager.

<figure><img src="../.gitbook/assets/spark_architecture.png" alt=""><figcaption><p>Apache Spark components and architecture</p></figcaption></figure>

### **Spark Application**

Code của người dùng sử dụng Spark API (Python, Java, Scala, hoặc Spark SQL) được đóng gói thành Spark Application. Chúng ta submit application này tới Cluster Manager thông qua câu lệnh `spark-submit`. Spark Application bao gồm một Spark Driver chịu trách nhiệm điều phối các hoạt động song song trên Spark cluster. Driver này truy cập các thành phần phân tán (Spark Executor và Cluster Manager) thông qua SparkSession.

### **Cluster Manager**

Cluster Manager hay Spark Master chịu trách nhiệm phân phối tài nguyên để thực thi các Spark Application. Master có thể là built-in standalone cluster manager, Hadoop YARN, Apache Mesos hoặc Kubernetes.

### **Spark Driver**

Spark Driver chịu trách nhiệm khởi tạo SparkSession; giao tiếp với Cluster Manager để yêu cầu tài nguyên (CPU, memory) cho Spark Executor ; chuyển đổi các Spark operations thành DAG computations, lập lịch và phân phối thành các tasks cho các Spark Executors.

### **Spark Session**

Spark Session là entry point cho các Spark Application kể từ Spark 2.0. Nó gần như là bản nâng cấp của Spark Context, bằng việc kết hợp Spark Context, SQL Context, Hive Context, và Streamin gContext, trong khi với Spark 1.x, ta phải tạo từng Context một.

### **Spark Executor**

Spark Executor là JVMs process chạy trên các Worker Node (đối với Spark in K8S thì executor chạy trong các Pod), chịu trách nhiệm thực thi các tasks. Trong hầu hết deployments modes, thì chỉ có 1 executor trên 1 node.

### Deployment Mode

| Mode           | Spark Driver                                     | Spark Executor                                          | Cluster Manager                                                                                                      |
| -------------- | ------------------------------------------------ | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Local          | Chạy trên một JVM                                | Chạy trên cùng JVM với driver                           | Chạy trên cùng host                                                                                                  |
| Standard Alone | Có thể chạy trên bất cứ node nào của cụm         | Mỗi node đều chạy 1 JVM và launch executor của chính nó | Một host sẽ chịu trách nhiệm làm master → subm                                                                       |
| YARN (client)  | Chạy trên Client (không phải thành phần của cụm) | YARN’s NodeManager’s container                          | YARN’s Resource Manager kết hợp với YARN’s Application Master để chỉ định containers trên NodeManagers cho executors |
| YARN (cluster) | Chạy cùng với YARN Applcation Master             | YARN’s NodeManager’s container                          | YARN’s Resource Manager kết hợp với YARN’s Application Master để chỉ định containers trên NodeManagers cho executors |
| Kubernetes     | Chạy trên một K8S Pod                            | Mỗi workers chạy trong 1 K8S Pod                        | Kubernetes Master                                                                                                    |

### **Distributed Data and Partitions**

Trong thực tế, physical data được lưu trữ và phân bố thành nhiều partition trên HDFS hoặc các Cloud Storage. Tuy nhiên, khi xử lý, Spark không làm việc trực tiếp với các partition vật lý này. Thay vào đó, Spark trừu tượng hoá mỗi partition thành một đơn vị logic trong bộ nhớ - tức một DataFrame.

Khi lập lịch thực thi, Spark **ưu tiên** giao nhiệm vụ cho executor đọc những partition “gần” nó nhất trong topology mạng (data locality). Điều này giúp giảm chi phí truyền dữ liệu qua mạng, dù không phải lúc nào Spark cũng đảm bảo được sự gần nhất này.

{% @gitbook-network/gitbook-network content="# Apache Spark Learning Network Configuration
# This file defines nodes, edges, groups, colors, and visual settings

# ============================================================================
# GROUPS: Define node groups with colors and display names
# ============================================================================
groups:
  distributed:
    name: "Distributed Systems"
    fill: "#1976D2"
    stroke: "#1976D2"
  
  sparkIdentity:
    name: "Spark Identity"
    fill: "#388E3C"
    stroke: "#388E3C"
  
  architecture:
    name: "Architecture"
    fill: "#F57C00"
    stroke: "#F57C00"
  
  jobStage:
    name: "Job/Stage/Task"
    fill: "#D32F2F"
    stroke: "#D32F2F"
  
  dataframe:
    name: "DataFrame & Engine"
    fill: "#7B1FA2"
    stroke: "#7B1FA2"
  
  fileFormat:
    name: "File Formats"
    fill: "#FFFDE7"
    stroke: "#FBC02D"
  
  optimization:
    name: "Optimization"
    fill: "#FCE4EC"
    stroke: "#C2185B"
  
  runtime:
    name: "Runtime & Tools"
    fill: "#ECEFF1"
    stroke: "#455A64"

# ============================================================================
# CONFIG: Visual and behavior settings
# ============================================================================
config:
  # Node size calculation
  nodeSize:
    minSize: 0.5
    maxSize: 3.0
    sizeMultiplier: 1.5  # For important nodes
  
  # Important nodes (get larger base size)
  importantNodes:
    - "B1"  # What is Apache Spark
    - "C1"  # Driver Program
    - "C8"  # Executor Concept
    - "D1"  # Job Concept
    - "D2"  # Stage Concept
    - "D9"  # Shuffle Operation
    - "E1"  # DataFrame API
    - "E7"  # Catalyst Optimizer
    - "E11" # Tungsten Engine
    - "E14" # DAG Construction
    - "F1"  # File Format Overview
    - "G1"  # Join Algorithms
    - "H6"  # SparkContext
    - "H10" # Memory Management
  
  # Default focus node (auto-focus on load)
  defaultFocusNode: "A1"
  
  # Node visual settings
  node:
    fillOpacity: 1.0      # Solid color (0.0 - 1.0)
    strokeWidth: 2
    baseRadius: 4         # Base radius multiplier
    sizeMultiplier: 6     # Size multiplier for radius calculation
  
  # Edge visual settings
  edge:
    normal:
      opacity: 0.4
      width: 1
      color: "#1976D2"
    strong:
      opacity: 0.6
      width: 1
      color: "#1976D2"
  
  # Highlight settings (when node is focused)
  highlight:
    selectedNode:
      sizeMultiplier: 1.6  # How much larger selected node becomes
    disconnectedEdges:
      opacity: 0.2       # Very faded

# ============================================================================
# NODES: Define all nodes in the network
# ============================================================================
nodes:
  # Distributed Systems (1-10)
  - id: "A1"
    label: "1. Distributed Computing Concepts"
    group: "distributed"
  
  - id: "A2"
    label: "2. Fault Tolerance"
    group: "distributed"
  
  - id: "A3"
    label: "3. Scalability Principles"
    group: "distributed"
  
  - id: "A4"
    label: "4. Network Communication"
    group: "distributed"
  
  - id: "A5"
    label: "5. Parallelism Concepts"
    group: "distributed"
  
  - id: "A6"
    label: "6. Concurrency vs Parallelism"
    group: "distributed"
  
  - id: "A7"
    label: "7. Cluster Architecture"
    group: "distributed"
  
  - id: "A8"
    label: "8. Master-Worker Pattern"
    group: "distributed"
  
  - id: "A9"
    label: "9. Data Locality Principles"
    group: "distributed"
  
  - id: "A10"
    label: "10. Rack Awareness"
    group: "distributed"
  
  # Spark Identity (11-20)
  - id: "B1"
    label: "11. What is Apache Spark"
    group: "sparkIdentity"
    references:
      - title: "Apache Spark Official Documentation"
        url: "https://spark.apache.org/docs/latest/"
      - title: "Spark Overview"
        url: "https://spark.apache.org/docs/latest/quick-start.html"
  
  - id: "B2"
    label: "12. Spark vs MapReduce - Speed"
    group: "sparkIdentity"
  
  - id: "B3"
    label: "13. Spark vs MapReduce - Memory"
    group: "sparkIdentity"
  
  - id: "B4"
    label: "14. Spark vs MapReduce - Iterative"
    group: "sparkIdentity"
  
  - id: "B5"
    label: "15. Spark vs MapReduce - Fault Tolerance"
    group: "sparkIdentity"
  
  - id: "B6"
    label: "16. Spark Ecosystem Overview"
    group: "sparkIdentity"
  
  - id: "B7"
    label: "17. Spark Core"
    group: "sparkIdentity"
  
  - id: "B8"
    label: "18. Spark SQL"
    group: "sparkIdentity"
  
  - id: "B9"
    label: "19. Spark Streaming"
    group: "sparkIdentity"
  
  - id: "B10"
    label: "20. MLlib & GraphX"
    group: "sparkIdentity"
  
  # Architecture (21-30)
  - id: "C1"
    label: "21. Driver Program"
    group: "architecture"
    references:
      - title: "Spark Architecture - Driver"
        url: "https://spark.apache.org/docs/latest/cluster-overview.html#driver"
      - title: "Driver Program Details"
        url: "https://spark.apache.org/docs/latest/submitting-applications.html"
  
  - id: "C2"
    label: "22. Driver Responsibilities"
    group: "architecture"
  
  - id: "C3"
    label: "23. Cluster Manager Types"
    group: "architecture"
  
  - id: "C4"
    label: "24. Standalone Cluster Manager"
    group: "architecture"
  
  - id: "C5"
    label: "25. YARN Cluster Manager"
    group: "architecture"
  
  - id: "C6"
    label: "26. Mesos Cluster Manager"
    group: "architecture"
  
  - id: "C7"
    label: "27. Kubernetes Cluster Manager"
    group: "architecture"
  
  - id: "C8"
    label: "28. Executor Concept"
    group: "architecture"
    references:
      - title: "Spark Executors"
        url: "https://spark.apache.org/docs/latest/cluster-overview.html#executors"
      - title: "Executor Configuration"
        url: "https://spark.apache.org/docs/latest/configuration.html#executors"
  
  - id: "C9"
    label: "29. Executor Cores & Memory"
    group: "architecture"
  
  - id: "C10"
    label: "30. Executor Tasks"
    group: "architecture"
  
  # Job/Stage/Task (31-41)
  - id: "D1"
    label: "31. Job Concept"
    group: "jobStage"
    references:
      - title: "Jobs, Stages, and Tasks"
        url: "https://spark.apache.org/docs/latest/job-scheduling.html"
      - title: "Understanding Spark Jobs"
        url: "https://spark.apache.org/docs/latest/monitoring.html#jobs"
  
  - id: "D2"
    label: "32. Stage Concept"
    group: "jobStage"
  
  - id: "D3"
    label: "33. Task Concept"
    group: "jobStage"
  
  - id: "D4"
    label: "34. Job Submission Flow"
    group: "jobStage"
  
  - id: "D5"
    label: "35. Stage Dependencies"
    group: "jobStage"
  
  - id: "D6"
    label: "36. Task Scheduling"
    group: "jobStage"
  
  - id: "D7"
    label: "37. Narrow Transformations"
    group: "jobStage"
  
  - id: "D8"
    label: "38. Wide Transformations"
    group: "jobStage"
  
  - id: "D9"
    label: "39. Shuffle Operation"
    group: "jobStage"
    references:
      - title: "Understanding Shuffle Operations"
        url: "https://spark.apache.org/docs/latest/rdd-programming-guide.html#shuffle-operations"
      - title: "Tuning Spark Shuffle"
        url: "https://spark.apache.org/docs/latest/tuning.html#shuffle-behavior"
  
  - id: "D10"
    label: "40. Shuffle Write"
    group: "jobStage"
  
  - id: "D11"
    label: "41. Shuffle Read"
    group: "jobStage"
  
  # DataFrame & Engine (42-56)
  - id: "E1"
    label: "42. DataFrame API Introduction"
    group: "dataframe"
    references:
      - title: "Spark SQL Guide"
        url: "https://spark.apache.org/docs/latest/sql-programming-guide.html"
      - title: "DataFrame API"
        url: "https://spark.apache.org/docs/latest/sql-getting-started.html"
  
  - id: "E2"
    label: "43. DataFrame vs RDD"
    group: "dataframe"
  
  - id: "E3"
    label: "44. Lazy Evaluation"
    group: "dataframe"
  
  - id: "E4"
    label: "45. Transformations"
    group: "dataframe"
  
  - id: "E5"
    label: "46. Actions"
    group: "dataframe"
  
  - id: "E6"
    label: "47. Logical Plan"
    group: "dataframe"
  
  - id: "E7"
    label: "48. Catalyst Optimizer"
    group: "dataframe"
    references:
      - title: "Catalyst Optimizer Paper"
        url: "https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html"
      - title: "Spark SQL Optimization"
        url: "https://spark.apache.org/docs/latest/sql-performance-tuning.html"
  
  - id: "E8"
    label: "49. Rule-Based Optimization"
    group: "dataframe"
  
  - id: "E9"
    label: "50. Cost-Based Optimization"
    group: "dataframe"
  
  - id: "E10"
    label: "51. Physical Plan"
    group: "dataframe"
  
  - id: "E11"
    label: "52. Tungsten Engine"
    group: "dataframe"
    references:
      - title: "Project Tungsten"
        url: "https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html"
      - title: "Tungsten Memory Management"
        url: "https://spark.apache.org/docs/latest/tuning.html#memory-management-overview"
  
  - id: "E12"
    label: "53. Whole-Stage Code Generation"
    group: "dataframe"
  
  - id: "E13"
    label: "54. Columnar Processing"
    group: "dataframe"
  
  - id: "E14"
    label: "55. DAG Construction"
    group: "dataframe"
  
  - id: "E15"
    label: "56. DAG Optimization"
    group: "dataframe"
  
  # File Formats (57-67)
  - id: "F1"
    label: "57. File Format Overview"
    group: "fileFormat"
  
  - id: "F2"
    label: "58. Parquet Format"
    group: "fileFormat"
  
  - id: "F3"
    label: "59. Parquet Columnar Storage"
    group: "fileFormat"
  
  - id: "F4"
    label: "60. ORC Format"
    group: "fileFormat"
  
  - id: "F5"
    label: "61. Avro Format"
    group: "fileFormat"
  
  - id: "F6"
    label: "62. JSON Format"
    group: "fileFormat"
  
  - id: "F7"
    label: "63. CSV Format"
    group: "fileFormat"
  
  - id: "F8"
    label: "64. Partitioning Concept"
    group: "fileFormat"
  
  - id: "F9"
    label: "65. Partition Pruning"
    group: "fileFormat"
  
  - id: "F10"
    label: "66. Bucketing Concept"
    group: "fileFormat"
  
  - id: "F11"
    label: "67. Bucket Pruning"
    group: "fileFormat"
  
  # Optimization (68-79)
  - id: "G1"
    label: "68. Join Algorithms Overview"
    group: "optimization"
    references:
      - title: "Join Strategies in Spark"
        url: "https://spark.apache.org/docs/latest/sql-performance-tuning.html#join-strategy-hints"
      - title: "Broadcast Join"
        url: "https://spark.apache.org/docs/latest/sql-performance-tuning.html#broadcast-hint-for-sql-queries"
  
  - id: "G2"
    label: "69. Sort-Merge Join"
    group: "optimization"
  
  - id: "G3"
    label: "70. Broadcast Join"
    group: "optimization"
  
  - id: "G4"
    label: "71. Hash Join"
    group: "optimization"
  
  - id: "G5"
    label: "72. Shuffle Hash Join"
    group: "optimization"
  
  - id: "G6"
    label: "73. Shuffle Tuning"
    group: "optimization"
  
  - id: "G7"
    label: "74. Shuffle Partitions"
    group: "optimization"
  
  - id: "G8"
    label: "75. Data Skew Problem"
    group: "optimization"
  
  - id: "G9"
    label: "76. Skew Detection"
    group: "optimization"
  
  - id: "G10"
    label: "77. Salting Technique"
    group: "optimization"
  
  - id: "G11"
    label: "78. Broadcast Variables"
    group: "optimization"
  
  - id: "G12"
    label: "79. Accumulators"
    group: "optimization"
  
  # Runtime & Tools (80-93)
  - id: "H1"
    label: "80. Client Mode"
    group: "runtime"
  
  - id: "H2"
    label: "81. Cluster Mode"
    group: "runtime"
  
  - id: "H3"
    label: "82. spark-submit Command"
    group: "runtime"
  
  - id: "H4"
    label: "83. Application JAR"
    group: "runtime"
  
  - id: "H5"
    label: "84. Driver Internals"
    group: "runtime"
  
  - id: "H6"
    label: "85. SparkContext"
    group: "runtime"
  
  - id: "H7"
    label: "86. SQLContext"
    group: "runtime"
  
  - id: "H8"
    label: "87. Executor Internals"
    group: "runtime"
  
  - id: "H9"
    label: "88. Task Execution"
    group: "runtime"
  
  - id: "H10"
    label: "89. Memory Management"
    group: "runtime"
  
  - id: "H11"
    label: "90. Spark UI"
    group: "runtime"
  
  - id: "H12"
    label: "91. DAG Visualization"
    group: "runtime"
  
  - id: "H13"
    label: "92. Stage Details"
    group: "runtime"
  
  - id: "H14"
    label: "93. Task Metrics"
    group: "runtime"

# ============================================================================
# EDGES: Define all connections between nodes
# ============================================================================
edges:
  # Original links within groups
  - source: "A1"
    target: "A2"
  
  - source: "A1"
    target: "A3"
  
  - source: "A1"
    target: "A4"
  
  - source: "A5"
    target: "A6"
  
  - source: "A7"
    target: "A8"
  
  - source: "A9"
    target: "A10"
  
  - source: "A1"
    target: "A7"
  
  - source: "A7"
    target: "A9"
  
  - source: "B1"
    target: "B2"
  
  - source: "B1"
    target: "B3"
  
  - source: "B1"
    target: "B4"
  
  - source: "B1"
    target: "B5"
  
  - source: "B1"
    target: "B6"
  
  - source: "B6"
    target: "B7"
  
  - source: "B6"
    target: "B8"
  
  - source: "B6"
    target: "B9"
  
  - source: "B6"
    target: "B10"
  
  - source: "C1"
    target: "C2"
  
  - source: "C3"
    target: "C4"
  
  - source: "C3"
    target: "C5"
  
  - source: "C3"
    target: "C6"
  
  - source: "C3"
    target: "C7"
  
  - source: "C8"
    target: "C9"
  
  - source: "C8"
    target: "C10"
  
  - source: "D1"
    target: "D2"
  
  - source: "D2"
    target: "D3"
  
  - source: "D1"
    target: "D4"
  
  - source: "D2"
    target: "D5"
  
  - source: "D3"
    target: "D6"
  
  - source: "D2"
    target: "D7"
  
  - source: "D2"
    target: "D8"
  
  - source: "D8"
    target: "D9"
  
  - source: "D9"
    target: "D10"
  
  - source: "D9"
    target: "D11"
  
  - source: "E1"
    target: "E2"
  
  - source: "E1"
    target: "E3"
  
  - source: "E1"
    target: "E4"
  
  - source: "E1"
    target: "E5"
  
  - source: "E4"
    target: "E6"
  
  - source: "E6"
    target: "E7"
  
  - source: "E7"
    target: "E8"
  
  - source: "E7"
    target: "E9"
  
  - source: "E7"
    target: "E10"
  
  - source: "E10"
    target: "E11"
  
  - source: "E11"
    target: "E12"
  
  - source: "E11"
    target: "E13"
  
  - source: "E10"
    target: "E14"
  
  - source: "E14"
    target: "E15"
  
  - source: "F1"
    target: "F2"
  
  - source: "F1"
    target: "F4"
  
  - source: "F1"
    target: "F5"
  
  - source: "F1"
    target: "F6"
  
  - source: "F1"
    target: "F7"
  
  - source: "F2"
    target: "F3"
  
  - source: "F1"
    target: "F8"
  
  - source: "F8"
    target: "F9"
  
  - source: "F8"
    target: "F10"
  
  - source: "F10"
    target: "F11"
  
  - source: "G1"
    target: "G2"
  
  - source: "G1"
    target: "G3"
  
  - source: "G1"
    target: "G4"
  
  - source: "G1"
    target: "G5"
  
  - source: "G6"
    target: "G7"
  
  - source: "G1"
    target: "G8"
  
  - source: "G8"
    target: "G9"
  
  - source: "G8"
    target: "G10"
  
  - source: "G3"
    target: "G11"
  
  - source: "H1"
    target: "H3"
  
  - source: "H2"
    target: "H3"
  
  - source: "H3"
    target: "H4"
  
  - source: "H5"
    target: "H6"
  
  - source: "H5"
    target: "H7"
  
  - source: "H8"
    target: "H9"
  
  - source: "H8"
    target: "H10"
  
  - source: "H11"
    target: "H12"
  
  - source: "H12"
    target: "H13"
  
  - source: "H12"
    target: "H14"
  
  # Cross-layer connections (strong links)
  - source: "A1"
    target: "B1"
    strong: true
  
  - source: "A5"
    target: "B1"
    strong: true
  
  - source: "A9"
    target: "D9"
    strong: true
  
  - source: "B1"
    target: "C1"
    strong: true
  
  - source: "B1"
    target: "C3"
    strong: true
  
  - source: "B1"
    target: "C8"
    strong: true
  
  - source: "B1"
    target: "H1"
    strong: true
  
  - source: "B1"
    target: "H2"
    strong: true
  
  - source: "C1"
    target: "D1"
    strong: true
  
  - source: "C8"
    target: "D1"
    strong: true
  
  - source: "C1"
    target: "H5"
    strong: true
  
  - source: "C8"
    target: "H8"
    strong: true
  
  - source: "C1"
    target: "G12"
    strong: true
  
  - source: "D8"
    target: "E14"
    strong: true
  
  - source: "D9"
    target: "E14"
    strong: true
  
  - source: "D9"
    target: "G1"
    strong: true
  
  - source: "D9"
    target: "G6"
    strong: true
  
  - source: "D8"
    target: "G1"
    strong: true
  
  - source: "D1"
    target: "H12"
    strong: true
  
  - source: "D2"
    target: "H13"
    strong: true
  
  - source: "D3"
    target: "H14"
    strong: true
  
  - source: "E4"
    target: "G1"
    strong: true
  
  - source: "E10"
    target: "F1"
    strong: true
  
  - source: "E7"
    target: "E15"
    strong: true
  
  - source: "E11"
    target: "C8"
    strong: true
  
  - source: "E14"
    target: "H12"
    strong: true
  
  - source: "F8"
    target: "D9"
    strong: true
  
  - source: "F10"
    target: "G5"
    strong: true
  
  # Enhanced cross-links
  - source: "E11"
    target: "H10"
    strong: true
  
  - source: "E11"
    target: "C9"
    strong: true
  
  - source: "E12"
    target: "H10"
    strong: true
  
  - source: "E7"
    target: "F1"
    strong: true
  
  - source: "E7"
    target: "F2"
    strong: true
  
  - source: "E9"
    target: "F1"
    strong: true
  
  - source: "D9"
    target: "F8"
    strong: true
  
  - source: "D9"
    target: "F10"
    strong: true
  
  - source: "D10"
    target: "F8"
    strong: true
  
  - source: "D11"
    target: "F8"
    strong: true
  
  - source: "D9"
    target: "G7"
    strong: true
  
  - source: "D5"
    target: "D7"
    strong: true
  
  - source: "D5"
    target: "D8"
    strong: true
  
  - source: "D5"
    target: "D9"
    strong: true
  
  - source: "E4"
    target: "D7"
    strong: true
  
  - source: "E4"
    target: "D8"
    strong: true
  
  - source: "D7"
    target: "E14"
    strong: true
  
  - source: "E10"
    target: "D2"
    strong: true
  
  - source: "E10"
    target: "D3"
    strong: true
  
  - source: "E10"
    target: "H9"
    strong: true
  
  - source: "C9"
    target: "H10"
    strong: true
  
  - source: "C10"
    target: "H10"
    strong: true
  
  - source: "H10"
    target: "D6"
    strong: true
  
  - source: "H10"
    target: "G7"
    strong: true
  
  - source: "E13"
    target: "F2"
    strong: true
  
  - source: "E13"
    target: "F3"
    strong: true
  
  - source: "E13"
    target: "F4"
    strong: true
  
  - source: "G1"
    target: "D9"
    strong: true
  
  - source: "G2"
    target: "D9"
    strong: true
  
  - source: "G5"
    target: "D9"
    strong: true
  
  - source: "G3"
    target: "C1"
    strong: true
  
  - source: "G8"
    target: "F8"
    strong: true
  
  - source: "G8"
    target: "F10"
    strong: true
  
  - source: "G10"
    target: "F8"
    strong: true
  
  - source: "E14"
    target: "D2"
    strong: true
  
  - source: "E15"
    target: "D5"
    strong: true
  
  - source: "E14"
    target: "D1"
    strong: true
  
  - source: "H6"
    target: "E1"
    strong: true
  
  - source: "H7"
    target: "E1"
    strong: true
  
  - source: "H6"
    target: "C1"
    strong: true
  
  - source: "H7"
    target: "E7"
    strong: true
  
  - source: "H9"
    target: "D3"
    strong: true
  
  - source: "H9"
    target: "C8"
    strong: true
  
  - source: "H9"
    target: "E11"
    strong: true
  
  - source: "H12"
    target: "E14"
    strong: true
  
  - source: "H12"
    target: "D2"
    strong: true
  
  - source: "H13"
    target: "D2"
    strong: true
  
  - source: "H14"
    target: "D3"
    strong: true
  
  - source: "C3"
    target: "H10"
    strong: true
  
  - source: "C5"
    target: "H10"
    strong: true
  
  - source: "A2"
    target: "B5"
    strong: true
  
  - source: "A2"
    target: "D1"
    strong: true
  
  - source: "A2"
    target: "C8"
    strong: true
  
  - source: "A5"
    target: "D7"
    strong: true
  
  - source: "A5"
    target: "D8"
    strong: true
  
  - source: "A6"
    target: "D6"
    strong: true
  
  - source: "A9"
    target: "F8"
    strong: true
  
  - source: "A9"
    target: "F10"
    strong: true
  
  - source: "A10"
    target: "D9"
    strong: true
  
  - source: "B7"
    target: "E1"
    strong: true
  
  - source: "B8"
    target: "E7"
    strong: true
  
  - source: "B8"
    target: "H7"
    strong: true
  
  - source: "E5"
    target: "D1"
    strong: true
  
  - source: "E5"
    target: "E14"
    strong: true
  
  - source: "E6"
    target: "G1"
    strong: true
  
  - source: "E6"
    target: "F8"
    strong: true
  
  - source: "G11"
    target: "G3"
    strong: true
  
  - source: "G11"
    target: "C1"
    strong: true
  
  - source: "G12"
    target: "E5"
    strong: true
  
  - source: "G12"
    target: "D1"
    strong: true" fullWidth="true" %}
