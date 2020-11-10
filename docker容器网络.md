虚拟化网络
========
在实现容器技术需要隔离的六种资源中提到过，网络名称空间(NET)主要实现网络设备/协议栈的隔离，如果为一个namespaces单独分配一个物理网卡设备，那么其他namespaces就看不见这个物理网卡了，这个namespaces内部与外界的通信也是没有问题的，可以为每一个namespaces单独分配一个物理网卡设备以解决namespaces的网络通信问题，但如果namespaces的数量大于物理网卡设备的数量时如何解决，每一个namespaces内部的网络进程也需要通过网络进行通信又如何解决

虚拟网卡
-------
用纯软件的方式模拟网卡设备来使用，Linux内核支持二层/三层网络设备的模拟

### 虚拟二层网络通信
对于二层网络通信的模拟，利用Linux内核对二层虚拟设备的支持创建虚拟网卡接口，每一个虚拟网卡接口都是成对出现的，可以模拟为一根网线的两头，一头可以模拟插在主机上，另一头可以模拟插在交换机上，相当于将一个主机连接到交换机上。而Linux原生支持模拟二层虚拟网桥设备，那么将虚拟网卡设备的一头分配到namespaces，另一头分配到虚拟网桥，则相当于namespaces连接到网桥上，如果将多个namespaces连接到网桥，且配置为同一网段，那么不同的namespaces之间也能够相互通信了

### 虚拟三层网络通信
二层网络之间的通信通过虚拟网桥和虚拟网卡可以实现，不同网段之间的通信则需要通过三层路由来实现，而Linux内核本身则可以实现路由转发功能(通过iptables或打开内核的路由转发功能)，将内核的路由转发放置在一个单独的namespaces内模拟路由器，并使用虚拟网卡连接路由器和网桥(一个单臂路由模型)即可实现不同网段之间的namespaces通信 <br /> <br />

docker的网络通信中，每创建一个容器，都会创建一个虚拟网卡，一头放在容器内，另一头接在docker0网桥上
```shell
# brctl show    #查看网桥上关联的端口
```
docker0桥默认是一个NAT桥，每创建并启动一个容器时，会自动创建一个iptables规则

docker的四种网络模型
closed container
不需要连接网络
bridge container
桥接网络，通过nat的方式将容器和docker0网桥连接
joined container
联盟网络，将一部分namespaces隔离(User,Mounted,Pid)，网络namespaces则共享同一个
open container
开放网络，直接共享物理机的namespaces

```shell
# docker network inspect bridge    #查看bridge网络详细信息
```

### ip命令操作网络名称空间
示例：创建网络名称空间
```shell
# ip netns add r1		创建两个netns
# ip netns add r2
# ip netns list			查看创建的netns
# ip netns exec r1 ip add	在netns里执行ip命令
```
注：使用ip命令模拟的网络名称空间，处了网络名称空间是隔离的，其他的都是共享的

示例：配置虚拟网卡接口
```shell
# ip link add name veth1.1 type veth peer name veth1.2		创建虚拟网卡接口，指定veth1.1的对端接口名称为veth1.2
# ip link show												查看虚拟网卡接口
# ip link set dev veth1.1 netns r1							将虚拟网卡的veth1.1端放至netns r1中
# ip netns exec r1 ip link set dev veth1.1 name eth0		将netns中的虚拟网卡接口改名为eth0
# ifconfig veth1.2 10.250.1.5/24 up							为物理设备上的veth1.2配置临时地址
# ip netns exec r1 ifconfig eth0 10.250.1.6/24 up			为netns内的veth1.1配置临时地址
# ping 10.250.1.6											测试物理设备与netns通信是否正常
```