## kubecon2019-shanghai


为了充分准备周一跟客户的交流，周日下午坐东航的空客350到达上海，因为没拿太多行李，所以就选择坐地铁去酒店，地铁就在机场的地下一层，很方便，到银星皇冠假日大概30分钟左右，到酒店感觉比往常热闹很多，仔细看才知道原来这里是22届上海国际电影节的指定酒店，大堂里和楼梯里的确碰到一些人ms影视演员，也许不是很大牌，也许是本人不太八卦，总之，没有认识的 :-)

![Hotel](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/hotel.jpg)

入住以后，准备晚上再看看ppt，就先到附近的一大会址溜达了一下。会址叫里德村。周围的建筑还是保持着租界原来的模样，走进去看却是一个个的饭馆，都挺有特色，背面则是很集中的一片饮品大排档，风格跟北京还是有点不同。另外在这里还碰到几个党建的旅游团，都穿着统一的制服，有点意思。
![Party-1](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/party-1.jpg)

周一去CMB拜访客户，数据中心在浦东新区，离公司在张江的office不太远，不过还是有点远，还没有地铁。跟客户在数据中心的交流细节就略过了。。。


借着在上海拜访客户的机会，参加了KubeCon 2019。这是我第一次参加KubeCon的会议，对我来说即熟悉又新奇。熟悉的是跟往届Youtube上的视频形式差不多，新奇的是现场的感觉真的不一样！

大会选在前几年比较火的世博中心，一天的会下来，晚上出来溜达一下，欣赏一下夜景也不错的。。。
![bench](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/bench.jpg)
![Bridge](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/bridge.jpg)
![Building](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/building.jpg)

登记的比较早，人还不太多
![Register](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/register.jpg)
![Hall](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/hall.jpg)


会场茶歇的时候提供的一些小食品有点差强人意，不过也不要要求太多了，毕竟咱们是来参会听报告的 :-) 何况午餐和茶歇的时候还有见缝插针的demo theater。。。
![Food-1](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/food-1.jpg)
![Food-2](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/food-2.jpg)
![Theater](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/demo-theater.jpg)
![sponsor](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/sponsor.jpg)

好了，言归正传，说说这两天的见闻和感受吧。。。


第一个就是公司很重视，看到了很多熟悉的小伙伴，早餐的时候竟然还偶遇东哥，遗憾的是没有留个影，好在部分小伙伴们散场的时候照了一张。。。
![IBM](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/ibm.jpg)

第二个感觉是同声传译的很棒，中文翻英文和英文翻中文都很流利，跟speaker同步也还可以，不过你要是正好坐在这个小格子旁边还是会有影响，因为同传的小格子隔音太差 :-)

第三个是大神就是大神，当Linux和Git创世人Linus大叔出现在现场的时候，会场达到了高潮。有几个小伙伴专程跑到主席台下面拍近照，呵呵。Linus大叔还是很朴素的程序员，全程的访谈都是和实在的感觉。在被问到什么是安全最重要的环节的时候，也并没有把自己专注的kernel放在第一位，而是认为在系统的每一个环节都可能对安全造成重要的影响，所以对安全来说同等重要，此言不虚。。。
![Linus](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/linus.jpg)


看来是Linux的影响力确实是越来越大。。。
![Linux](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/linux.jpg)

K8s的大规模部署已经不是问题，国内最大的Ali已经可以在单集群部署10000个节点1000000个pod， Kata已经在生产环境开始全面铺开了，蚂蚁金服，华为，百度。。。ms gVisor也已经在路上。。。
![Large](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/ali-large.jpg)
![kata](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/kata.jpg)


安全方面，Intel 的SGX 推出了LibOS，通过重写glibc，让已经编译过的binary可以直接应用SGX的enclave而不需要改代码或者重新编译，这也许会是x86架构上一个在安全方面比较重量级的项目，但是局限性也很明显，第一是当系统调用比较多的时候性能估计会有不小的影响。第二是还不能很好的处理go程序。IBM也又一个关于secure image的session是一个research team做的，ms跟我们的ssc有些重叠的思路。。
![SGX](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/sgx-libos.jpg)
![Secure](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/secure-image.jpg)


今年Redhat带来一个eBPF的session，意图通过用户层的程序提高网络的性能，目标是达到DPDK的性能而又保留应用的灵活性，感兴趣的同学可以查一下原文。。。
![eBPF](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/eBPF-2.jpg)

IBM的session我还看了几个，一个是istio的最佳实践，一个是Hyperledge Cello，还有一个是KubeFlow的介绍，算是一些入门的session，将来投稿也许可以从这些入手:-), 还有一个Salesforce的关于Housekeeping的也是一个很好的入门参考。。。
![Istio](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/istio-ibm.jpg)
![Kubeflow](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/kubeflow.jpg)
![Eviction](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/eviction.jpg)


KubeEdge 是今年一个比较热门的话题，也许跟5G的到来有关。。。
![Edge](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/edge-arch.jpg)

看Trend，多云和异构必将是未来的方向，加油吧，LinuxONE的小伙伴。。。


相逢总是太短，两天的日程实际上只有一天半，周三中午conf就早早结束了，也许是组委会鼓励大家就近转转？也好，出来世博中心，几百米远就是世博的中国文化馆，俗称中国尊，就在地铁口。。。
![China](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/china.jpg)

中国尊出来，看看还有时间，就又来到外滩匆匆一瞥。。。
![Waitan](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/waitan-1.jpg)
![Waitan](https://github.com/huoqifeng/document/blob/master/kubecon2019/images/waitan-2.jpg)

再见上海！再见 Kubeconf 2019！