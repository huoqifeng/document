# MongoDB in Kubernetes （In Progress...）

## What's MongoDB

MongoDB 是一个基于分布式文件存储的数据库。由C++语言编写。旨在为WEB应用提供可扩展的高性能数据存储解决方案。

MongoDB是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。他支持的数据结构非常松散，是类似json的bson格式，因此可以存储比较复杂的数据类型。Mongo最大的特点是他支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。

“面向集合”（Collection-Oriented），意思是数据被分组存储在数据集中，被称为一个集合（Collection)。每个集合在数据库中都有一个唯一的标识名，并且可以包含无限数目的文档。集合的概念类似关系型数据库（RDBMS）里的表（table），不同的是它不需要定义任何模式（schema)。Nytro MegaRAID技术中的闪存高速缓存算法，能够快速识别数据库内大数据集中的热数据，提供一致的性能改进。模式自由（schema-free)，意味着对于存储在mongodb数据库中的文件，我们不需要知道它的任何结构定义。如果需要的话，你完全可以把不同结构的文件存储在同一个数据库里。存储在集合中的文档，被存储为键-值对的形式。键用于唯一标识一个文档，为字符串类型，而值则可以是各种复杂的文件类型。我们称这种存储形式为BSON（Binary Serialized Document Format）。

参考： 

 - http://www.runoob.com/mongodb/mongodb-databases-documents-collections.html
 - https://m.w3cschool.cn/mongodb/
 - https://docs.mongodb.com/manual/tutorial/getting-started/


## Deployment of MongoDB

### Standalone

Standalone 是最简单的部署方式，只有一个节点，在开发和测试环境中可以使用，生产环境有诸多限制。

### Replica Set (master-slaves)
现实中，生产环境用的最左的是Replica Set的模式，在Replica Set模式下， MongoDB 有这么几种节点：

 - Primary （唯一的写节点）
 - Secondary （多个备份节点，保存完整的数据，可以单独提供读服务， 当Primary挂掉的时候，其中的一个Secondary会被选举成Primary）
 - Arbitor （Failover的时候用于选举新的Master）
 - Hidden Secondary （隐藏的Secondary， 用于Disaster Recovery）

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-replicaset-1.png) 

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-replicaset-2.png) 

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-replicaset-3.png) 

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-replicaset-4.png) 
 

参考：
 
 -  https://docs.mongodb.com/manual/replication/
  

### Sharding


Sharding is a method for distributing data across multiple machines. MongoDB uses sharding to support deployments with very large data sets and high throughput operations.

MongoDB shards data at the collection level, distributing the collection data across the shards in the cluster.

- Shard Keys.  

To distribute the documents in a collection, MongoDB partitions the collection using the shard key. The shard key consists of an immutable field or fields that exist in every document in the target collection.

You choose the shard key when sharding a collection. The choice of shard key cannot be changed after sharding. A sharded collection can have only one shard key. See Shard Key Specification.

To shard a non-empty collection, the collection must have an index that starts with the shard key. For empty collections, MongoDB creates the index if the collection does not already have an appropriate index for the specified shard key. See Shard Key Indexes.

The choice of shard key affects the performance, efficiency, and scalability of a sharded cluster. A cluster with the best possible hardware and infrastructure can be bottlenecked by the choice of shard key. The choice of shard key and its backing index can also affect the sharding strategy that your cluster can use.

See the shard key documentation for more information.

- Chunks.  

MongoDB partitions sharded data into chunks. Each chunk has an inclusive lower and exclusive upper range based on the shard key.


- Connecting to a Sharded Cluster

You must connect to a mongos router to interact with any collection in the sharded cluster. This includes sharded and unsharded collections. Clients should never connect to a single shard in order to perform read or write operations.

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-shard-1.png) 

- Hashed Sharding

Hashed Sharding involves computing a hash of the shard key field’s value. Each chunk is then assigned a range based on the hashed shard key values.

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-shard-2.png) 
 
- Ranged Sharding

Ranged sharding involves dividing data into ranges based on the shard key values. Each chunk is then assigned a range based on the shard key values. 

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-shard-3.png) 

- Zones in Sharded Clusters

In sharded clusters, you can create zones of sharded data based on the shard key. You can associate each zone with one or more shards in the cluster. A shard can associate with any number of zones. In a balanced cluster, MongoDB migrates chunks covered by a zone only to those shards associated with the zone.

Each zone covers one or more ranges of shard key values. Each range a zone covers is always inclusive of its lower boundary and exclusive of its upper boundary.

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-shard-4.png) 

参考：
 
 - https://docs.mongodb.com/manual/sharding/

## Mongodb Deployment in Kubernetes

### Stateless vs Stateful

Stateless is Easy, Stateful is Hard.  

With Kubernetes, it is relatively easy to manage and scale web apps, mobile backends, and API services right out of the box. Why? Because these applications are generally stateless, so the basic Kubernetes APIs, like Deployments, can scale and recover from failures without additional knowledge.

A larger challenge is managing stateful applications, like databases, caches, and monitoring systems. These systems require application domain knowledge to correctly scale, upgrade, and reconfigure while protecting against data loss or unavailability. We want this application-specific operational knowledge encoded into software that leverages the powerful Kubernetes abstractions to run and manage the application correctly.


### Standalone

参考：

 - https://developer.ibm.com/recipes/tutorials/deploy-mongodb-into-ibm-cloud-private/
 
 
### Replica Set

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-k8s-replicaset.png) 

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-k8s-replicaset-pods.png) 

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-k8s-replicaset-svc.png) 


参考：
 
 - https://developer.ibm.com/tutorials/cl-deploy-mongodb-replica-set-using-ibm-cloud-container-service/
 - https://www.mongodb.com/blog/post/running-mongodb-as-a-microservice-with-docker-and-kubernetes
 - https://kubernetes.io/blog/2017/01/running-mongodb-on-kubernetes-with-statefulsets/
 
### Sharding

参考文档的例子基于K8s创建了如下的pods和Mongo Componenets

This tallies up with what Kubernetes was asked to be deployed, namely:
 
 - 3x DaemonSet startup-script containers (one per host machine)
 - 3x Config Server mongod containers
 - 3x Shards, each composed of 3 replica mongod containers
 - 2x Router mongos containers

K8s 的pod如下：
 
 ![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/mongo-k8s-shard-1.png) 

参考：

 - http://pauldone.blogspot.com/2017/07/sharded-mongodb-kubernetes.html
 

## More smarter deployment method -- operator

上面讲了MongoDB的三种部署方式以及如何在K8s里面部署这三种方式，但是正如前面讲到的，Stateful resource是比较难以管理的，需要很多手工或者半自动的方式来完成，感谢K8s引入的CRD和社区引入的Operator-Framework，我们可以用更Smart的方式解决Stateful Resurce的管理。。。

下面我们来看看这些。。。

### CRD

参考：
 
 - https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
  
### Operator framework

参考：

  - https://coreos.com/blog/introducing-operator-framework
  - https://github.com/operator-framework/getting-started
  

### Mongodb operator

参考：

 - https://docs.opsmanager.mongodb.com/current/tutorial/install-k8s-operator/
 - https://github.com/operator-framework/awesome-operators



