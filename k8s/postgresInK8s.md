# PostgresSQL in Kubernetes

## What's PostgresSQL

- PostgreSQL

PostgreSQL 标榜自己是世界上最先进的开源数据库。PostgreSQL 的一些粉丝说它能与 Oracle 相媲美，而且没有那么昂贵的价格和傲慢的客服。它拥有很长的历史，最初是 1985 年在加利福尼亚大学伯克利分校开发的，作为 Ingres 数据库的后继。

PostgreSQL 是完全由社区驱动的开源项目，由全世界超过 1000 名贡献者所维护。它提供了单个完整功能的版本，而不像 MySQL 那样提供了多个不同的社区版、商业版与企业版。PostgreSQL 基于自由的 BSD/MIT 许可，组织可以使用、复制、修改和重新分发代码，只需要提供一个版权声明即可。

可靠性是 PostgreSQL 的最高优先级。它以坚如磐石的品质和良好的工程化而闻名，支持高事务、任务关键型应用。PostgreSQL 的文档非常精良，提供了大量免费的在线手册，还针对旧版本提供了归档的参考手册。PostgreSQL 的社区支持是非常棒的，还有来自于独立厂商的商业支持。

数据一致性与完整性也是 PostgreSQL 的高优先级特性。PostgreSQL 是完全支持 ACID 特性的，它对于数据库访问提供了强大的安全性保证，充分利用了企业安全工具，如 Kerberos 与 OpenSSL 等。你可以定义自己的检查，根据自己的业务规则确保数据质量。在众多的管理特性中，point-in-time recovery（PITR）是非常棒的特性，这是个灵活的高可用特性，提供了诸如针对失败恢复创建热备份以及快照与恢复的能力。但这并不是 PostgreSQL 的全部，项目还提供了几个方法来管理 PostgreSQL 以实现高可用、负载均衡与复制等，这样你就可以使用适合自己特定需求的功能了。

vs

- MySQL

MySQL 相对来说比较年轻，首度出现在 1994 年。它声称自己是最流行的开源数据库。MySQL 就是 LAMP（用于 Web 开发的软件包，包括 Linux、Apache 及 Perl/PHP/Python）中的 M。构建在 LAMP 栈之上的大多数应用都会使用 MySQL，包括那些知名的应用，如 WordPress、Drupal、Zend 及 phpBB 等。

一开始，MySQL 的设计目标是成为一个快速的 Web 服务器后端，使用快速的索引序列访问方法（ISAM），不支持 ACID。经过早期快速的发展之后，MySQL 开始支持更多的存储引擎，并通过 InnoDB 引擎实现了 ACID。MySQL 还支持其他存储引擎，提供了临时表的功能（使用 MEMORY 存储引擎），通过 MyISAM 引擎实现了高速读的数据库，此外还有其他的核心存储引擎与第三方引擎。

MySQL 的文档非常丰富，有很多质量不错的免费参考手册、图书与在线文档，还有来自于 Oracle 和第三方厂商的培训与支持。

MySQL 近几年经历了所有权的变更和一些颇具戏剧性的事件。它最初是由 MySQL AB 开发的，然后在 2008 年以 10 亿美金的价格卖给了 Sun 公司，Sun 公司又在 2010 年被 Oracle 收购。Oracle 支持 MySQL 的多个版本：Standard、Enterprise、Classic、Cluster、Embedded 与 Community。其中有一些是免费下载的，另外一些则是收费的。其核心代码基于 GPL 许可，对于那些不想使用 GPL 许可的开发者与厂商来说还有商业许可可供使用。

现在，基于最初的 MySQL 代码还有更多的数据库可供选择，因为几个核心的 MySQL 开发者已经发布了 MySQL 分支。最初的 MySQL 创建者之一 Michael "Monty" Widenius 貌似后悔将 MySQL 卖给了 Sun 公司，于是又开发了他自己的 MySQL 分支 MariaDB，它是免费的，基于 GPL 许可。知名的 MySQL 开发者 Brian Aker 所创建的分支 Drizzle 对其进行了大量的改写，特别针对多 CPU、云、网络应用与高并发进行了优化。

vs

- MariaDB

最初的 MySQL 创建者之一 Michael "Monty" Widenius 貌似后悔将 MySQL 卖给了 Sun 公司，于是又开发了他自己的 MySQL 分支 MariaDB，它是免费的，基于 GPL 许可。


参考：
 
 - http://www.postgres.cn/docs/9.4/tutorial.html
 - https://mariadb.org/
 - https://www.infoq.cn/article/2013%2F12%2Fmysql-vs-postgresql

 
 
## Deploy PostgresSQL in ICP
 
参考：
 
 - https://github.com/IBM/charts/tree/master/stable/ibm-postgres-dev
 - https://developer.ibm.com/recipes/tutorials/deploy-postgresql-into-ibm-cloud-private/
 
 
## PostgresSQL HA deployment
 
![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/postgresInK8s.imgs/ha-options.png)


参考：

 - https://www.postgresql.org/docs/9.5/high-availability.html
	

## Templates of PostgresSQL HA deployment


Patroni (MIT)

> Patroni is a template for you to create your own customized, high-availability solution using Python and - for maximum accessibility - a distributed configuration store like ZooKeeper, etcd or Consul. Database engineers, DBAs, DevOps engineers, and SREs who are looking to quickly deploy HA PostgreSQL in the datacenter - or anywhere else - will hopefully find it useful.

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/postgresInK8s.imgs/ha-patroni.png) 


Crunchy (Apache 2.0)

> Crunchy Container Suite provides Docker containers that enable rapid deployment of PostgreSQL, including administration and monitoring tools. Multiple styles of deploying PostgreSQL clusters are supported.

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/postgresInK8s.imgs/ha-crunchy.png) 

The project includes the following containers:

- crunchy-postgres - executes Postgres

- crunchy-postgres-gis - executes Postgres plus the PostGIS extensions

- crunchy-backup - performs a full database backup

- crunchy-pgpool - executes pgpool

- crunchy-pgbadger - executes pgbadger

- crunchy-collect - collects Postgres metrics

- crunchy-prometheus -stores Postgres metrics

- crunchy-grafana - graphs Postgres metrics

- crunchy-pgbouncer - pgbouncer connection pooler and simple form of failover

- crunchy-pgadmin4 - pgadmin4 web application

- crunchy-dba - implements a cron scheduler to perform simple DBA tasks

- crunchy-upgrade - allows you to perform a major postgres upgrade using pg_upgrade

- crunchy-backrest-restore - allows you to perform a pgbackrest restore

- crunchy-sim - executes queries over a specified interval range for Postgres traffic simulation purposes

- crunchy-pgdump - provides a means of performing either a pg_dump or pg_dumpall on a Postgres database

- crunchy-pgrestore - provides a means of performing a restore of a dump from pg_dump or pg_dumpall via psql or pg_restore to a Postgres container database

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/postgresInK8s.imgs/ha-crunchy1.png) 


Stolon (Apache 2.0)

> Stolon is a cloud native PostgreSQL manager for PostgreSQL high availability. It’s cloud native because it’ll let you keep an high available PostgreSQL inside your containers (kubernetes integration) but also on every other kind of infrastructure (cloud IaaS, old style infrastructures etc…)

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/postgresInK8s.imgs/ha-stolon.png) 

Stolon is made up of 3 main components:

- keeper: it manages a PostgreSQL instance converging to the clusterview provided by the sentinel(s).
- sentinel: it discovers and monitors keepers and calculates the optimal clusterview.
- proxy: the client’s access point. It enforces connections to the right PostgreSQL master and forcibly closes connections to unelected masters.
Stolon uses etcd or consul as a main storage for cluster state.


参考：

 - https://github.com/zalando/patroni
 - https://patroni.readthedocs.io/en/latest/
 - https://github.com/CrunchyData/crunchy-containers
 - https://crunchydata.github.io/crunchy-containers/stable/
 - https://github.com/sorintlab/stolon
 

## K8s Operators 

参考：

 - https://github.com/operator-framework/awesome-operators
 - https://github.com/zalando-incubator/postgres-operator （Patroni, 300+ stars）
 - https://github.com/CrunchyData/postgres-operator （Crunchy, 500+ stars）
 - https://github.com/sorintlab/stolon  (Stolon, 1800+ stars)