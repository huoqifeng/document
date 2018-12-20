# Helm Chart in Kubernetes and ICP (IBM Cloud Private)

本文基于 IBM Cloud Private（ICP）3.1.0 和ICP自带的Kubernetes 1.11.1.  
参考： https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/getting_started/whats_new.html

## K8s预备知识
### K8s架构
 - 架构
 - 代码结构

 参考：

### K8s里面的一些术语
Deployment. 
Service. 
ReplicaSet. 
Pod. 
Container. 
Ingress. 

### Deployment如何管理Pods

### Service如何导流
### Pod的生命周期以及Liveness和Readiness
### 

## HelmChart 基础
### 从一个例子开始
 - .tpl and .yaml
 -  

 参考： https://github.com/IBM/charts/tree/master/stable/ibm-cem/charts/ibm-sch
 
### 与标准Helm Chart 比，ICP里多了什么？
 - values-metadata.yanl
 - ibm_cloud_ppa/maniefest.yaml
 - sch

 
 
 
### Helm Chart 最佳实践
参考：  
 
 - https://github.com/helm/helm/tree/master/docs   
 - https://github.com/helm/helm/tree/master/docs/chart_best_practices  
 - https://github.com/ibm/charts  

## 在ICP上的准备工做
### 安装ICP
假定已经安装了ICP。  
安装参考：https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.0/installing/install_containers_CE.html#prep_boot

### 安装 cloudctl
cloudcli 是一个命令行工具，安装cloudcli的同时也会安装kubectl。通过cloudcli和kubectl可以访问和控制bluemix（IBM Cloud）和 ICP的 K8s cluster.  
安装步骤参考： https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/manage_cluster/install_cli.html

### 安装 helm
要创建，打包，安装Helm Chart需要安装Helm客户端程序。  
安装步骤参考：https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/app_center/create_helm_cli.html

### 让cloudcli，docker和helm能访问ICP Master
修改client host并copy CA文件使得client安装的cloudcli，kubectl， docker 和helm 能访问 K8s master。  
  
参考：   
 
 - https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/app_center/add_int_helm_repo_to_cli.html  
 - https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/manage_images/configuring_docker_cli.html  
 

## 创建和部署Helm Chart （in ICP）

### 创建Helm Chart
 - Helm 命令：  
      
   `helm create sampleApp`.  
   
   参考： https://docs.helm.sh/helm/
 - 从Sample拷贝
 - 按照目录结构分别创建文件。
 
我们以Project “ibm-nodejs-sample” 为例，下面的演示都是以这个为例子。  
例子地址： https://github.com/IBM/charts/tree/master/stable/ibm-nodejs-sample   
### 构建 Helm Chart
 - 用lint命令做静态检查：  
`helm lint ibm-nodejs-sample`  
 - 打包：   
`helm package ibm-nodejs-sample`  
 - 那么一个Helm Chart就会在当前目录构建出来：  
`ibm-nodejs-sample-1.2.1.tgz`. 

### 上传Helm Chart
打包的Helm Chart需要上传到一个Helm Chart Repositiry，任何的HTTP服务器只要提供了GET方法都可作为Helm Chart Repositiry，ICP可以通过 Helm -> Repositories 来添加或者删除Cluester里面的Helm Repo， 如下图：  
![Manage Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/manage-helm-repos.png).  
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/helm-repos.png)

我们用cloudcli把Chart上传到ICP本地的Chart repo， 命令如下：  

 - 登陆到ICP Master			    
`cloudctl login -a https://172.16.26.215:8443 --skip-ssl-validation`  
  
 - 上传Chart.  	  		
`cloudctl catalog load-chart --archive ibm-nodejs-sample-1.2.1.tgz`


### 安装Helm Chart
在K8s里面一般用命令行安装Chart： `helm install`。  
我们看看在ICP里怎么通过UI来安装，打开： Helm -> Repositories -> Catalog， 如下图：   
![app nodejs in catalog](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/helm-catalog-nodejs.png).   

![nodejs configure](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-helm-configure.png)

![nodejs install](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-helm-install.png)

![manage workload release](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/workload-helm-release.png)

![check in release](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-release.png)

### 上传Image
我们用	`kubectl get po` 命令查看新创建的pod会发现有错误，原因是ICP的worker node 不能访问image registry (docker hub),所以要把docker hub上的image上传到ICP的local image registry上， 后面会介绍另外一种方法： PPA

前面的准备工作已经保证了client端的docker可以访问ICP的docker server。同时在/etc/hosts map了ICP master IP 到domain： mycluster.icp   
 
下面是pull image， tag， push的命令：  
 
 - login icp docker server.  
 `docker login mycluster.icp:8500`
 - pull image from docker hub to local.  
 `docker pull ibmcom/icp-nodejs-sample:8`
 - pull the s390 image.   
 `docker pull ibmcom/icp-nodejs-sample-s390x:8`
 - tag the image.  
 `docker tag ibmcom/icp-nodejs-sample:8 mycluster.icp:8500/ibmcom/icp-nodejs-sample:8`
 - tag the s390x image.   
 `docker tag ibmcom/icp-nodejs-sample-s390x:8 mycluster.icp:8500/ibmcom/icp-nodejs-sample-s390x:8`
 - push the image from local to ICP docker registry.  
 `docker push mycluster.icp:8500/ibmcom/icp-nodejs-sample:8`
 - push the s390x image.  
 `docker push mycluster.icp:8500/ibmcom/icp-nodejs-sample-s390x:8`

				
	
我们到ICP的管理界面看看上传的image：  			
![images nodejs](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/images-nodejs.png)				
				
				
				

### 修改image repo
这时候images都上传到ICP了，再来看`kubectl get po` pod还是有错误，因为image的URL发生了变化，我们再来改一下`Deployment`的`image url`.  
执行下面的命令：   
`kubectl edit deploy ibm-nodejs-sample-nodejssample-nodejs -n default`
替换下面的响应的`image url`:  
`mycluster.icp:8500/ibmcom/icp-nodejs-sample:8`.  
同时修改下面的几个变量：  
`- name: CLUSTER_CA_DOMAIN    
   value: mycluster.icp`    
`imagePullPolicy: IfNotPresent `


### 看看结果
再看pod的状态，结果就应该是这样的了：  

```
huoqifengdembp:document huoqifeng$ kubectl get po  
NAME                                                     READY   STATUS    RESTARTS   AGE  
ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-274l8   1/1     Running   0          7h
```
再到ICP的管理界面：  
![images nodejs](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/release-to-launch.png).  
点击 `launch` 就打开了这个sample app.  
![images nodejs](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-launch.png)	

我们可以看到，这里IP地址是ICP Master的IP，port是前面创建的service的port。 
### 为Pod选择worker节点
默认的image是用的amd64，我们部署的cluster实际上包含两种worker node，x86节点和基于LAPR的s390x节点， 如下：  

```
huoqifengdembp:document huoqifeng$ kubectl get nodes --show-labels
NAME            STATUS   ROLES                          AGE   VERSION          LABELS
172.16.26.215   Ready    etcd,management,master,proxy   16d   v1.11.1+icp-ee   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,etcd=true,kubernetes.io/hostname=172.16.26.215,management=true,master=true,node-role.kubernetes.io/etcd=true,node-role.kubernetes.io/management=true,node-role.kubernetes.io/master=true,node-role.kubernetes.io/proxy=true,proxy=true,role=master
172.16.26.216   Ready    worker                         16d   v1.11.1+icp-ee   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.26.216,node-role.kubernetes.io/worker=true
172.16.32.185   Ready    worker                         15d   v1.11.1+icp-ee   beta.kubernetes.io/arch=s390x,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.32.185,node-role.kubernetes.io/worker=true

```

我们看到节点`172.16.26.216`的标签包含`beta.kubernetes.io/arch=amd64`, 节点`172.16.32.185`的标签包含`beta.kubernetes.io/arch=s390x`,这是因为216节点是x86架构，185节点是s390架构，他们在加入K8s cluster的时候会被自动加上响应的arch标签。  

现在我们就用这个标签，通过修改deployment来在创建pod的时候选择worker node。  
通过下面的命令编辑Deployment：  
```
kubectl edit deploy ibm-nodejs-sample-nodejssample-nodejs -n default
```   
增加下面的代码：  

```
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - s390x
```   
同时要修改相应的image：  

```
mycluster.icp:8500/ibmcom/icp-nodejs-sample-s390x:8
```


我们再来看，pod已经重新建立并且被assigne到了s390的节点，我们可以用kubectl来查一下：   

```
huoqifengdembp:document huoqifeng$ kubectl describe node 172.16.32.185
Name:               172.16.32.185
Roles:              worker
Labels:             beta.kubernetes.io/arch=s390x
                    beta.kubernetes.io/os=linux
                    kubernetes.io/hostname=172.16.32.185
                    node-role.kubernetes.io/worker=true
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 05 Dec 2018 12:53:28 +0800
Taints:             <none>
Unschedulable:      false
  Namespace                  Name                                                      CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                                                      ------------  ----------  ---------------  -------------  ---
  default                    ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-274l8    100m (1%)     100m (1%)   128Mi (0%)       128Mi (0%)     8h
  kube-system                audit-logging-fluentd-ds-qf8g7                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                calico-node-49lfl                                         300m (3%)     0 (0%)      150Mi (0%)       0 (0%)         15d
  kube-system                k8s-proxy-172.16.32.185                                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                logging-elk-filebeat-ds-nv5sm                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                metering-reader-l4dk4                                     250m (2%)     0 (0%)      512Mi (1%)       0 (0%)         15d
  kube-system                monitoring-prometheus-nodeexporter-48bpk                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                nvidia-device-plugin-plrxg                                150m (1%)     0 (0%)      0 (0%)           0 (0%)         15d

```


当然，你也可以自己添加或者删除标签来实现，比如：  

```
kubectl label nodes 172.16.32.185 arch=lpar
kubectl label nodes 172.16.26.216 arch=x86
				
kubectl label nodes 172.16.32.185 arch-
kubectl label nodes 172.16.26.216 arch-
```

更多的选择选项请参考下面的文档：  

参考：  

 - https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/
 - https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

 

### Scale pods

`kubectl scale deploy ibm-nodejs-sample-nodejssample-nodejs --replicas=2 -n default`  

```
huoqifengdembp:document huoqifeng$ kubectl get po
NAME                                                     READY   STATUS    RESTARTS   AGE
ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-274l8   1/1     Running   0          8h
ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-qwk9v   1/1     Running   0          1d

```
### 为ICP制作ppa
为什么需要PPA
### Helm Chart & PPA 开发流程






