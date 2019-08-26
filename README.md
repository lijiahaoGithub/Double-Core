# Double-Core
模拟企业网络拓扑 双核心 单核心 高可靠性相关配置 ppp链路
![image](https://github.com/lijiahaoGithub/Double-Core/blob/master/images/%E5%9B%BE%E7%89%871.png)

这是我模拟设计的公司拓扑图，左边是我们的总公司，因为总公司终端多业务多，采用了双核心结构，右边终端数量少业务，采用了单核心结构。

首先看终端，有不同的部门，不同的业务，所以采用了vlan划分技术，可以把相同业务划分的一个vlan，这样的好处有3个，1.隔离广播域，避免没有必要的广播包。2.隔离业务故障，比如arp攻击，dhcp私设。3.布线灵活。  vlan对应的就是相应的access口，因为access口只让相应标签的流量通过，接入层上面通常只有一条链路，所以设置为trunk口，它能让多种业务流量通过。接入层交换机可以选用CISCO WS-C2960-24TC-L，传输速率：10/100Mbps 端口数量：26个 ，价格在4千元左右。

分公司使用一个三层核心交换机连接各个接入层的交换机，因为它的转发速率快，且能实现各个vlan之间的相互通信。可以选用Cisco 3750X-24T-L ，24口全千兆。每个终端都要有地址，而手工配置工作量大，且容易出错。所以我们采用dhcp协议，对pc动态分配地址。因为dhcp服务器在总部，不属于同一个网段，所以在三层核心交换机配置了一个dhcp中继。分公司与总公司通信，数据走互联网的话不安全，所以我们采用了广域网专线，使用的是ppp协议。

双核心实现了高可靠性，两台核心交换机之间使用了链路聚合技术。并且这两台都充当了网关设备，运用的是VRRP技术。在使用高可靠性的同时，它会带来一个问题，那就是环路。我们为了解决问题使用了一个收敛速度快，又能负载均衡的生成树协议——mstp。

因为公网ip数量少，价格高。所以我们采用了pat技术，大量pc使用少量公网ip上网。我们配置了acl，可以对相应的ip做限定。

配置：

LSW5：

#
vlan batch 30 40
#

#
stp region-configuration

 region-name text
 
 instance 1 vlan 30
 
 instance 2 vlan 40
 
 active region-configuration
 
#
#
interface Ethernet0/0/1

 port link-type access
 
 port default vlan 30
 
#
interface Ethernet0/0/2

 port link-type access
 
 port default vlan 40
 
#
interface Ethernet0/0/3

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#
interface Ethernet0/0/4

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#
LSW6：

#
vlan batch 10 20

#
#
interface Ethernet0/0/1

 port link-type access
 
 port default vlan 10
 
#
interface Ethernet0/0/2

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#
interface Ethernet0/0/3

 port link-type access
 
 port default vlan 20
 
#
三层设备：

LSW4：
#

vlan batch 10 20 50

#
#
interface Vlanif10

 ip address 192.168.10.254 255.255.255.0
 
 dhcp select relay
 
 dhcp relay server-ip 180.76.76.76
 
#
interface Vlanif20

 ip address 192.168.20.254 255.255.255.0
 
 dhcp select relay
 
 dhcp relay server-ip 180.76.76.76
 
#
interface Vlanif50

 ip address 192.168.50.2 255.255.255.0
 
#
#
interface GigabitEthernet0/0/1

 port link-type access
 
 port default vlan 50
 
#
interface GigabitEthernet0/0/2

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#
#
ospf 1 router-id 5.5.5.5

 area 0.0.0.1
 
  network 192.168.50.0 0.0.0.255
  
  network 192.168.10.0 0.0.0.255
  
  network 192.168.20.0 0.0.0.255
  
#
ip route-static 0.0.0.0 0.0.0.0 192.168.50.1

#
LSW1：

#

vlan batch 30 40 60 100 200

#
stp instance 1 root primary

stp instance 2 root secondary

dhcp enable

#
stp region-configuration

 region-name text
 
 instance 1 vlan 30
 
 instance 2 vlan 40
 
 active region-configuration
 
#
interface Vlanif30

 ip address 192.168.30.252 255.255.255.0
 
 vrrp vrid 30 virtual-ip 192.168.30.254
 
 vrrp vrid 30 priority 150
 
 dhcp select relay
 
 dhcp relay server-ip 180.76.76.76
 
#
interface Vlanif40

 ip address 192.168.40.252 255.255.255.0
 
 vrrp vrid 40 virtual-ip 192.168.40.254
 
 dhcp select relay
 
 dhcp relay server-ip 180.76.76.76
 
#
interface Vlanif60

 ip address 192.168.60.1 255.255.255.0
 
#
interface Vlanif100

 ip address 180.76.76.254 255.255.255.0
 
#
interface Vlanif200

 ip address 192.168.200.254 255.255.255.0
 
#
interface Eth-Trunk1

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#
interface GigabitEthernet0/0/1

 port link-type access
 
 port default vlan 100
 
#
interface GigabitEthernet0/0/2

 port link-type access
 
 port default vlan 200
 
#
interface GigabitEthernet0/0/3

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#

interface GigabitEthernet0/0/4

#

interface GigabitEthernet0/0/5

 eth-trunk 1
 
#

interface GigabitEthernet0/0/6

 eth-trunk 1
 
#

interface GigabitEthernet0/0/7

 port link-type access
 
 port default vlan 60
 
#

ospf 1 router-id 8.8.8.8

 area 0.0.0.0
 
  network 180.76.76.0 0.0.0.255
  
  network 192.168.60.0 0.0.0.255
  
  network 192.168.30.0 0.0.0.255
  
  network 192.168.40.0 0.0.0.255
  
  network 192.168.200.0 0.0.0.255
  
#

ip route-static 0.0.0.0 0.0.0.0 192.168.60.2

#

LSW2：

#
vlan batch 30 40 70

#

stp instance 1 root secondary

stp instance 2 root primary

#

#
stp region-configuration

 region-name text
 
 instance 1 vlan 30
 
 instance 2 vlan 40
 
 active region-configuration
 
#
#
interface Vlanif30

 ip address 192.168.30.253 255.255.255.0
 
 vrrp vrid 30 virtual-ip 192.168.30.254
 
 dhcp select relay
 
 dhcp relay server-ip 180.76.76.76
 
#
interface Vlanif40

 ip address 192.168.40.253 255.255.255.0
 
 vrrp vrid 40 virtual-ip 192.168.40.254
 
 vrrp vrid 40 priority 150
 
 dhcp select relay
 
 dhcp relay server-ip 180.76.76.76
 
#
interface Vlanif70

 ip address 192.168.70.1 255.255.255.0
 
#

interface MEth0/0/1
#

interface Eth-Trunk1

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#

interface GigabitEthernet0/0/1
#

interface GigabitEthernet0/0/2
#

interface GigabitEthernet0/0/3
#

interface GigabitEthernet0/0/4

 port link-type trunk
 
 port trunk allow-pass vlan 2 to 4094
 
#
interface GigabitEthernet0/0/5

 eth-trunk 1
 
#

interface GigabitEthernet0/0/6

 eth-trunk 1
#

interface GigabitEthernet0/0/7

 port link-type access
 
 port default vlan 70
 
#
#
ospf 1 router-id 5.5.5.5

 area 0.0.0.0
 
  network 192.168.70.0 0.0.0.255
  
  network 192.168.30.0 0.0.0.255
  
  network 192.168.40.0 0.0.0.255
  
#
ip route-static 0.0.0.0 0.0.0.0 192.168.70.2

#
LSW3：模拟dhcp服务器

vlan batch 200

#
ip pool vlan10

 gateway-list 192.168.10.254
 
 network 192.168.10.0 mask 255.255.255.0
 
 dns-list 192.168.200.1
 
#
ip pool vlan20

 gateway-list 192.168.20.254
 
 network 192.168.20.0 mask 255.255.255.0
 
 dns-list 192.168.200.1
 
#
ip pool vlan30

 gateway-list 192.168.30.254
 
 network 192.168.30.0 mask 255.255.255.0
 
 dns-list 192.168.200.1
 
#
ip pool vlan40

 gateway-list 192.168.40.254
 
 network 192.168.40.0 mask 255.255.255.0
 
 dns-list 192.168.200.1
 
#
#

interface Vlanif200

 ip address 180.76.76.76 255.255.255.0
 
 dhcp select global
 
#
interface MEth0/0/1

#

interface GigabitEthernet0/0/1

 port link-type access
 
 port default vlan 200
 
#
ip route-static 0.0.0.0 0.0.0.0 180.76.76.254

Server1：充当本地DNS服务器和公司web服务器

![image](https://github.com/lijiahaoGithub/Double-Core/blob/master/images/%E5%9B%BE%E7%89%872.png)
![image](https://github.com/lijiahaoGithub/Double-Core/blob/master/images/%E5%9B%BE%E7%89%873.png)

路由器：

AR1：

#

interface Serial4/0/0

 link-protocol ppp
 
 ip address 192.168.80.2 255.255.255.0 
 
#

interface Serial4/0/1

 link-protocol ppp
 
#
#
interface GigabitEthernet0/0/1

 ip address 192.168.50.1 255.255.255.0 
 
#
#
ospf 1 router-id 1.1.1.1 

 area 0.0.0.1 
 
  network 192.168.50.0 0.0.0.255 
  
  network 192.168.80.0 0.0.0.255 
  
#
ip route-static 0.0.0.0 0.0.0.0 192.168.80.1

#
AR2：

#
acl number 2000  

 rule 5 permit source 192.168.10.0 0.0.0.255 
 
 rule 10 permit source 192.168.20.0 0.0.0.255 
 
 rule 15 permit source 192.168.30.0 0.0.0.255 
 
#
#

 nat address-group 1 64.1.1.3 64.1.1.5
 
#
interface Serial4/0/0

 link-protocol ppp
 
 ip address 192.168.80.1 255.255.255.0 
 
#
interface Serial4/0/1

 link-protocol ppp
 
#
interface GigabitEthernet0/0/0

 ip address 192.168.60.2 255.255.255.0 
 
#

interface GigabitEthernet0/0/1

 ip address 192.168.70.2 255.255.255.0 
 
#
interface GigabitEthernet0/0/2

 ip address 64.1.1.1 255.255.255.248 
 
 nat outbound 2000 address-group 1 
 
 nat static global 64.1.1.2 inside 192.168.200.1
 
#
interface NULL0

#
interface LoopBack0

 ip address 1.1.1.1 255.255.255.0 
 
#

ospf 1 router-id 2.2.2.2 

 area 0.0.0.0 
 
  network 1.1.1.0 0.0.0.255 
  
  network 64.1.1.0 0.0.0.7 
  
  network 192.168.60.0 0.0.0.255 
  
  network 192.168.70.0 0.0.0.255 
  
 area 0.0.0.1 
 
  network 192.168.80.0 0.0.0.255 
  
#
ip route-static 0.0.0.0 0.0.0.0 64.1.1.6

#
AR3：模拟外网isp，实际应用中不用配置，只需向运营商购买公网ip即可
#

interface GigabitEthernet0/0/0

 ip address 64.1.1.6 255.255.255.0 
#

interface GigabitEthernet0/0/1

 ip address 8.8.8.254 255.255.255.0 
#



数据流分析：

DHCP:动态主机设置协议

我们公司的终端很多，手动分配地址会非常的麻烦，而且容易出错，所以我们用到DHCP动态主机设置协议，DHCP服务器会自动分配地址给终端。

发现阶段：主机发送一个DHCP Discover广播包，其中：

三层封装：

源端口是68，目标端口是67

源ip是0.0.0.0，目标ip是255.255.255.255

二层封装：

源mac是本机的mac地址，目标mac地址是全f


二层交换机收到后，拆开二层，首先会学习，将源mac地址与接口绑定，查看mac表做相应转发，而这里的mac地址的全f，所以它会广播这个包。

三层交换机收到，他也具备二层功能，先学习，发现目标mac是全f，就会在它的vlan里广播。对于vlan的网关收到先看目标mac，全f，拆开。查看目标ip是广播地址，可以拆开，发现端口是68，目标端口是67，，是一条去往dhcp服务器的包，因为其配置了dhcp中继，所以它会重新封装这个包。其中：

三层封装：

源ip是三层交换机的ip，目标ip是配置中继的ip地址即dhcp服务器地址

源端口是67，目标端口是67

二层封装：

		源mac是三层交换机出端口的mac，目的mac是对面路由器接口的mac地址
数据部分会加入网关的ip。


路由器收到这个包，拆开二层目的mac是我的，接着拆开三层，发现目标ip不是我，则查看路由表，封装ppp协议向s4/0/0口发出。对面路由器收到拆开ppp头部，查看路由表进行转发。


DHCP服务器回复一个DHCP offer包，其中：

源ip为dhcp服务器的ip，目标ip是最开始三层交换的ip

源端口是67，目标端口是67

回来的路径大致一样，到了LSW4会变成广播包，以广播的形式转发。很多主机会收到，但只有对照自己的mac与数据包中的数据中mac一致才会使用。


DHCP request：当Client收到了DHCP Offer包以后（如果有多个可用的DHCP服务器，那么可能会收到多个DHCP Offer包），确认有可以和它交互的DHCP服务器存在，于是Client发送Request数据包，请求分配IP。 

此时的源IP和目的IP依然是0.0.0.0和255.255.255.255。

接着服务器回复一个DHCP ack



