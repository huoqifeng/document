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
 - https://github.com/IBM/charts/tree/master/stable/ibm-mongodb-dev 
 
 
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

- 自定义资源 (CRD).  

一种资源就是Kubernetes API中的一个端点，它存储着某种API 对象的集合。 
例如，内建的pods资源包含Pod对象的集合。

自定义资源是对Kubernetes API的一种扩展，它对于每一个Kubernetes集群不一定可用。
换句话说，它代表一个特定Kubernetes的定制化安装。

在一个运行中的集群内，自定义资源可以通过动态注册出现和消失，集群管理员可以独立于集群本身更新自定义资源。
一旦安装了自定义资源，用户就可以通过kubectl创建和访问他的对象，就像操作内建资源pods那样。

- 自定义控制器.  

自定义资源本身让你简单地存储和索取结构化数据。
只有当和控制器结合后，他们才成为一种真正的declarative API。 控制器将结构化数据解释为用户所期望状态的记录，并且不断地采取行动来实现和维持该状态。

定制化控制器是用户可以在运行中的集群内部署和更新的一个控制器，它独立于集群本身的生命周期。 定制化控制器可以和任何一种资源一起工作，当和定制化资源结合使用时尤其有效。

- 使用CRD(CustomResourceDefinitions)扩展Kubernetes API

当创建一个新的自定义资源定义（CRD）时，Kubernetes API Server 通过创建一个新的RESTful资源路径进行应答，无论是在命名空间还是在集群范围内，正如在CRD的scope域指定的那样。
与现有的内建对象一样，删除一个命名空间将会删除该命名空间内所有的自定义对象。
CRD本身并不区分命名空间，对所有的命名空间可用。

例如，如果将以下的CRD保存到resourcedefinition.yaml中:

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # version name to use for REST API: /apis/<group>/<version>
  version: v1
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

创建它：

```
kubectl create -f resourcedefinition.yaml
```

然后一个新的区分命名空间的RESTful API 端点被创建了：

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

然后可以使用此端点URL来创建和管理自定义对象。 这些对象的kind就是你在上面创建的CRD中指定的CronTab对象。


在CRD对象创建完成之后，你可以创建自定义对象了。自定义对象可以包含自定义的字段。这些字段可以包含任意的JSON。
以下的示例中，在一个自定义对象CronTab种类中设置了cronSpec和image字段。这个CronTab种类来自于你在上面创建的CRD对象。

```
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * /5"
  image: my-awesome-cron-image
```

创建它：

```
kubectl create -f my-crontab.yaml
```

你可以使用kubectl来管理你的CronTab对象。例如：

```
kubectl get crontab
```

应该打印这样的一个列表：

```
NAME                 KIND
my-new-cron-object   CronTab.v1.stable.example.com
```

要让CRD正确的工作，你还需要写一个Custom Controller，创建一个Custom Controller， 可以按照这个例子

```
https://github.com/kubernetes/sample-controller 
```

Custom Controller所做的事情就是通过Recouncile让声明的资源达到声明的状态，如下图。。。

![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/mongodbInK8s.imgs/k8s-crd-customcontroller.png)

参考：
 
 - https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
 - https://k8smeetup.github.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/
 - https://github.com/kubernetes/sample-controller
 - https://docs.openshift.com/container-platform/3.7/admin_guide/custom_resource_definitions.html
 - https://github.com/kubernetes/sample-controller 
 
  
### Operator framework

- Operators are Kubernetes applications

 > Conceptually, an Operator takes human operational knowledge and encodes it into software that is more easily packaged and shared with consumers. Think of an Operator as an extension of the software vendor’s engineering team that watches over your Kubernetes environment and uses its current state to make decisions in milliseconds. Operators follow a maturity model that ranges from basic functionality to having specific logic for an application. Advanced Operators are designed to handle upgrades seamlessly, react to failures automatically, and not take shortcuts, like skipping a software backup process to save time.

- The Operator Framework: Introducing the SDK, Lifecycle Management, and Metering
   
   1) Operator SDK: 

 Enables developers to build Operators based on their expertise without requiring knowledge of Kubernetes API complexities.
 
   2) Operator Lifecycle Management: 

 Oversees installation, updates, and management of the lifecycle of all of the Operators (and their associated services) running across a Kubernetes cluster.
 
   3) Operator Metering (joining in the coming months): 

 Enables usage reporting for Operators that provide specialized services.
 
 
- Steps to create a new operator
 
   1) Create a new project.  
   2) Manager.  
   > The main program for the operator cmd/manager/main.go initializes and runs the Manager.
The Manager will automatically register the scheme for all custom resources defined under pkg/apis/... and run all controllers under pkg/controller/....

   3) Add a new Custom Resource Definition   
   4) Define the resource spec and status.   
   5) Add a new Controller
 

参考：

  - https://coreos.com/blog/introducing-operator-framework
  - https://github.com/operator-framework/getting-started
 
  

### MongoDB operator

参考：

 - https://docs.opsmanager.mongodb.com/current/tutorial/install-k8s-operator/
 - https://github.com/mongodb/mongodb-enterprise-kubernetes
 - https://github.com/operator-framework/awesome-operators

 
  
## Cloud Based MongoDB -- current status

### AWS

参考：

 - https://docs.aws.amazon.com/quickstart/latest/mongodb/architecture.html
 
### Azure

参考：

 - https://azure.microsoft.com/en-us/resources/templates/mongodb-sharding-centos/
 - https://azure.microsoft.com/en-us/resources/templates/mongodb-replica-set-centos/
 
### GCE

参考：

 - https://cloud.google.com/solutions/deploy-mongodb 
  
### IBM Compose
 
参考：
 
 - https://www.compose.com/databases/mongodb  



