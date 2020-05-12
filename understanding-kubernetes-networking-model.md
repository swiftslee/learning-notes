#### 宿主机<->容器的网络

##### bridge方案

同一网段内发送数据是不需要经过网关(路由器)的，而不同网段之间发送数据必需经过网关(路由器)

###### 容器的南北流量(访问外网)

假如起了一个container 在宿主机通过ifconfig会看到有一个docker0 虚拟网卡

宿主机通过route命令可以看到所有到172网段的路由都交给了docker0 并且docker0  把veth连了起来

docker0是一个很大的网桥 上面连接了很多组veth

容器内部 通过route命令发现docker0的ip是默认网关(0.0.0.0的destination都交给了docker0的ip) 相当于所有的流量先走docker0 然后再由docker0交给eth0 去访问外网

因此 数据包的流程是从容器的eth0->veth0->docker0网桥->host eth0 最后在iptables这里 会有一次snat

就是把从eth0接口出去的流量中源地址是172.17的做SNAT源地址转换 这样一来，从Docker容器访问外网的流量，在外部看来就是从宿主机上发出的，外部感觉不到Docker容器的存在

同样的 外部想要通过 hostport访问容器的端口 都会经过一次 dnat 把ip地址转换成172.

###### 东西流量( 容器之间)

与南北不同的是 docker0这个网桥直接在一对veth之间转发数据即可 所以这时候docker0充当了二层虚拟交换机的角色

![](https://github.com/yuswift/learning-notes/raw/master/images/docker0.png)

https://mp.weixin.qq.com/s/nDzJQq8nysywicctr7EAhw

https://morningspace.github.io/tech/k8s-net-docker0/

https://www.jianshu.com/p/464774e6e5c5

https://zhonghua.io/2018/11/13/container-network/

##### container/host/none模式比较简单 跳过

#### 容器跨节点访问的解决思路

由于每个pod或者container都有独立的ip 由于是容器自己配置的ip underlay平面的底层网络设备如交换机、路由器等完全不感知这些IP的存在，也就导致容器的IP不能直接路由出去实现跨主机通信 需要解决如下问题

- 所有 Pod 都可以与所有其他 Pod 通信，而无需使用网络地址转换（NAT）。
- 所有节点都可以在没有 NAT 的情况下与所有 Pod 通信。
- Pod 所看到的自己的 IP 与其他 Pod 或节点看到的相同。

1.要么修改底层网络设备 加入容器IP地址的管理 

2.复用underlay平面网络  overlay隧道传输(多了封包拆包的过程 flannel 的vxlan或者weave)和修改主机路由(flannel的host-gw和calico)

##### docker的overlay

![](https://github.com/yuswift/learning-notes/raw/master/images/docker-overlay.jpeg)

一个container两张虚拟网卡 eth0负责东西流量 eth1负责南北 其实eth1就是bridge模式下的docker0

eth0通过网桥连上了vxlan 由于原生的overlay都在一个子网里面 而同一个子网里面则是 通过Mac地址识别对方 传统的方案是通过arp协议获取ip mac的转换 这样会带来广播包很多 成本加大 所以docker把一些endpoint的ip mac存在了etcd里面 再由vxlan0去代理本地容器arp

ARP代理属于L2层问题，而容器的数据包最终还是通过Vxlan隧道传输的 如果只知道目标容器的Mac地址 而不知道目标容器在哪个节点上 还是不行的 创建vxlan隧道时可以指定本地ip（local IP)和对端IP(remote IP)建立点对点通信 fdb表决定了容器与node ip的映射关系

跨节点数据包的流程是eth0->veth0->vxlan0打隧道(通过arp代理和fdb表进行学习)

https://mp.weixin.qq.com/s/-L_2qPpFmc85lMmVUi_UCQ

##### weave

与docker的overlay类似

![](https://github.com/yuswift/learning-notes/raw/master/images/weave.png)

##### flannel的host-gw

直接添加路由，将目的主机当做网关，直接路由原始封包。这种方法要求所有的主机都在一个子网以内 必须是二层可达的(连在一台交换机上) 性能最好

##### flannel的udp

而udp类型backend的基本思想是：既然主机之间是可以相互通信的（并不要求主机在一个子网中），那么我们为什么不能将容器的网络封包作为负载数据在集群的主机之间进行传输呢？这就是所谓的overlay。使用设备flannel.0进行封包解包，不是内核原生支持，上下文切换较大，性能非常差 目前已被弃用x

#### vxlan

传统的交换网络解决了二层的互通及隔离问题，这个架构发展了几十年已经相当成熟。而随着云时代的到来，却渐渐暴露出了无法支持多租户之间隔离的缺点。

只要是三层可达（能够通过 IP 互相通信）的网络就能部署 vxlan

工作在ip层 通过把MAC地址封装到OSI模型的第3层中(mac in udp)，从而让处于不同网络的主机看起来像在同一个局域网，使用着相同的广播域一样，形成一个大局域网环境 vni=24 bit解决了vlan12bit太少的问题

 vxlan 协议比原始报文多 50 字节的内容，这会降低网络链路传输有效数据的比例

一个 vxlan 报文需要确定两个地址信息：目的虚拟机的 MAC 地址和目的 vtep 的 IP 地址

![](https://github.com/yuswift/learning-notes/raw/master/images/vxlan.png)

##### flannel的vxlan

核心:路由信息+arp+fdb

Flannel初始化时通常指定一个16位的网络，然后每个Node单独分配一个独立的24位子网。

flannel只有一张网卡 这一点比较优雅 本地通信直接使用原生的bridge网络

在容器内部通过网桥查看本地路由 可以看到 到不同的子网都有不同的网关 到本地的流量直接交给cni0/docker0去处理即可 因为这里不涉及跨节点的访问  而跨节点的流量由flannel.1网卡去代理

1. `# ip r`
2. `default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.68 metric 100`
3. `40.15.26.0/24 via 40.15.26.0 dev flannel.1 onlink`
4. `40.15.43.0/24 dev docker0 proto kernel scope link src 40.15.43.1`
5. `40.15.56.0/24 via 40.15.56.0 dev flannel.1 onlink`

同样本地路由这里只有ip要想获取 mac(实际上这个mac就是目标宿主机的flannel.1的mac地址)都要从arp表里获取 这里也是像docker的overlay一样采用了自代理的模式 这里搞定了目标ip 要给哪一张虚拟网卡转发的问题 为什么需要Mac地址呢 个人理解是当转发到另一台宿主机以后 宿主机与flannel.1位于同一个局域网 他们是通过Mac地址进行通信的 

获取到Mac地址以后 vxlan负载部分的已经封包完成

 `[目的MAC:node2.flannel.1.MAC, 目的IP:container2.IP, ...]`

然后增加VXLAN Header

 `[VxlanHeader:VNI:1 [目的MAC:node2.flannel.1.MAC, 目的IP:container2.IP, ...]]`

 目标Mac地址对应的宿主机的ip又可以从fdb表去获取 

生成外UDP包: `[目的MAC:node2.MAC, 目的IP:node2.IP [VxlanHeader:VNI:1 [目的MAC:node2.flannel.1.MAC, 目的IP:container2.IP, ...]]]`

这时候外部的包也封好了 这里又搞定了数据包应该发到哪台宿主机上的问题

数据包封装好以后 先经过iptables链 再到达目标机器的eth0 通过拆包根据vni值转发到flannel设备 对比Mac地址相等以后 转发到本地的flannel.1 又进而通过cni0/docker0转发到目标容器的veth

![](https://github.com/yuswift/learning-notes/raw/master/images/flannel-vxlan)

#### calico

Calico和Flannel一样，每个节点分配一个子网，只不过Flannel默认分24位子网，而Calico分的是26位子网。

所有的容器Mac地址都是ee:ee:ee:ee:ee:ee？ calico认为不是所有的内核都能自动分配Mac地址 所以calico自己制定 而Calico完全使用三层路由通信，MAC地址是什么其实无所谓，因此直接都使用 ee:ee:ee:ee:ee:ee。

容器本地路由都走169.254.1.1这个网关？回顾之前的网络模型，大多数都是把容器的网卡通过VETH连接到一个bridge设备上，而这个bridge设备往往也是容器网关，相当于主机上多了一个虚拟网卡配置。Calico认为容器网络不应该影响主机网络，因此容器的网卡的VETH另一端没有经过bridge直接挂在默认的namespace中。而容器配的网关其实也是假的，通过proxy_arp修改MAC地址模拟了网关的行为，所以网关IP是什么也无所谓，那就直接选择了local link的一个ip，这还节省了容器网络的一个IP

按理说同一主机的容器之间会像之前一样 处于同一个子网 而Mac地址一样如何通信呢？仔细看容器配置的IP掩码居然是32位的，那也就是说跟谁都不在一个子网了，也就不存在二层的链路层直接通信了。所以说calico是一个纯三层通信的cni~

宿主机路由同flannel的host-gw差不多 下一跳直接指向hostip 不一样的是到达宿主机后，Flannel会通过路由转发流量到bridge设备中，再由bridge转发给容器，而Calico则为每个容器的IP生成一条明细路由，直接指向容器的网卡对端。因此如果容器数量很多的话，主机路由规则数量也会越来越多，因此才有了路由反射

calico通过iptables+ipset实现多个网络的隔离 而flannel则通过vni

如果要支持跨网段通信(即不在一个子网 因为calico默认都在一个子网的 flannel的host-gw则不支持)则会通过ipip隧道去传输

https://mp.weixin.qq.com/s/-L_2qPpFmc85lMmVUi_UCQ

https://zhonghua.io/2018/11/13/container-network/

##### https://www.cnblogs.com/YaoDD/p/7681811.html

https://xuxinkun.github.io/2019/06/05/flannel-vxlan/

https://cizixs.com/2017/09/25/vxlan-protocol-introduction/

https://juejin.im/post/5da92ea55188253a8f7c495a

#### 为什么需要NAT

个人理解是为了弥补ipv4公网地址不足的问题 多个私有ip在对外访问时 可以通过统一的公网ip对外

NAT的本质就是让一群机器公用同一个IP

**三种NAT技术：**

- **静态NAT：**静态NAT就是一对一映射，内部有多少私有地址需要和外部通信，就要配置多少外网IP地址与其对应
- **动态NAT：**动态NAT是在路由器上配置一个外网IP地址池，当内部有计算机需要和外部通信时，就从地址池里动态的取出一个外网IP，并将他们的对应关系绑定到NAT表中，通信结束后，这个外网IP才被释放，可供其他内部IP地址转换使用，这个DHCP租约IP有相似之处。
- **PAT(port address Translation，端口地址转换，也叫端口地址复用)：**这是最常用的NAT技术，也是IPv4能够维持到今天的最重要的原因之一，它提供了一种多对一的方式，对多个内网IP地址，边界路由可以给他们分配一个外网IP，利用这个外网IP的不同端口和外部进行通信。

https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/

https://vflong.github.io/sre/k8s/2020/02/29/understanding-kubernetes-networking-model.html

#### 同一个节点的pod 之间通信

最开始 有一个根命名空间 里面有一张eth0网卡负责与外部进行沟通

然后 每个pod都有自己的网络ns与eth0 他们怎么与宿主机沟通呢 就是通过veth0

![](https://github.com/yuswift/learning-notes/raw/master/images/pod-to-pod-same-node.gif)



##### 同一个节点一个数据包的流动过程

Veth0 与veth1之间 通过网桥进行沟通 网桥会判断是否能够转发

Linux 以太网桥是一种虚拟的第 2 层网络设备，用于将两个或多个网段结合在一起，透明地将两个网络连接在一起。该网桥通过维护源和目标之间的转发表实现，而转发表检查通过它的数据包的目的地，并确定是否将是否将数据包传递到连接网桥的其他网段。桥接代码通过查看网络中每个以太网设备唯一的 MAC 地址来决定桥接数据还是丢弃数据。

![](https://github.com/yuswift/learning-notes/raw/master/images/pod-to-pod-same-node.gif)

#### 跨节点的pod之间的通信

![](https://github.com/yuswift/learning-notes/raw/master/images/pod-to-pod-different-nodes.gif)

至于如何正确地把pod ip正确路由到正确的节点 这就是由不同的cni插件实现的了 

#### pod to service之间的通信

![](https://github.com/yuswift/learning-notes/raw/master/images/pod-to-service.gif)

##### 这里会有一次DNAT的转换

简单来说就是pod1->svc1 DNAT pod1->pod4 根据 [这篇文章](https://mp.weixin.qq.com/s/7oqyIjCvdqVtPqkFsWaxyw) 这里应该还有一次 SNAT的转换

#### service to pod之间的通信



![](https://github.com/yuswift/learning-notes/raw/master/images/service-to-pod.gif)

##### 这里有一次SNAT的转换

由于为了让pod1以为他一直在跟svc通信 pod4->pod1的响应成了 svc1->pod1 

 (这里有一些自己的看法 假如在之前的那一步 已经做了snat 那么这一次除了有snat 应该还有dnat)

#### pod如何访问外网

![](https://github.com/yuswift/learning-notes/raw/master/images/pod-to-internet.gif)

网桥发现网段不匹配 则会通过宿主机的iptables 这是否会有一次SNAT 因为网关只认识节点的ip  到了网关以后 又会有一次SNAT 把它转换成公网ip

