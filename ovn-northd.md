
## 逻辑流表结构
ovn-northd的一个主要功能是生成OVN南向数据库中的Logical_Flow表。本部分描述ovn-northd是怎么为交换机和路由器的逻辑数据通路生成逻辑流表的。

### 逻辑交换机数据通路

#### Ingress Table 0: 进入控制和ingress端口安全 - 二层
入口表0包含以下逻辑流表：
* 优先级100的流丢弃打VLAN标签的包或以太网源地址多播包
* 优先级50的流对每一个启用的逻辑端口实现了入口端口安全。对于启用端口安全的逻辑端口，这些流匹配inport和合法的eth.src地址后将包传入下一个流表中。对于未启用端口安全的逻辑端口，仅仅将匹配inport的包传入下一个流表

这样流表中，对于未启用的逻辑端口没有相关的流，因为逻辑流表默认丢弃的行为将对弃掉从这些端口进入的包

#### Ingress Table 1: Ingress端口安全 - IP
入口表1包含如下逻辑流:
* 对于端口安全集中有一个或更多IPv4或/和IPv6的元素都有如下流表
  * 优先级90的流允许匹配inport、有效eth.src和有效ip4.src地址的IPv4数据流
  * 优先级90的流允许具有合法eth.src的DHCP发现包。这是必须的，因为在IPv4地址还没有设置时，DHCP发现包的IPv4地址是0.0.0.0
  * 优先级90的流允许具有IPv6地址并且匹配inport、有效eth.src和有效ip6.src地址的数据流通过
  * 优先级90的流允许具有有效eth.src地址的IPv6 DAD(Duplicate Address Detection)检测流通过。这个是必须的，因为 DAD  include  requires  joining an multicast group and sending neighbor solicitations for the newly assigned address. Since no  address  is  yetassigned, these are sent from the unspecified IPv6 ad‐dress (::).
  * 优先级80的流表丢弃其他匹配到inport和有效eth.src的IP(IPv4和IPv6)数据流
* 一个优先级0的fallback流将其他所用数据包导入下一流表

#### Ingress Table 2: Ingress端口安全 - 邻居发现
入口表2包含如下逻辑流:
* 对于端口安全集中的每一个端口安全项
  * 优先级90的流允许匹配inport和有效eth.src及arp.sha的ARP数据流。如果端口安全项有一个或多个IPv4地址，也需要匹配合法的arp.spa
  * 优先级90的流允许匹配inport、合法eth.src和nd.ssl/nd.tll的IPv6 邻居请求和邻居通告数据流。如果端口安全项有一个或多个IPv6地址，对于邻居通告此流还需要匹配合法的nd.target
  * 优先级90的流丢弃其他匹配inport和合法eth.src的ARP、IPv6邻居请求和邻居通过流
* 一个优先级0的fallback流将其他所有包advance到下一流表

#### Ingress Table 3: from-lport Pre-ACLs
这个流表为数据包在入口ACLs流表进行有状态的ACL处理做准备。此流表包含一个优先级0的流将数据流导入到下一流表。如果在数据通路中使用了有状态ACLs，此流表中会添加一个优先级为100的流hat sets a hint (with reg0[0] = 1; next;) for  table Pre-stateful to send IP packets to the connection tracker before eventually advancing to ingress table ACLs. 如果像route端口或localnet端口不能使用ct(), 此流表中会被添加一个优先级为110的流跳过有状态ACLs。

此流表也有一个优先级110的流将所用交换机数据通路中匹配eth.dst == E的数据包导入下一流表中。其中E是在NB_Global表中options:svc_monitor_mac列中定义的服务监测器的mac地址。

#### Ingress Table 4: Pre-LB
这个流表为可能在入口LB和Stateful流表中进行有状态负载均衡处理的流做准备。其中包含一个优先级0的流简单的将所用的数据流导入到下一流表做处理。另外也包含一个优先级110的流将IPv6邻居发现数据流导入下一流表处理。如果在ovn北向数据库中一个逻辑交换机数据通路被配置了具有虚拟IP地址(和端口)的负载均衡，对于每一个虚拟IP地址VIP都会添加一个优先级100的流。对于IPv4类型的VIPs，匹配规则为ip && ip4.dst == VIP。对于IPv6类型的VIPs，匹配规则为ip && ip6.dst == VIP。此流的动作是reg0[0] = 1; next; to act as a hint for table Pre-stateful to send IP packets to the connection tracker  for  packet  defragmentation  before  eventually  advancing  to  ingress table LB. 如果启用了controller_event并且OVN北向数据库中添加了一个具有空的后端的负载均衡规则，则会添加一个优先级130的流，当chassis收到这样一个数据包时会产生ovn-controller事件If event-elb meter has been previously created, it will be associated to the empty_lb logical flow。

此流表也有一个优先级110的流将所用交换机数据通路中匹配eth.dst == E的数据包导入下一流表中。其中E是在NB_Global表中options:svc_monitor_mac列中定义的服务监测器的mac地址。

此流表也有一个优先级110的流将所用交换机通路中inport == I的数据流导入到下一流表中。其中I是逻辑路由端口的对端。添加这一流的目的是跳过对从逻辑路由器通路进入逻辑交换机通路数据流的追踪。

#### Ingress Table 5: Pre-stateful
这个流表对所用可能在下一流表进行状态处理的数据流进行准备工作。包含一个优先级0的流简单的将所用流量导入下一流表进行处理。另外一个优先级100的流通过使用ct_next动作将在前面表中进行标记(reg0[0] == 1)的数据包进行链接追踪。

#### Ingress Table 6: from-lport ACLs
这个表中的流完全由OVN北向数据库ACL流表中from-lport方向上的ACL产生。The priority values from the ACL table have a limited range and have 1000 added to them to leave room for OVN default flows at both higher and lower priorities.

* allow ACLS翻译成逻辑流中的netx;动作。如果在数据通路中有任意的有状态ACLs，则ACLs被翻译为ct_commit; next;(暗含在下一流表中对数据流进行conntrack)
* 对于新连接allow-related ACLs翻译成逻辑流表中的ct_commit(ct_label=0/1); next;动作，对于已经存在的链接翻译成reg0[1] = 1; next;
* regect ACLS 对于TCP连接翻译成tcp_reset{output <-> inport; next(pipeline=egress,table=5);}动作，对于UDP连接翻译成icmp4/icmp6动作
* Other ACLs 对于新的或未追踪连接翻译成drop;动作；对于未知连接添加ct_label标记，表明此连接之前被允许但是因为策略改变后续不再被允许

此流表包含一个优先级0的流，其动作为next，默认的ACL允许包通过。如果逻辑数据通路有有状态ACL，则会添加下面的流：
* 一个优先级1的流设置建议将IP数据流提交到连接跟踪器(其动作为reg0[1] = 1; next;)。
This is needed for the default allow  policy  because,  while  the initiator’s  direction  may  not have any stateful rules, the server’s may and then its return traffic would not  be  known and marked as invalid.
* 一个优先级65535的流对于一个已被连接追踪的练级，如果这个连接没有ct_label.blocked标志，则此连接的回复允许通过。We  only handle  traffic  in  the reply direction here because we want all packets going  in  the  request  direction  to  still  go through the flows that implement the currently defined policy based on ACLs. 如果一个连接不再被策略所允许通过，则在其回复连接上设置ct_label.blocked，这样其回复连接也不被允许通过。
* 一个优先级65535的流允许一个已被连接追踪的流相关的流(如ICMP Port Unreachable from  a  non-listening  UDP port)通过，只要已被连接追踪的流没有设置ct_label.blocked。
* 一个优先级65535的流丢弃被连接追踪标记为非法的所用数据流。
* 一个优先级65535的流丢弃被标记为ct_label.blocked的回复流，ct_label.blocked意味着因为策略改变连接不被允许通过。在这里忽略请求方向的包以便新建的ACL重新允许此连接。
* 一个优先级65535的流允许IPv6邻居请求、邻居发现，路由请求和路由发现包
* 对于每一个逻辑交换机数据通路添加一个34000的逻辑流匹配eth.dst = E  to allow the  service monitor reply packet destined to ovn-controller with the action next, 其中E是在NB_Global表中options:svc_monitor_mac列中定义的服务监测器的mac地址。

#### Ingress Table 7: from-lport QoS Marking
这个表中的逻辑流完全由OVN北向数据库QoS表中action列方向为from-lport的项
产生。
* For every qos_rules entry in a logical switch with DSCP marking enabled, a flow will be added at the  priority  mentioned in the QoS table.
* 一个优先级0的fallback流匹配所用包，并将其导入下一流表处理

#### Ingress Table 8: from-lport QoS Meter
这个表中的逻辑流完全由OVN北向数据库QoS表中bandwidth列方向为from-lport的项产生
* 对于启用metering的逻辑交换机中的每一条qos_rules项，a  flow will be added at the priorirty mentioned in the QoS table。
* 一个优先级0的fallback流匹配所用包，并将其导入下一流表处理

#### Ingress Table 9: LB
包含一个优先级0的流简单的将所用数据流导入下一流表处理。

一个优先级65535的流对于所用交换机数据通路中inport == I的数据流导入下一流表处理。其中I是逻辑交换机端口的对端。添加此流表是对从逻辑路由器到逻辑交换机的包跳过连接追踪。

对于已经建立的连接，一个优先级65534的流匹配ct.est && !ct.rel && !ct.new && !ct.ivn然后设置reg0[2] = 1; next;为Stateful流表添加将包通过连接追踪后进行NAT的建议。（The packet will automatically get DNATed to the  same  IP address as the first packet in that connection.）

#### Ingress Table 10: Stateful
* 对于每一个配置在北向数据库中交换机上匹配L4端口PROT、协议P和IP地址VIP的负载均衡项，一个优先级120的流会添加到这个流表中。对于IPv4的VIPs，流匹配ct.new && ip && ip4.dst == VIP && P && P.dst == PORT.对于IPv6的VIPs，流匹配ct.new && ip && ip6.dst == VIP && P && P.dst == PORT. 流的动作是ct_lb(args), 其中args包含逗号分割的需要负载均衡到的IP地址（和可选的端口号）。args中的IP地址的地址族需要与VIP的地址族相同。如果启用了健康检查，则args将仅包含OVN南向数据库中service monitor项为online或空的后端。

* 对于每一个配置在北向数据库中交换机上规则中仅匹配IP地址VIP的负载均衡项，一个优先级为110的流会被添加。对于IPv4地址VIPs，流匹配ct.new && ip && ip4.dst == VIP。对于IPv6的VIPs，流匹配ct.new && ip && ip6.dist == VIP。流的动作是ct_lb(args)，其中args包含逗号分割的与VIP地址族相同的IP地址

* 一个优先级100的流基于之前流表提供的建议(with a match for reg0[1] == 1)通过ct_commit; next;动作将包提交连接追踪器

* 一个优先级100的流基于之前流表提供的建议(with a match for reg0[2] == 1)通过ct_lb;将包送入连接追踪器

* 一个优先级0的流简单的将数据流移入下一流表处理

#### Ingress Table 11: Pre-Hairpin
* 对于所用配置的负载均衡VIPs，一个优先级2的流匹配需要被hairpin的数据流，例如在负载均衡后目的IP地址匹配源IP，此流设置reg0[6] = 1并且执行ct_snat(VIP) to force replies to these packets to come back through OVN.
* 对于所有已配置的负载均衡VIPs，一个优先级1的流匹配被hairpin数据的的回复流，例如目的IP是VIP，源IP是后端IP以及源L4端口是后端端口，这个流设置reg0[6] = 1并且执行ct_snat.
* 一个优先级0的流简单的将数据包导入下一流表处理

#### Ingress Table 12: Hairpin
* 一个优先级1的流hat hairpins traffic matched by non-default flows in the Pre-Hairpin table。Hairpinning在二层进行，交换以太网地址后包被发回入口端口。
* 一个优先级0的流简单的将数据包导入下一流表处理

#### Ingress Table 13: ARP/ND responder
这个流表在逻辑交换机中实现对已知IP的ARP/ND应答。ARP应答流的优势是通过在本地响应ARP请求而不是发送到其他hypervisor来限制ARP的广播。一个常见的场景是当入口是一个和VIF相关联的逻辑端口时，the broadcast is responded to on the local hypervisor rather than broadcast across the whole network and  responded  to by the destination VM.