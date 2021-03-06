既然要讲Bridge模式， 那么首先要了解网桥是什么,首先看看维基百科是如何定义的吧。

> 橋接器（英语：network bridge），又称网桥，一種網路裝置，負責網路橋接（network bridging）之用。根據MAC分區塊，可隔離碰撞。橋接器将网络的多个网段在数据链路层（OSI模型第2层）连接起来（即桥接）。桥接器在功能上与集线器等其他用于连接网段的设备类似，不过后者工作在物理层（OSI模型第1层）。桥接器仅仅在不同网络之间有数据传输的时候才将数据转发到其他网络，不是像集线器那样对所有数据都进行广播。对于以太网，“桥接”这一术语正式的含义是指符合IEEE 802.1D标准的设备，即“网络切若有通訊頻繁的機器，則應置於同區之內，否則效能將降低。橋接器可以分割網段，不似集線器仍是在為同一碰撞網域，所以對頻寬耗損較大。因橋接器透過其內之MAC表格，讓傳送帧不會通過，所以其稱之為資料鏈結層操作之網路元件。</blockquote>

从上面可以看出来，网桥工作在OSI模型的第二层，用于网络包的转发，和集线器以及交换机的功能相似.既然和交换机的功能相似，那么应该也有对应的MAC地址和端口的映射关系。为了验证这一点，我们来做一个实验。

启动三个docker实例,然后查看默认的docker0网桥的映射表。

```

docker run -t -i --name test1 docker-java

docker run -t -i --name test2 docker-java

docker run -t -i --name test3 docker-java

```

来看看这两个docker实例对应的mac地址是多少,

![http://ww2.sinaimg.cn/large/87f5e2f6jw1fa3b80ex6nj20ih04pq52.jpg](http://ww2.sinaimg.cn/large/87f5e2f6jw1fa3b80ex6nj20ih04pq52.jpg)

![http://ww4.sinaimg.cn/large/87f5e2f6jw1fa3b80tp8sj20ih04omza.jpg](http://ww4.sinaimg.cn/large/87f5e2f6jw1fa3b80tp8sj20ih04omza.jpg)

![http://ww3.sinaimg.cn/large/87f5e2f6jw1fa3b7zwoajj20ie04o40n.jpg](http://ww3.sinaimg.cn/large/87f5e2f6jw1fa3b7zwoajj20ie04o40n.jpg)

运行上面的命令后可以看到如下的结果,

![http://ww1.sinaimg.cn/large/87f5e2f6jw1fa3b7zvd5nj20hv046wfx.jpg](http://ww1.sinaimg.cn/large/87f5e2f6jw1fa3b7zvd5nj20hv046wfx.jpg)

这三个docker实例的mac地址都在网桥的映射表中。但是过一段时间再次运行该命令(期间不要在各个docker实例中做任何操作)，会发现三个mac地址均从上面的表中消失了，这是为什么呢？仔细观察可以发现他们的共同点是都有一个ageting timer在不停的计数，而且随着时间的推移而变大。使用man命令可以看到关于ageing的描述如下，

> brctl setageing &lt;brname&gt; &lt;time&gt; sets the ethernet (MAC) address ageing time, in seconds. After &lt;time&gt; seconds of not having seen a frame coming from a certain address, the bridge will time out (delete) that address from the Forwarding DataBase (fdb).



原来是因为三个docker长时间没有发送报文到网桥，超过设置的最大ageing的时候就将其从映射表(fdb)中删除了，同时我们也可以通过brctl setageing来设置这个值。



那么我们在test1上ping一下test2后,再看会有什么结果呢？

```

ping 172.17.0.6</pre>

```

运行后再次查看映射表,

![http://ww2.sinaimg.cn/large/87f5e2f6jw1fa3b7zlnmxj20hu03lq43.jpg](http://ww2.sinaimg.cn/large/87f5e2f6jw1fa3b7zlnmxj20hu03lq43.jpg)

两个docker实例的mac地址也出现了，这期间发生了什么呢？在下面的章节中会介绍。



除了三个docker实例的mac地址，另外3个mac地址是属于谁的呢？在宿主机上执行如下命令,

```

ifconfig |grep HWaddr|awk '{print $1 "  " $5}'

```

上面的命令主要是找出本机的设备名称以及对应的mac地址。

```

docker0 02:42:0a:b2:f7:44

eth0 b8:88:e3:ee:cc:52

lxcbr0 2a:d5:57:ac:74:fe

veth3065f93 ee:3c:0e:11:75:89

vetha0a4a89 96:76:cd:a7:1a:1b

vethb1afec1 3e:d0:13:7e:f9:8c

wlan0 60:6c:66:00:bf:86

```

其他三个mac地址对应的设备名称正好和三个veth与网桥桥接在一起的设备名称一样。



那么网桥和交换机到底有没有什么不同的地方呢？当然是有的，在宿主机运行ifconfig之后可以发现docker0有ip地址，而我们知道交换机是没有ip地址的~~为什么会发生这种情况呢。。。我们知道网桥是在linux内核中虚拟出来的网络设备，作为通用网络设备有ip地址是很正常的。如果大家有兴趣可以看看这篇文章，



<a href="https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/">Linux 上的基础网络设备详解</a>

> Bridge 和现实世界中的二层交换机有一个区别，图中左侧画出了这种情况：数据被直接发到 Bridge 上，而不是从一个端口接受。这种情况可以看做 Bridge 自己有一个 MAC 可以主动发送报文，或者说 Bridge 自带了一个隐藏端口和寄主 Linux 系统自动连接，Linux 上的程序可以直接从这个端口向 Bridge 上的其他端口发数据。所以当一个 Bridge 拥有一个网络设备时，如 bridge0 加入了 eth0 时，实际上 bridge0 拥有两个有效 MAC 地址，一个是 bridge0 的，一个是 eth0 的，他们之间可以通讯。由此带来一个有意思的事情是，Bridge 可以设置 IP 地址。通常来说 IP 地址是三层协议的内容，不应该出现在二层设备 Bridge 上。但是 Linux 里 Bridge 是通用网络设备抽象的一种，只要是网络设备就能够设定 IP 地址。当一个 bridge0 拥有 IP 后，Linux 便可以通过路由表或者 IP 表规则在三层定位 bridge0，此时相当于 Linux 拥有了另外一个隐藏的虚拟网卡和 Bridge 的隐藏端口相连，这个网卡就是名为 bridge0 的通用网络设备，IP 可以看成是这个网卡的。当有符合此 IP 的数据到达 bridge0 时，内核协议栈认为收到了一包目标为本机的数据，此时应用程序可以通过 Socket 接收到它。一个更好的对比例子是现实世界中的带路由的交换机设备，它也拥有一个隐藏的 MAC 地址，供设备中的三层协议处理程序和管理程序使用。设备里的三层协议处理程序，对应名为 bridge0 的通用网络设备的三层协议处理程序，即寄主 Linux 系统内核协议栈程序。设备里的管理程序，对应 bridge0 寄主 Linux 系统里的应用程序。



在有了网桥的基础后，可以进入正题了，docker实例是如何使用网桥进行通讯的呢?

我们先来看看docker在bridge模式下的网络拓扑图。

![http://ww4.sinaimg.cn/large/87f5e2f6jw1fa3b80yu99j20cl0b2dgg.jpg](http://ww4.sinaimg.cn/large/87f5e2f6jw1fa3b80yu99j20cl0b2dgg.jpg)

在上图中，定义了一个网桥docker0，该网桥的ip地址是172.17.42.1.然后定义了两个veth设备。这里先解释下什么是veth，我们可以将veth理解为一个管道，管道的两端各有一个设备，任何一端接收到数据都会将数据传送到另外一端。上图中，两个veth设备分别将一端与网桥相连。那么这个连接行为怎么理解呢？来看看brctl addif指令的解释，

>The command brctl addif &lt;brname&gt; &lt;ifname&gt; will make the interface &lt;ifname&gt; a port of the bridge &lt;brname&gt;. This means that all frames received on &lt;ifname&gt; will be processed as if destined for the bridge. Also, when sending frames on &lt;brname&gt;, &lt;ifname&gt; will be considered as a potential output interface.



大致意思就是将interface作为网桥的一个端口。当interface接收到消息的时候就仿佛是发送给网桥的，网桥就会代为处理这个消息。回到上面的图，veth在将一端放入到bridge时，同时将另一端作为docker实例的网卡，这样就实现了两个docker实例通过网桥(在这里理解为交换机就可以了)相连。

**在通俗点说明就是veth，充当了网线的作用，而网桥则充当了交换机的作用。**

对网络拓扑图有大致的了解后，我们就可以看看具体是如何通信的了。我们知道在通常情况下docker实例的ip和网桥在同一网段，和外界ip不在同一网段。所以docker实例之间(<strong><em><span style="color: #ff0000;">在这里假设所有实例都在相同的网段</span></em></strong>)的通信和docker实例与外界通信是两种不同的情况.下面会针对这两种情况分开说明。

### 宿主机内的不同docker实例通信

docker1发送消息到其对应的网卡eth0，根据veth的特性这个消息会被传送到veth的另一端设备veth*上，这也相当于数据发送到了网桥上。网桥根据其mac地址查找对应的端口将其发送到对应的veth*上，然后由于veth的特性这个消息最终被发送到了docker2的eth0上。

然而，上文说过了这个映射表中的某些数据是可能因为时间原因而失效的，如果失效了会怎么办呢？网桥可能会将这个消息进行广播,然后只有mac地址正确的docker实例才会进行处理然后将数据返回给网桥，网桥会提取除返回信息的mac地址并且将这个mac地址与端口的映射记录下来，下次消息过来的时候就可以进行单播了。

如果发送方再发送消息时并不知道对方的mac地址(比如第一次交互)怎么办呢？根据在docker1进行实验(在docker1上ping docker2)发现，docker1会发送arp广播让ip为目标ip的docker实例发送消息给docker1。

```

18:18:55.215940 02:42:ac:11:00:0a (oui Unknown) &gt; Broadcast, ethertype ARP (0x0806), length 42: Request who-has 172.17.0.11 tell 172.17.0.10, length 28

18:18:55.215975 02:42:ac:11:00:0b (oui Unknown) &gt; 02:42:ac:11:00:0a (oui Unknown), ethertype ARP (0x0806), length 42: Reply 172.17.0.11 is-at 02:42:ac:11:00:0b (oui Unknown), length 28

```

这样docker1和网桥就都知道了docker2的mac地址，同时docker1会将docker2的mac地址和ip写入到arp表中作为缓存，下次访问的时候访问arp表就可以获得到对应的mac地址了。

### 与宿主机外部的主机通信

如果访问的ip不在同一网段内，比如访问了192.168.1.1，当消息发送到网桥的时候也就是发送到了宿主机的网卡上，这时候网卡会根据本机路由设置将消息转发.但由于网卡收到的原始数据的源ip为docker容器的ip，外界无法识别，所以在发送前要进行一次NAT(网络地址转换)，源ip会被转换为宿主机的ip,并且源端口号也会从本机选择出一个未使用的端口进行替换，这样外界看到消息就好像宿主机发送出去的一样了。

我们来看看宿主机的上配置的NAT规则，

```

sudo iptables -t nat -vL

```

![http://ww4.sinaimg.cn/large/87f5e2f6jw1fa3b7z5mmgj20uz0at0yr.jpg](http://ww4.sinaimg.cn/large/87f5e2f6jw1fa3b7z5mmgj20uz0at0yr.jpg)

## 总结

上面零零碎碎写了不少，不是很系统，全当做自己在学习docker中的笔记。不过也希望可以给大家一些帮助.....有什么不对的地方欢迎大家指出，后续如果发现有什么理解错误的地方也会随时更新~~~

