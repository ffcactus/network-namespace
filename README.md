# Overview

这个项目利用Linux的network namespace功能实现在一台物理机上模拟两个硬件。具体来说，我们假设：

- 有2个服务器，他们有独立的IP地址。
- 有1个bridge，这2个服务器通过这个bridge与外界的网络通讯。

我们需要创建以下虚拟部件来实现上面的效果：
- 2个network namespace，用来模拟2个服务器各自所在的网络，它们之间是隔离的。
- 2个virtual ethernet interface (veth) pair，可以认为一个veth pair是2张网卡和连接2张网卡的物理线。
- 1个virtual bridge，来模拟bridge。
  
## 准备工作
我们一共会用到3个IP地址，分别用于2个veth和一个bridge。这3个IP地址的网段要相同，但是不要和主机IP在同一个网段上，比如有：
- host: 192.168.206.130
- veth1: 192.168.106.101
- veth2: 192.168.106.102
- bridge: 192.168.106.103
同时假设host上能与外界通讯的eth是ens33

## 创建network namespace
创建我们需要的2个network namespace, 假设它们分别叫simulator1, simulator2。
```code
ip netns add simulator1
ip netns add simulator2
ip netns list
```
我们可以在network namespace里执行各种命令
```code
ip netns exec <netns_name> <command_to_execute>
```
比如
```code
[root@localhost baibin]# ip netns exec simulator1 ip address show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
可见新建的network namespace里的网络只有lookback。

可以使用下面的命令删除network namespace:
```code
ip netns delete <netns_name>
```

## 创建veth
创建我们需要的2个veth, 假设它们分别叫veth1, veth2。并把veth1在bridge那一端叫做br-veth1,veth2在bridge那一端叫做br-veth2。
```code
# Create the two pairs.
ip link add veth1 type veth peer name br-veth1
ip link add veth2 type veth peer name br-veth2
```
把这2个veth分别分配给network namespace.
```code
# Associate the non `br-` side
# with the corresponding namespace
ip link set veth1 netns simulator1
ip link set veth2 netns simulator2
```
此时，我们再看network namespace里面的网络设备时就会发现有所不同了，新加的veth会出现再其中。
```code
[root@localhost baibin]# ip netns exec simulator1 ip address show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
29: veth1@if28: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 8a:fb:cb:6c:36:d8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
可以看到虽然有了veth但是还没有IP地址，可以通过ip addr add命令添加IP地址。
```code
[root@localhost baibin]# ip netns exec simulator1 ip addr add 192.168.106.101/24 dev veth1

[root@localhost baibin]# ip netns exec simulator1 ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
29: veth1@if28: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 8a:fb:cb:6c:36:d8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.106.101/24 scope global veth1
       valid_lft forever preferred_lft forever

[root@localhost baibin]# ip netns exec simulator2 ip addr add 192.168.106.102/24 dev veth2

[root@localhost baibin]# ip netns exec simulator2 ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
31: veth2@if30: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 2e:d2:43:89:b0:f6 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.106.102/24 scope global veth2
       valid_lft forever preferred_lft forever
```
可以使用下面的命令删除veth：
```code
ip link delete <veth_name>
```

## 创建bridge
创建连接2个network namespace的bridge，假设名叫br1。
```
ip link add name br1 type bridge
```
建好以后，可以把在br1这一测的veth端打开
```code
ip link set br-veth1 up
ip link set br-veth2 up
```
然后把在namespace一侧的veth端也打开
```code
ip netns exec simulator1 ip link set veth1 up
ip netns exec simulator2 ip link set veth2 up
```
接着通过设置veth的master device，将其与bridge连接。
```code
ip link set br-veth1 master br1
ip link set br-veth2 master br1
```
我们可以通过下面的命令来查看bridge是那些interface的master device。
```code
[root@localhost baibin]# bridge link show br1
28: br-veth1 state UP @(null): <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br1 state disabled priority 32 cost 2
30: br-veth2 state UP @(null): <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br1 state disabled priority 32 cost 2
```
接下来给我们的bridge添加一个IP，这个IP会被记录到host的route table，当与2个veth通讯时可以通过它来路由。
```code
ip addr add 192.168.106.103/24 brd + dev br1
ip link set br1 up
```
我们可以查看以下route
```code
[root@localhost baibin]# ip route list
192.168.106.0/24 dev br1 proto kernel scope link src 192.168.106.103
```
也可以发现能ping通2个veth了。
```code
[root@localhost baibin]# ping 192.168.106.101
PING 192.168.106.101 (192.168.106.101) 56(84) bytes of data.
64 bytes from 192.168.106.101: icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from 192.168.106.101: icmp_seq=2 ttl=64 time=0.040 ms
64 bytes from 192.168.106.101: icmp_seq=3 ttl=64 time=0.043 ms
```
虽然host能ping通veth，但是从network namespace里却没法ping通host,这是因为network namespace里没有默认的route table，需要添加。让它们都通过bridge的IP。
```code
[root@localhost baibin]# ip -all netns exec ip route add default via 192.168.106.103

netns: simulator2

netns: simulator1

```
在这之后，在network namespace之内也可以ping通主机192.168.106.133了。
```code
[root@localhost baibin]# ip netns exec simulator1 ip route
default via 192.168.106.103 dev veth1
192.168.106.0/24 dev veth1 proto kernel scope link src 192.168.106.101

[root@localhost baibin]# ip netns exec simulator1 ping 192.168.206.133
PING 192.168.106.133 (192.168.106.133) 56(84) bytes of data.
64 bytes from 192.168.106.133: icmp_seq=1 ttl=64 time=0.044 ms
64 bytes from 192.168.106.133: icmp_seq=2 ttl=64 time=0.044 ms
64 bytes from 192.168.106.133: icmp_seq=3 ttl=64 time=0.041 ms
```
虽然可以在network namespace内ping通主机，但是没法ping通外界。这是因为返回的包没法route到bridge。我们利用NAT (network address translation)来解决这个问题。
```code
# Enable IP-forwarding.
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables --policy FORWARD ACCEPT

# Flush forward rules.
iptables -P FORWARD DROP
iptables -F FORWARD
 
# Flush nat rules.
iptables -t nat -F

iptables -t nat -A POSTROUTING -s 192.168.106.0/24 -o ens33 -j MASQUERADE

iptables -A FORWARD -i ens33 -o veth1 -j ACCEPT
iptables -A FORWARD -o ens33 -i veth1 -j ACCEPT
iptables -A FORWARD -i ens33 -o veth2 -j ACCEPT
iptables -A FORWARD -o ens33 -i veth2 -j ACCEPT
```




