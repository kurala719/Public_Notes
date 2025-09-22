---
Title: 关于VRF Lite的一些疑难点
Author: Izumi, Akiko
Date: 2025-09-16
---

# 关于VRF Lite的基础配置

## 前言

笔者最近在和同事做实验时用到了VRF技术，然而由于相关资料的语焉不详以及设备的各种版本和型号问题导致实验过程中问题层出不穷，一度花了三小时才完成该实验。中途笔者查阅了各种文档和资料，发现对于VRF技术的探讨似乎总是不够深入，即多数文档只给出配置命令和配置步骤，但完全没有谈及何以如此配置——甚至有些时候连配置步骤都十分简略，以至于根本无法按照其对应的命令进行配置。故而笔者在此给出一些小小的见解，但一家之言，供诸读者一观。

## VRF的介绍

> In [IP-based](https://en.wikipedia.org/wiki/Internet_Protocol) [computer networks](https://en.wikipedia.org/wiki/Computer_network), **virtual routing and forwarding** (**VRF**) is a technology that allows multiple instances of a [routing table](https://en.wikipedia.org/wiki/Routing_table) to co-exist within the same router at the same time. One or more logical or physical interfaces may have a VRF and these VRFs do not share routes. Therefore, the packets are only forwarded between interfaces on the same VRF. VRFs are the [TCP/IP](https://en.wikipedia.org/wiki/Internet_Protocol) layer 3 equivalent of a [VLAN](https://en.wikipedia.org/wiki/VLAN). Because the routing instances are independent, the same or overlapping [IP addresses](https://en.wikipedia.org/wiki/IP_address) can be used without conflicting with each other. Network functionality is improved because network paths can be segmented without requiring multiple routers.
>
> （摘自Wikipedia[[1]](#[Virtual routing and forwarding - Wikipedia](https://en.wikipedia.org/wiki/Virtual_routing_and_forwarding))）

VRF说白了，就是一种三层的VPN技术，通过在一台物理设备上，创建出多台虚拟设备，每一个虚拟设备都有自己独立的路由表。VRF主要用于实现_路由隔离_。配置的主要方法，是将三层接口和创建出来的VPN实例进行绑定，从而将该接口对应的路由收集到对应VPN实例的路由表中，而非全局路由表，从而实现不同接口、不同VPN实例之间的路由隔离。接触VPN较少的读者，可以将其和VLAN技术做一个类比——即VRF是三层的VLAN。

> ### What it does
>
> - **Overlapping IP Addresses:**
>
>   This allows different networks or customers to use the same IP address ranges without interfering with each other. 
>
> - **Isolation:**
>
>   Each VRF (Virtual Routing and Forwarding) instance is isolated, meaning traffic destined for one VRF cannot reach devices in another VRF. 
>
> - **Simplifies Management:**
>
>   VRF Lite can simplify network management by allowing for the logical segmentation of a network. 
>
> - **No MPLS Required:**
>
>   Unlike full VRF implementations, VRF Lite does not require MPLS (Multiprotocol Label Switching) or MP-BGP (Multiprotocol Border Gateway Protocol). 
>
> - **Creates Virtual Routing Tables:**
>
>   VRF Lite essentially carves up a single physical router into multiple logical routers, each with its own routing table. 
>
> 
>
> ### How it works:
>
> - **Interfaces:** Each interface on the router is assigned to a specific VRF. 
>
> - **Routing Tables:** Each VRF has its own routing table, independent of other VRFs. 
>
> - **Forwarding:** Traffic is forwarded based on the routing table associated with the VRF to which the incoming interface belongs. 
>
> （摘自Cisco Meraki Documentation[[2](#[VRF - Cisco Meraki Documentation](https://documentation.meraki.com/MS/Layer_3_Switching/VRF))]）

VPF技术最早是为MPLS VPN而产生的，因此最早是基于BGP进行路由学习的。然而VRF的特性实在太过有效，以至于各厂商纷纷为其扩展了许多路由协议的支持，这种技术可以称之为VPF-Lite，即不基于MPLS的VRF。此外，MCE技术也应运而生，作为传统MPLS VPN的有力扩展。

废话少说，所谓“实践出真知”，我们接下来就开始进行相关的配置实验。

## 实验配置

### 实验环境

以下所有实验均为MCE设备上VRF Lite的配置，使用华为O3社区的eNSP Pro云实验环境进行实验模拟，实验中使用的交换机为CE6800系列。

### 一、基于静态路由的VPN互访（下一跳为本设备的接口）

#### 网络规划

* 设备拓扑

  ![image-20250916104657928](D:\Notes\assets\image-20250916104657928.png)

* 设备基本信息

  | Device | Interface | Interface Type | VLAN |   IPv4 Address    |
  | :----: | :-------: | :------------: | :--: | :---------------: |
  |  CE-1  |  GE1/0/9  |       L2       |  20  |         /         |
  |        | GE1/0/10  |       L2       |  10  |         /         |
  |        | Vlanif10  |       L3       |  /   | 192.168.10.254/24 |
  |        | Vlanif20  |       L3       |  /   | 192.168.20.254/24 |
  |  PC-1  |  GE1/0/0  |       L3       |  /   |  192.168.10.1/24  |
  |  PC-2  |  GE1/0/0  |       L3       |  /   |  192.168.20.1/24  |

* VPN实例信息

  | Device | VPN Instance |  RD  | IRT  | ERT  | Binding Interface |
  | :----: | :----------: | :--: | :--: | :--: | :---------------: |
  |  CE-1  |    VPN_A     | 1:10 |  /   |  /   |     Vlanif10      |
  |        |    VPN_B     | 1:20 |  /   |  /   |     Vlanif20      |
  
* 静态路由信息

  | Device | VPN Instance |   Destination   | Next-Hop |
  | :----: | :----------: | :-------------: | :------: |
  |  CE-1  |    VPN_A     | 192.168.20.0/24 | Vlanif20 |
  |        |    VPN_B     | 192.168.10.0/24 | Vlanif10 |

#### 设备配置

* CE-1

  ```
  sy im
  v b 10 20
  ip vpn VPN_A
  route-d 1:10
  ip vpn VPN_B
  route-d 1:20
  int g1/0/9
  p l a
  p d v 20
  int g1/0/10
  p l a
  p d v 10
  int vl10
  ip bind vpn VPN_A
  ip add 192.168.10.254 24
  int vl20
  ip bind vpn VPN_B
  ip add 192.168.20.254 24
  ip route-static vpn VPN_A 192.168.20.0 24 vl20
  ip route-static vpn VPN_B 192.168.10.0 24 vl10
  q
  ```

* PC-1和PC-2配置略

#### 结果验证

* [ ] 检查设备的接口地址信息和绑定实例：`dis ip int b`、`dis ip vpn int`
* [ ] 检查设备路由表信息：`dis ip routing`、`dis ip routing vpn VPN_A`、`dis ip routing vpn VPN_A`
* [ ] 检查设备连通性，即VPN实例间的互通性：使用PC-1 ping PC-2

### 二、基于静态路由的VPN互访（下一跳为其他设备的地址）

#### 网络规划

* 设备基本信息

  | Device | Interface | Interface Type | VLAN |   IPv4 Address    |
  | :----: | :-------: | :------------: | :--: | :---------------: |
  |  CE-1  |  GE1/0/9  |       L2       |  20  |         /         |
  |        | GE1/0/10  |       L2       |  10  |         /         |
  |        | Vlanif10  |       L3       |  /   | 192.168.10.254/24 |
  |        | Vlanif20  |       L3       |  /   | 192.168.20.254/24 |
  |  PC-1  |  GE1/0/0  |       L3       |  /   |  192.168.10.1/24  |
  |  PC-2  |  GE1/0/0  |       L3       |  /   |  192.168.20.1/24  |

* VPN实例信息

  | Device | VPN Instance |  RD  | IRT  | ERT  | Binding Interface |
  | :----: | :----------: | :--: | :--: | :--: | :---------------: |
  |  CE-1  |    VPN_A     | 1:10 |  /   |  /   |     Vlanif10      |
  |        |    VPN_B     | 1:20 |  /   |  /   |     Vlanif20      |

* 静态路由信息

  | Device | VPN Instance |   Destination   |   Next-Hop   |
  | :----: | :----------: | :-------------: | :----------: |
  |  CE-1  |    VPN_A     | 192.168.20.0/24 | 192.168.10.1 |
  |        |    VPN_B     | 192.168.10.0/24 | 192.168.20.1 |

#### 设备配置

* CE-1（由于大部分配置相同，故只修改静态路由的配置）

  ```
  ip route-static vpn VPN_A 192.168.20.0 24 vpn VPN_B 192.168.20.1
  ip route-static vpn VPN_B 192.168.10.0 24 vpn VPN_A 192.168.10.1
  ```

#### 结果验证

* [ ] 检查设备的接口地址信息和绑定实例：`dis ip int b`、`dis ip vpn int`
* [ ] 检查设备路由表信息：`dis ip routing`、`dis ip routing vpn VPN_A`、`dis ip routing vpn VPN_A`
* [ ] 检查设备连通性，即VPN实例间的互通性：使用PC-1 ping PC-2

### 三、基于MP-BGP的VPN互访（直连路由）

#### 网络规划

* 设备基本信息

  | Device | Interface | Interface Type | VLAN |   IPv4 Address    |
  | :----: | :-------: | :------------: | :--: | :---------------: |
  |  CE-1  |  GE1/0/9  |       L2       |  20  |         /         |
  |        | GE1/0/10  |       L2       |  10  |         /         |
  |        | Vlanif10  |       L3       |  /   | 192.168.10.254/24 |
  |        | Vlanif20  |       L3       |  /   | 192.168.20.254/24 |
  |  PC-1  |  GE1/0/0  |       L3       |  /   |  192.168.10.1/24  |
  |  PC-2  |  GE1/0/0  |       L3       |  /   |  192.168.20.1/24  |

* VPN实例信息

  | Device | VPN Instance |  RD  |  IRT  |  ERT  | Binding Interface |
  | :----: | :----------: | :--: | :---: | :---: | :---------------: |
  |  CE-1  |    VPN_A     | 1:10 | 20:10 | 10:20 |     Vlanif10      |
  |        |    VPN_B     | 1:20 | 10:20 | 20:10 |     Vlanif20      |

* BGP信息

  | Device | BGP AS | VPN Instance |
  | :----: | :----: | :----------: |
  |  CE-1  | 65000  |    VPN_A     |
  |        |        |    VPN_B     |

#### 设备配置

* CE-1（由于大部分配置相同，故只修改BGP的配置）

  ```
  bgp 65000
   private-4-byte-as enable
   #
   ipv4-family unicast
   #
   vpn-instance VPN_A
   #
   ipv4-family vpn-instance VPN_A
    import-route direct
   #
   ipv4-family vpn-instance VPN_B
    import-route direct
  #
  ```

#### 结果验证

* [ ] 检查设备的接口地址信息和绑定实例：`dis ip int b`、`dis ip vpn int`
* [ ] 检查BGP信息：`dis BGP vpn VPN_A routing-table`、`dis BGP vpn VPN_A routing-table`
* [ ] 检查设备路由表信息：`dis ip routing`、`dis ip routing vpn VPN_A`、`dis ip routing vpn VPN_A`
* [ ] 检查设备连通性，即VPN实例间的互通性：使用PC-1 ping PC-2

### 四、基于MP-BGP的VPN互访（与路由协议联动）

#### 网络规划

* 设备基本信息

  | Device | Interface | Interface Type | VLAN |   IPv4 Address    |
  | :----: | :-------: | :------------: | :--: | :---------------: |
  |  CE-1  |  GE1/0/9  |       L2       |  20  |         /         |
  |        | GE1/0/10  |       L2       |  10  |         /         |
  |        | Vlanif10  |       L3       |  /   | 192.168.10.254/24 |
  |        | Vlanif20  |       L3       |  /   | 192.168.20.254/24 |
  |  PC-1  |  GE1/0/0  |       L3       |  /   |  192.168.10.1/24  |
  |  PC-2  |  GE1/0/0  |       L3       |  /   |  192.168.20.1/24  |

* VPN实例信息

  | Device | VPN Instance |  RD  |  IRT  |  ERT  | Binding Interface |
  | :----: | :----------: | :--: | :---: | :---: | :---------------: |
  |  CE-1  |    VPN_A     | 1:10 | 20:10 | 10:20 |     Vlanif10      |
  |        |    VPN_B     | 1:20 | 10:20 | 20:10 |     Vlanif20      |

* BGP信息

  | Device | BGP AS | VPN Instance |
  | :----: | :----: | :----------: |
  |  CE-1  | 65000  |    VPN_A     |
  |        |        |    VPN_B     |

* OSPF信息

  | Device | OSPF Process | VPN Instance | Area |     Network     |
  | :----: | :----------: | :----------: | :--: | :-------------: |
  |  CE-1  |      1       |    VPN_A     |  1   | 192.168.10.0/24 |
  |        |      2       |    VPN_B     |  2   | 192.168.20.0/24 |

#### 设备配置

* CE-1（由于大部分配置相同，故只修改BGP和路由协议的配置）

```
bgp 65000
 private-4-byte-as enable
 #
 ipv4-family unicast
 #
 ipv4-family vpn-instance VPN_A
  import-route ospf 2
 #
 ipv4-family vpn-instance VPN_B
  import-route ospf 3
#
ospf 2 router-id 10.0.0.2 vpn-instance VPN_A
 area 0.0.0.1
  network 192.168.10.0 0.0.0.255
#
ospf 3 router-id 10.0.0.3 vpn-instance VPN_B
 area 0.0.0.2
  network 192.168.20.0 0.0.0.255
#
```

#### 结果验证

* [ ] 检查设备的接口地址信息和绑定实例：`dis ip int b`、`dis ip vpn int`
* [ ] 检查BGP信息：`dis BGP vpn VPN_A routing-table`、`dis BGP vpn VPN_A routing-table`
* [ ] 检查设备路由表信息：`dis ip routing`、`dis ip routing vpn VPN_A`、`dis ip routing vpn VPN_A`
* [ ] 检查设备连通性，即VPN实例间的互通性：使用PC-1 ping PC-2

## 总结

VRF技术脱胎于MPLS VPN，后才被扩展和开发成为单独的VRF-Lite特性，故而有许多配置都需要依赖传统的MPLS或者相应的方式进行，这大大降低了VRF的可扩展性。即便如此，VRF技术仍旧作为VPN技术的一大代表，在网络的方方面面都有着大量的运用，是一种强大的技术手段。此外，由于本文只聚焦于简单的VRF Lite应用，故之后笔者可能会在其他文章中会深入探讨如何和MPLS以及其他VPN技术结合使用。

## 附录

### 参考资料

1. #### [Virtual routing and forwarding - Wikipedia](https://en.wikipedia.org/wiki/Virtual_routing_and_forwarding)
2. #### [VRF - Cisco Meraki Documentation](https://documentation.meraki.com/MS/Layer_3_Switching/VRF)
3. #### [CloudEngine S5700 S6700 产品文档](https://support.huawei.com/hedex/hdx.do?docid=EDOC1100277347&tocURL=resources%2Fhedex-homepage.html)
