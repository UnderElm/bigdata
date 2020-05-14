基于 EMR、Airlfow 、Superset 的大数据调度报表平台搭建



>  背景 ：由于原产品团队从母公司剥离，独立运营在国内做视频类型社交产品，失去了母公司数据仓库，数据开发的支持，仅有我（一个菜鸡数据分析师）需要独自承担搭建大数据查询平台，调度平台，报表平台的任务。



### EMR

所幸阿里云的EMR的集成度已经很高，无需额外自己单独配置组件，之前用的AWS总感觉有点水土不服。

选型方面：选择了 1 Master 2 Core 的配置，硬件均为 8C + 16G 的配置，后续根据业务需要动态增加 Core 节点和 Task 节点。（其实我本来发给运维的需求单是 1 Master 3 Core 而且 Master 硬件也较高，主要是怼不过运维，被打成了 1+2 ），集群类型为 Hadoop 预计一年成本在三万左右。

|  实例  | 数量 | 配置                    | 系统盘        | 数据盘         |
| :----: | :--: | ----------------------- | ------------- | -------------- |
| master |  1   | ecs.c6.2xlarge 8C + 16G | 120G 高效云盘 | 50G*4 高效云盘 |
|  core  |  2   | ecs.c6.2xlarge 8C + 16G | 120G 高效云盘 | 50G*4 高效云盘 |

除了必选的组件外，还额外选配了 Zookeeper 和 Superset 组件。

具体可参考阿里云官网EMR：

 https://emr.console.aliyun.com/?spm=5176.cnemapreduce.0.0.18c13a1cvhWwFH&accounttraceid=a70a7ef900a7493db242e8da48fb5a18vvgb#/cn-hangzhou/cluster/create



1、由于EMR的组件安装在系统盘，所以务必不要将系统盘选择的过小，之前运维坚持了 40G 系统盘，马上就用完了开始报错，好在按量付费测试，后面重新换了120G 。

2、由于不准备将数据放在数据盘，准备将数据存在 oss，所以数据盘选择的很小，你可以像访问 hdfs:// 一样方便访问 oss://。



### Airflow

额外购置了一台 ECS 作为调度机来使用，硬件为 4C + 8G，同样被运维砍了一刀，/卑微 ，预计一年成本不到一万吧，需要放到EMR同一内网。有预算还是上 16C + 32G 吧。

调度平台选择了 Airflow，http://airflow.apache.org/

Airflow 是 Airbnb 家基于DAG(有向无环图)的任务管理系统，使用 python 语言开发，灵活性上具有一定的优势。而对比其他调度平台，Azkaban 通过压缩包上传任务的形式感觉有点奇怪，Oozie 不太熟。

Airflow 具体的安装可以参考之前写的另一篇文章

[https://underelm.github.io/2020/04/01/Airflow%20%E5%AE%89%E8%A3%85%E7%AC%94%E8%AE%B0/](https://underelm.github.io/2020/04/01/Airflow 安装笔记/)

其中没提到的部分是

* 时区问题，我安装的是 1.10.9 版本，原生是 UTC 时区，需要改为国内东八区时间，具体可以参考修改源码的方式。https://blog.csdn.net/crazy__hope/article/details/83688986
* 由于后续需要在调度机 把 beeline 命令封装 到 Airflow 的Bash_Operator，需要上传解压 Hadoop 和 HIve 包，版本最好与 EMR 的版本一致，我之前是直接 EMR SCP 环境到调度机，亲测不行，猜测阿里云把很多参数都变成了动态。所以最后还是到Apache 官网下载解压了对应版本，这里就不附上链接了。

* 数据库的密码建议修改为简单密码，同样在之前写的另一篇文章有提及。

* 在插入 Airflow mysql 时可能会报时间长度的错，这是由于mysql 的 STRICT_TRANS_TABLES 规则，

  可以通过 

  ```mysql
  SELECT @@GLOBAL.sql_mode;
  
  # 即把 STRICT_TRANS_TABLES 这条规则去掉
  SET @@GLOBAL.sql_mode="ONLY_FULL_GROUP_BY,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION";
  
  ```



#### 日志文件ETL

数据ETL上，服务端和客户端的主要日志会上传到oss，需要跟他们约定好日志的上传格式（这样可以清洗出来公共参数和动态参数），通过python 编写 MapJob 任务清洗到 oss 下存放元数据的 Object。这一步骤主要通过 "ssh emr-master" + "执行MapJob的 bash 命令 " 传递给 Airflow 的 Bash_Operator 来执行。



#### 关系型数据库同步（datax）

关系型数据库的同步上，用的是阿里的 datax 组件 https://github.com/alibaba/DataX，支持的数据库比较多，但是，有一说一 Github 的 issue 还是 文档都讲的很不清楚，有些甚至打不开页面，issue 甚至基本上没人维护，只能靠自己摸索。端口上，EMR master 需要向 调度机开放 defaultFS 端口（9000），每个worker 需要向 调度机开放 （50010） 端口。才能使得 datax 的 hdfswriter 正常工作，且 datax 所在机器需要与 EMR 同在一个内网。

如果调度平台安装在外网，那你最好看一下这个 issue：

 https://hadoop.apache.org/docs/r2.8.0/hadoop-project-dist/hadoop-hdfs/HdfsMultihoming.html#Clients_use_Hostnames_when_connecting_to_DataNodes



### Superset 

 http://superset.apache.org/ 

同样是 Airbnb 开源的轻型BI工具，提供了较为丰富的数据源接入，和报表类型。

我这里由于在买EMR的时候就选择了该组件，所以就可以直接把端口开出来直接用了，由于Superset 搭在了EMR-master 上，这边数据源查询就直接用了Sparksql，基础表就不得不放在hive了。

Superset 的搭建还是比较方便的，稍一摸索就能掌握基本的逻辑，用户权限分配上也较为简单。就是由于是 Hive 查询，整个 Dashboard 出来就很慢，据说用 EMR 的 Druid 集群非常快。



### 最后

在此感谢Airbnb，由于疫情打断了原先的出行计划，只是写了一封邮件给Airbnb，Airbnb就秒退了订单，甚至对方不是国内团队，用英语 Reply，Google translate，都能快速的解决困扰，相反国内的一些企业，甚至国企，就很呵呵，扯远了。

至此整套轻量级框架就搭建完了，初步达到了能查能看能用的水平吧，早期自己也经常加班到深夜解决问题，有时候因为解决不了而失眠，但好在是最终问题都迎刃而解了。

















