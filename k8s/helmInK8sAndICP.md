# Helm Chart in Kubernetes and ICP (IBM Cloud Private)

本文基于 IBM Cloud Private（ICP）3.1.0 和ICP自带的Kubernetes 1.11.1
参考： https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/getting_started/whats_new.html

## K8s预备知识
### K8s架构
### K8s里面的一些术语
Deployment. 
Service. 
ReplicaSet. 
Pod. 
Container. 
### Deployment如何管理Pods

### Service如何导流
### Pod的生命周期以及Liveness和Readiness
### 

## HelmChart 基础
### 从一个例子开始
 - .tpl and .yaml
 -  

 
### 与标准Helm Chart 比，ICP里多了什么？
 - values-metadata.yanl
 - ibm_cloud_ppa/maniefest.yanl
 
 
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
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/manage-helm-repos.png).  

我们用cloudcli把Chart上传到ICP本地的Chart repo， 命令如下：  

 - 登陆到ICP Master			    
`cloudctl login -a https://172.16.26.215:8443 --skip-ssl-validation`  
  
 - 上传Chart.  	  		
`cloudctl catalog load-chart --archive ibm-nodejs-sample-1.2.1.tgz`


### 安装Helm Chart
在K8s里面一般用命令行安装Chart： `helm install`。  
我们看看在ICP里怎么通过UI来安装，打开： Helm -> Repositories -> Catalog， 如下图：   
[!PIC -- TODO].   


### 上传Image
### 修改image repo
### 为Pod选择worker节点
### Scale pods
### 为ICP制作ppa
为什么需要PPA
### Helm Chart & PPA 开发流程






