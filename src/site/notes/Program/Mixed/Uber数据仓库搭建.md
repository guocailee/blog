---
{"dg-publish":true,"permalink":"/Program/Mixed/Uber数据仓库搭建/","noteIcon":"","created":"2025-01-02T16:22:56.395+08:00"}
---

> 作者：dbaplus社群  
链接：https://www.zhihu.com/question/48945137/answer/69613034608  
来源：知乎  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。  

## **概述**

  

前段时间，我们经历了 [Uber](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Uber&zhida_source=entity) 大数据基础设施的演变，并总结了这一转变每个阶段的挑战、解决方案和创新。

  

注意：本文主要关注 Uber 的批处理，想要了解实时处理细节可查看：[https://vutr.substack.com/p/i-spent-7-hours-understanding-ubers?r=2rj6sg&utm_campaign=post&utm_medium=web&triedRedirect=true](https://link.zhihu.com/?target=https%3A//vutr.substack.com/p/i-spent-7-hours-understanding-ubers%3Fr%3D2rj6sg%26utm_campaign%3Dpost%26utm_medium%3Dweb%26triedRedirect%3Dtrue)

  

此外，本文引用了 Uber 2018 年的原始文章，因此一些细节和数据可能已经发生了变化。

  

## **数据增长的挑战（第一代）**

  

Uber 早期的数据管理很简单。2014 年之前，公司的数据量很小，可以放在几个 [MySQL](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=MySQL&zhida_source=entity) 和 [PostgreSQL](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=PostgreSQL&zhida_source=entity) 数据库中。


此时，Uber 的数据也非常零散，分散在不同的数据库中。用户如果想要查看整体信息，必须手动整合数据。

  

  

![](https://picx.zhimg.com/v2-d6ff9e910f4653784da88e4fb9523bd3_1440w.jpg)

  

  

虽然这种设置一度奏效，但许多城市的乘客、司机和行程数量激增， Uber 的全球扩张需要更强大的数据解决方案。短时间内，Uber 需要处理的数据不只是 TB 级，甚至可能是 PB 级，需要以可靠且可扩展的方式进行处理。

  

## **第一个数据仓库**

  

![](https://picx.zhimg.com/v2-4c0799652d4040130dd96afd84a18abb_1440w.jpg)

  

Uber 通过建立其首个数据仓库实现了重大飞跃，这个新系统将 Uber 的所有数据集中到一个地方。由于其速度、可扩展性和面向列的设计，他们选择了 [Vertica](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Vertica&zhida_source=entity) 作为数据仓库解决方案。


借助 Vertica，Uber 的工程师将 SQL 标准化为数据访问的主要接口，使得公司范围内的数千名用户能够轻松运行查询并提取有价值的数据。

几个月后，Uber 的数据仓库增长到数十 TB，每天有数百名用户在查询系统。

然而，这又产生了新的问题。他们通过向 Vertica 提取数据的 ETL 作业，观察到了数据不可靠性。特别是，上游数据生产者和下游数据消费者之间缺乏正式的架构协议，导致源数据格式发生变化时提取经常失败。Uber 的数据通常以灵活的 JSON 格式存储，这使得很难强制执行架构一致性并导致数据管道频繁中断。

  

  

![](https://pic2.zhimg.com/v2-331d2a01a8354908b2c6bab58fb6fc6d_1440w.jpg)

  

此外，摄取作业缺乏标准化，导致相同的数据被以不同的转换方式多次摄取，给上游数据源带来额外的压力，并因重复数据而增加存储成本。


![](https://pic2.zhimg.com/v2-f56bcb6e8a179a9ef335519393275733_1440w.jpg)

  

## **[Hadoop](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Hadoop&zhida_source=entity) 作为数据湖（第二代）**

  

![](https://pic2.zhimg.com/v2-ea7cc2cd6f6911cc6e9b5587b1872f55_1440w.jpg)

  

继而，Uber他们选择 Hadoop 作为其下一代数据平台的核心；原始数据被导入基于 Hadoop 的数据湖，而不是直接将数据加载到 Vertica 数据仓库。

  

这为 Uber 的大数据平台带来了全新的范式，使 Uber 能够从各种来源获取和存储大量原始数据，且无需在获取时进行转换。

  

这一转变减轻了 Uber 源数据存储的负载，因为数据现在可以以其原生格式输入 Hadoop，而无需进行预处理。一旦数据进入 Hadoop，就可以使用各种工具对其进行转换和分析。

  

对于数据消费，选项不仅限于Vertica：

  

- 对于交互式查询，Uber 使用了 [Presto](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Presto&zhida_source=entity)，这是一个开源分布式 SQL 引擎，允许快速查询大型数据集。
- [Apache Spark](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Apache+Spark&zhida_source=entity) 是为了执行更复杂的数据处理任务而推出的，允许团队使用 SQL 或编程语言运行大规模作业。
- 还部署了 [Apache Hive](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Apache+Hive&zhida_source=entity) 来处理大型查询。

  

Uber 确保所有数据转换都在 Hadoop 中进行；只有实时 SQL 查询的关键表才会被保存在数据仓库中。这样可以快速进行回填和恢复；如果需要这些操作，他们只需要处理 Hadoop 中已有的数据，避免接触源数据。

  

Uber 向 Hadoop 转型的最关键方面是采用了 [Apache Parquet](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Apache+Parquet&zhida_source=entity)，这是一种列式存储格式，可显著节省存储空间并提高计算资源效率。Parquet 的列式特性使 Uber 能够更有效地压缩数据，从而降低存储成本并加快分析工作负载的查询性能。

当第二代数据平台全面投入使用时，该公司已将数十 PB 的数据导入其 Hadoop 数据湖。

### **第二代的挑战**

  

虽然 Hadoop 使 Uber 能够扩展其数据操作，但它并不完美。最大的挑战之一是 HDFS 中有大量小文件，这些文件来自摄取或临时批处理作业或 ETL 过程。这些文件的积累开始给负责管理文件系统元数据的 HDFS NameNode 带来压力。随着文件数量增长到数百万，NameNode 难以跟上。

  

![](https://picx.zhimg.com/v2-c04622b9eacc8bbfea7d322aef0eb07f_1440w.jpg)

  

另一个主要问题是数据延迟。当时，Uber 的数据每 24 小时才向用户提供一次，这对于该公司的许多实时业务需求来说太慢了。这种延迟限制了 Uber 在许多情况下做出及时决策的能力，例如需求预测和欺诈检测。

  

![](https://pica.zhimg.com/v2-4b1f5599863fe8a96986b8cc9fb40672_1440w.jpg)

  

最后，虽然 Hadoop 解决了许多可扩展性问题，但它不支持数据更新或删除。例如，乘客和司机评级、行程费用调整和其他实时数据必须频繁更新，以确保准确的报告和分析。然而，Hadoop 基于快照的提取模型，意味着 Uber 每次进行小更新时都必须从源重新加载整个数据集，这非常低效耗时。

  

## **开发 [Hudi](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Hudi&zhida_source=entity)（第三代）**

  

![](https://pic4.zhimg.com/v2-02316343a31a55611c3715fac1151d2f_1440w.jpg)

  

为了解决第二代产品带来的这些挑战，他们花了大量时间来确定下一代产品需要解决的四个主要痛点：

  

- **HDFS 可扩展性限制：**当数据超过 10 PB 时，HDFS 的 NameNode 会遇到困难，超过 50-100 PB 时情况会更糟。ViewFS 和 HDFS NameNode Federation 等解决方案以及将数据移动到单独的集群可以缓解这些问题。
- **HDFS ViewFS ：**ViewFS 在 Hadoop 中提供虚拟文件系统，允许用户通过统一命名空间访问多个 HDFS 集群或目录。它通过创建无缝的单一访问点简化了使用各种 HDFS 位置的工作。
- **HDFS NameNode 联合：**NameNode 联合通过使用多个独立的 NameNode（每个 NameNode 管理命名空间的一部分）来提高 HDFS 的可扩展性。这可以减少瓶颈、增强容错能力并支持更大规模的部署。
- **Hadoop 中的数据传输速度更快：**Uber 第二代平台的 24 小时数据延迟对于实时需求来说太慢了。为了加快交付速度，Uber 不得不重新设计其管道，以便增量提取更新和新数据，而不是加载完整快照。
- **支持 Hadoop/Parquet 中的更新和删除：**Uber 的数据涉及频繁更新，但基于快照的提取效率不高。
- **更快的 ETL 和建模：**与原始数据提取一样，ETL 和建模作业每次运行时都会重建整个表。它们转向增量更新，只提取更改的数据并更新派生表而无需完全重建，从而减少了延迟。

  

考虑到这一点，Uber 开发了一个名为 Hudi（Hadoop Upserts and Incremental）的开源项目，从根本上改变了 Hadoop 生态系统中数据的提取和管理方式。

  

Hudi 引入了执行 upsert（更新插入）和增量数据提取的功能，使 Uber 摆脱了基于快照的提取方法。Hudi 使 Uber 无需每天重新加载整个数据集，而是仅提取更改（新记录、增量更新和删除），从而将数据延迟从 24 小时缩短至 1 小时以内。

这种增量方法提高了数据的新鲜度，并减少了处理更新所需的计算资源。例如，Uber 现在只需更新相关分区，而不必每次添加新数据时重新处理整个 100 TB 的数据集，从而显著提高效率。


除了创建 Hudi，Uber 还使用 Apache Kafka 简化了存储之间的数据更改。所有上游数据存储事件（包括来自各种服务的日志）都以统一的 Avro 编码发送到 Kafka。

[Marmaray](https://zhida.zhihu.com/search?content_id=707038210&content_type=Answer&match_order=1&q=Marmaray&zhida_source=entity) 是 Uber 的数据采集平台，以小批量方式运行并使用来自 Kafka 的变更日志。它通过 Hudi 将它们应用于现有的 Hadoop 数据，从而允许更新或删除记录。Marmaray 的后台是每 10-15 分钟运行一次的 Spark 作业，确保数据延迟保持在 30 分钟以下。

  

通过消除摄取阶段的转换需求，Marmaray 确保可以快速可靠地摄取原始数据，并在 Hadoop 中下游执行任何必要的转换。数据可靠性也得到了提高，因为他们可以避免摄取过程中容易出错的转换。

  

### **通用数据采集**

  

Uber 的上游数据存储数量随着时间的推移而增加。他们决定构建一个统一的摄取平台，以简化将原始数据摄取到 Hadoop 的过程。借助此平台，更新过程可以以 10-15 分钟的延迟逐步更新 Hadoop 表。Hudi 在这里起着至关重要的作用；它允许 ETL 作业仅从源表中获取已更改的数据。转换/建模作业只需在每次运行期间将检查点时间戳传递给 Hudi 读取器，即可从原始源表中接收新记录或更新记录流。

  

## **展望未来（第四代）**

  

随着第三代大数据平台的推出，Uber 的数据基础设施已经达到了强大、可扩展和高效的水平。在 Uber 数据之旅的下一阶段，计划重点关注四个关键领域：

  

 - **数据质量：**Uber 正在努力对上游数据源实施更严格的模式验证。
- **更快的数据访问：**他们旨在通过将原始数据延迟缩短至五分钟并将建模数据延迟缩短至十分钟来减少数据延迟。
- **运营效率：**Uber 放弃了专用硬件，转而采用容器化来提供其服务。这将使资源管理更加灵活，并确保能够更高效地在公司的 Hadoop 和非 Hadoop 服务中安排和执行作业。
- **可扩展性和可靠性：**Uber 通过标准化所有上游数据源的变更日志并采用更统一的数据提取方法，继续优化其数据提取平台，使其更具弹性和可扩展性。

  

在2024 年 5 月和9 月初的文章中，Uber 分享了他们正在将其批处理基础设施迁移到 Google Cloud。

  

DataMesh：Uber 如何为数据湖云迁移奠定基础[https://www.uber.com/en-VN/blog/datamesh](https://link.zhihu.com/?target=https%3A//www.uber.com/en-VN/blog/datamesh)/

  

## **总结**

在本文中，我们探索 Uber 数据平台从 MySQL 数据库到 Hadoop 集群以及 Apache Hudi 的创建历程，通过了解 Uber 的数据平台革命，希望能对大家的数据项目带来帮助。

  

**>>>>参考资料**

- [1]Reza Shiftehfar，Uber 大数据平台：100+PB 数据，延迟时间仅几分钟(2018)[https://www.uber.com/en-SG/blog/uber-big-data-platform/?uclick_id=15b6739c-0acd-406e-bdf6-884992beefa0](https://link.zhihu.com/?target=https%3A//www.uber.com/en-SG/blog/uber-big-data-platform/%3Fuclick_id%3D15b6739c-0acd-406e-bdf6-884992beefa0)  
    

  

作者丨Vu Trinh 编译丨Rio

来源丨[https://vutr.substack.com/p/ubers-big-data-revolution-from-mysql](https://link.zhihu.com/?target=https%3A//vutr.substack.com/p/ubers-big-data-revolution-from-mysql)

  

***本文为dbaplus社群编译整理，如需转载请取得授权并标明出处！**欢迎广大技术人员投稿，投稿邮箱：editor@dbaplus.cn