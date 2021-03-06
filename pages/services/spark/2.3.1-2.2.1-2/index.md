---
layout: layout.pug
title: Spark 2.3.1-2.2.1-2
navigationTitle: Spark 2.3.1-2.2.1-2
menuWeight: 0
excerpt:
featureMaturity:
model: /services/spark/data.yml
render: mustache
---

Welcome to the documentation for the DC/OS {{ model.techName }} service. For more information about new and changed features, see the [release notes](/services/spark/2.3.1-2.2.1-2/release-notes/).


{{ model.techName }} is a fast and general-purpose cluster computing system for big data. It provides high-level APIs in Scala, Java, Python, and R, and an optimized engine that supports general computation graphs for data analysis. It also supports a rich set of higher-level tools including Spark SQL for SQL and DataFrames, MLlib for machine learning, GraphX for graph processing, and Spark Streaming for stream processing. For more information, see the [Apache Spark documentation][1].

DC/OS {{ model.techName }} consists of [{{ model.techName }} with a few custom commits][17] along with [DC/OS-specific packaging][18].

DC/OS {{ model.techName }} includes:

*   [Mesos Cluster Dispatcher][2]
*   [Spark History Server][3]
*   DC/OS {{ model.techName }} CLI
*   Interactive {{ model.techShortName }} shell

# Benefits

*   Utilization: DC/OS {{ model.techName }} leverages Mesos to run Spark on the same cluster as other DC/OS services
*   Improved efficiency
*   Simple management
*   Multi-team support
*   Interactive analytics through notebooks
*   UI integration
*   Security, including file- and environment-based secrets

# Features

*   Multiversion support
*   Runs multiple {{ model.techShortName }} dispatchers
*   Runs against multiple HDFS clusters
*   Backports of scheduling improvements
*   Simple installation of all {{ model.techShortName }} components, including the dispatcher and the history server
*   Integration of the dispatcher and history server
*   Zeppelin integration
*   Kerberos and SSL support

# Related Services

*   [HDFS][4]
*   [Kafka][5]
*   [Zeppelin][6]

 [1]: http://spark.apache.org/documentation.html
 [2]: http://spark.apache.org/docs/latest/running-on-mesos.html#cluster-mode
 [3]: http://spark.apache.org/docs/latest/monitoring.html#viewing-after-the-fact
 [4]: https://docs.mesosphere.com/services/hdfs/
 [5]: https://docs.mesosphere.com/services/kafka/
 [6]: https://zeppelin.incubator.apache.org/
 [17]: https://github.com/mesosphere/spark
 [18]: https://github.com/mesosphere/spark-build
