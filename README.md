# redshift-advanced

## 1.Migration
Checklist for a complete evaluation
Make sure that a complete evaluation meets all your data warehouse needs. Consider including the following items in your success criteria:
* Data load time – using the COPY command is a common way to test how long it takes to load data. For more information, see Amazon Redshift best practices for loading data.
* Throughput of the cluster – measuring queries per hour is a common way to determine throughput. To do so, set up a test to run typical queries for your workload.
* Data security – you can easily encrypt data at rest and in transit with Amazon Redshift. You also have a number of options for managing keys. Amazon Redshift also supports single sign-on integration. Amazon Redshift pricing includes built-in security, data compression, backup storage, and data transfer.
* Third-party tools integration – you can use either a JDBC or ODBC connection to integrate with business intelligence and other external tools.
* Interoperability with other AWS services – Amazon Redshift integrates with other AWS services, such as Amazon EMR, Amazon QuickSight, AWS Glue, Amazon S3, and Amazon Kinesis. You can use this integration when setting up and managing your data warehouse.
* Backups and snapshots – backups and snapshots are created automatically. You can also create a point-in-time snapshot at any time or on a schedule. Try using a snapshot and creating a second cluster as part of your evaluation. Evaluate if your development and testing organizations can use the cluster.
* Resizing – your evaluation should include increasing the number or types of Amazon Redshift nodes. Evaluate that the workload throughput before and after a resize meets any variability of the volume of your workload. For more information, see Resizing clusters in Amazon Redshift in the Amazon Redshift Cluster Management Guide.
* Concurrency scaling – this feature helps you handle variability of traffic volume in your data warehouse. With concurrency scaling, you can support virtually unlimited concurrent users and concurrent queries, with consistently fast query performance. For more information, see Working with concurrency scaling.
* Automatic workload management (WLM) – prioritize your business critical queries over other queries by using automatic WLM. Try setting up queues based on your workloads (for example, a queue for ETL and a queue for reporting). Then enable automatic WLM to allocate the concurrency and memory resources dynamically. For more information, see Implementing automatic WLM.
* Amazon Redshift Advisor – the Advisor develops customized recommendations to increase performance and optimize costs by analyzing your workload and usage metrics for your cluster. Sign in to the Amazon Redshift console to view Advisor recommendations. For more information, see Working with recommendations from Amazon Redshift Advisor.
* Table design – Amazon Redshift provides great performance out of the box for most workloads. When you create a table, the default sort key and distribution key is AUTO. For more information, see Working with automatic table optimization.
* Support – we strongly recommend that you evaluate AWS Support as part of your evaluation. Also, make sure to talk to your account manager about your proof of concept. AWS can help with technical guidance and credits for the proof of concept if you qualify. If you don't find the help you're looking for, you can talk directly to the Amazon Redshift team. For help, submit the form at Request support for your Amazon Redshift proof-of-concept.
* Lake house integration – with built-in integration, try using the out-of-box Amazon Redshift Spectrum feature. With Redshift Spectrum, you can extend the data warehouse into your data lake and run queries against petabytes of data in Amazon S3 using your existing cluster. For more information, see Querying external data using Amazon Redshift Spectrum.

https://docs.aws.amazon.com/redshift/latest/dg/proof-of-concept-playbook.html

## 2.Table Design
Taking advantage of Amazon Redshift data lake integration
-denormalized：
	大表存s3，中小表存在redshift
	如果top的复杂查询中出现的大表，还是建议存储在redshift中

Amazon Redshift is tightly integrated with other AWS-native services such as Amazon S3 which let’s the Amazon Redshift cluster interact with the data lake in several useful ways.
Amazon Redshift Spectrum lets you query data directly from files on Amazon S3 through an independent, elastically sized compute layer. Use these patterns independently or apply them together to offload work to the Amazon Redshift Spectrum compute layer, quickly create a transformed or aggregated dataset, or eliminate entire steps in a traditional ETL process.
* Use the Amazon Redshift Spectrum compute layer to offload workloads from the main cluster, and apply more processing power to the specific SQL statement. Amazon Redshift Spectrum automatically assigns compute power up to approximately 10 times the processing power of the main cluster. This may be an effective way to quickly process large transform or aggregate jobs.
* Skip the load in an ELT process and run the transform directly against data on Amazon S3. You can run transform logic against partitioned, columnar data on Amazon S3 with an INSERT … SELECT statement. It’s easier than going through the extra work of loading a staging dataset, joining it to other tables, and running a transform against it.
* Use Amazon Redshift Spectrum to run queries as the data lands in Amazon S3, rather than adding a step to load the data onto the main cluster. This allows for real-time analytics.
* Land the output of a staging or transformation cluster on Amazon S3 in a partitioned, columnar format. The main or reporting cluster can either query from that Amazon S3 dataset directly or load it via an INSERT … SELECT statement.
Within Amazon Redshift itself, you can export the data into the data lake with the UNLOAD command, or by writing to external tables. Both options export SQL statement output to Amazon S3 in a massively parallel fashion. You can do the following:
* Using familiar CREATE EXTERNAL TABLE AS SELECT and INSERT INTO SQL commands, create and populate external tables on Amazon S3 for subsequent use by Amazon Redshift or other services participating in the data lake without the need to manually maintain partitions. Materialized views can also cover external tables, further enhancing the accessibility and utility of the data lake.
* Using the UNLOAD command, Amazon Redshift can export SQL statement output to Amazon S3 in a massively parallel fashion. This technique greatly improves the export performance and lessens the impact of running the data through the leader node. You can compress the exported data on its way off the Amazon Redshift cluster. As the size of the output grows, so does the benefit of using this feature. For writing columnar data to the data lake, UNLOAD can write partition-aware Parquet data.

distribution key设计
四种distribution方式：
-Auto: 默认的方式，会在all和even之间选择一种方式
-All: 整表会存储在compute node上的第一个slice中。（如果表经常被写，并且你不能承受由此带来的性能下降（等于没有并发处理吞吐的能力），那么all不应该被考虑）（all模式，单表的所有数据都存储在一个slice中，那么集群的扩容和表的数据分布、读写性能都没有关系。）
-Even: 利用round-robin算法将表的数据均匀地存储在compute node上的slice的RMS上（以RA3为例）
-key: 通过key，将数据存储在slice的RMS上（以RA3为例）
	比如，key的cardinality值有100个，集群的slice有10个，那么每个slice上会被分配10个表片段；
	如果key的cardinality值有100个，而集群的slice数量有150个，那么会有50个slice不会被分配到表的数据片段。这个key的选择不是最优的。
如何选择distribution key，请见：
Aws.amazon.com/blogs/big-data/amazon-redshift-engineerings-advanced-table-design-playbook-distribution-styles-and-distribution-keys/

列的数据是否均匀的分布？--> 列是否具有高cardinality值—> query的filter会不会用到这个列，如果不会，那么这列是合适的key选项
如果会，继续看这一列是不是compound sortkey中的一列—>会不会有利于merge join的发生？ 如果会，那么这列同样也可以作为key的选项

 merge join、harsh join、nested loop join
https://dev.to/mahmoudhossam917/nested-join-vs-hash-join-vs-merge-join-in-postgresql-1ha6


Key选择不好，可能会造成数据倾斜（data skew），进而会导致query的时候出现slice的过热。
时间字段不是个key的好选择，因为，时间字段往往会出现在filter之中。但，如果sortkey中也有时间字段的话，join的时候应该会出现merge join，如此，则选择时间作为key也是好的。时间字段的cardinality可能不够高，这个时候可以考虑把时间字段的granularity提高，用timestamp来表示，这样cardinality就上去了。



Sort key设计
-Compound sort key
联合key，每个column的权重不同，第一个最大
能促使merge join发生的key

-Interleaved sort key
每个column的权重相同
￼
￼

https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data.html


Aws.amazon.com/blogs/big-data/amazon-redshift-engineerings-advanced-table-design-playbook-compound-and-interleaved-sort-keys/


compression选择
redshift支持多种compression方式：A64，Zstd，LZO等等。不同的数据类型选择的compression方式不同
https://docs.aws.amazon.com/redshift/latest/dg/c_Compression_encodings.html

一般情况下，选择compression都会提高性能（默认是LZO），也有特殊的情况，见下面的链接说明。
https://aws.amazon.com/cn/blogs/big-data/amazon-redshift-engineerings-advanced-table-design-playbook-compression-encodings/

可以通过analyze compression命令查看字段的compression情况和效果
https://docs.aws.amazon.com/redshift/latest/dg/r_ANALYZE_COMPRESSION.html

## 3.Performance Optimization
https://aws.amazon.com/cn/blogs/big-data/top-10-performance-tuning-techniques-for-amazon-redshift/

Tip #1: Precomputing results with Amazon Redshift materialized views

Tip #2: Handling bursts of workload with concurrency scaling and elastic resize

Tip #3: Using the Amazon Redshift Advisor to minimize administrative work

Distribution key recommendation
Sort key recommendation
Table compression recommendation
Table statistics recommendation

Tip #4: Using Auto WLM with priorities to increase throughput

Tip #5: Taking advantage of Amazon Redshift data lake integration

Amazon Redshift is tightly integrated with other AWS-native services such as Amazon S3 which let’s the Amazon Redshift cluster interact with the data lake in several useful ways.
Amazon Redshift Spectrum lets you query data directly from files on Amazon S3 through an independent, elastically sized compute layer. Use these patterns independently or apply them together to offload work to the Amazon Redshift Spectrum compute layer, quickly create a transformed or aggregated dataset, or eliminate entire steps in a traditional ETL process.
* Use the Amazon Redshift Spectrum compute layer to offload workloads from the main cluster, and apply more processing power to the specific SQL statement. Amazon Redshift Spectrum automatically assigns compute power up to approximately 10 times the processing power of the main cluster. This may be an effective way to quickly process large transform or aggregate jobs.
* Skip the load in an ELT process and run the transform directly against data on Amazon S3. You can run transform logic against partitioned, columnar data on Amazon S3 with an INSERT … SELECT statement. It’s easier than going through the extra work of loading a staging dataset, joining it to other tables, and running a transform against it.
* Use Amazon Redshift Spectrum to run queries as the data lands in Amazon S3, rather than adding a step to load the data onto the main cluster. This allows for real-time analytics.
* Land the output of a staging or transformation cluster on Amazon S3 in a partitioned, columnar format. The main or reporting cluster can either query from that Amazon S3 dataset directly or load it via an INSERT … SELECT statement.
Within Amazon Redshift itself, you can export the data into the data lake with the UNLOAD command, or by writing to external tables. Both options export SQL statement output to Amazon S3 in a massively parallel fashion. You can do the following:
* Using familiar CREATE EXTERNAL TABLE AS SELECT and INSERT INTO SQL commands, create and populate external tables on Amazon S3 for subsequent use by Amazon Redshift or other services participating in the data lake without the need to manually maintain partitions. Materialized views can also cover external tables, further enhancing the accessibility and utility of the data lake.
* Using the UNLOAD command, Amazon Redshift can export SQL statement output to Amazon S3 in a massively parallel fashion. This technique greatly improves the export performance and lessens the impact of running the data through the leader node. You can compress the exported data on its way off the Amazon Redshift cluster. As the size of the output grows, so does the benefit of using this feature. For writing columnar data to the data lake, UNLOAD can write partition-aware Parquet data.

Tip #6: Improving the efficiency of temporary tables

Tip #7: Using QMR and Amazon CloudWatch metrics to drive additional performance improvements

Tip #8: Federated queries connect the OLAP, OLTP and data lake worlds

Tip #9: Maintaining efficient data loads

Tip #10: Using the latest Amazon Redshift drivers from AWS


有用的可以诊断查询优化的sql：
https://docs.aws.amazon.com/redshift/latest/dg/diagnostic-queries-for-query-tuning.html

Identifying queries that are top candidates for tuning

Identifying tables with data skew or unsorted rows

Identifying queries with nested loops

Reviewing queue wait times for queries

Reviewing query alerts by table

Identifying tables with missing statistics


## 4.Trouble Shooting
trouble-shooting

query hung

lock
如果query被lock住，可以通过sql query，在svv_transaction, pg_locks, stv_tbl_perm, pg_class表中查询到被lock的query id。
然后将这个pid终止。
Aws.amazon.com/premiumsupport/knowledge-center/prevent-locks-blocking-queries-redshift/



集群性能下降
https://aws.amazon.com/cn/premiumsupport/knowledge-center/redshift-cluster-degrade/

-监控集群的性能指标
￼
-查看redshift advisor的建议


-查看查询执行的告警和多余磁盘的使用使用情况

https://aws.amazon.com/cn/premiumsupport/knowledge-center/redshift-high-disk-usage/


-检查lock的情况

-检查WLM的情况
https://aws.amazon.com/cn/premiumsupport/knowledge-center/redshift-wlm-memory-allocation/


-检查集群节点的硬件维护和性能


Redshift developer guide:
https://docs.amazonaws.cn/en_us/redshift/latest/dg/redshift-dg.pdf


Disk spill
-查询内存不够会spill到disk上
-temporary table会在disk上

STL 视图：
* STL_AGGR
* STL_ALERT_EVENT_LOG
* STL_ANALYZE
* STL_ANALYZE_COMPRESSION



STV 系统表中的数据的子集
