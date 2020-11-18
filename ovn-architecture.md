### 名称
ovn-architecture - 开发虚拟网络(Open Virtual Network架构)

### 描述
OVN(The Open Virtual Network)是一个在虚拟机和容器环境中支持逻辑网络抽象的系统。OVN在OVS已有能力的基础上添加了对逻辑网络抽象的原生支持，如逻辑二层和三层overlays及安全组。DHCP等服务包含其中。同OVS一样，OVN的设计目标是实现一个能够在相当规模下运行的具有生产级质量的系统。

一个物理网络包括物理网线、交换机和路由器。虚拟网络将物理网络扩展到hypervisor或容器平台中，将虚拟机或容器桥接到物理网络中。OVN逻辑网络是一个用软件实现的通过隧道或其他封装技术实现与物理网络隔离的网络。这允许逻辑网络中使用的IP地址或其他地址可以与物理网络中的重合而不产生冲突。逻辑网络可以重新拓扑布置而不需考虑它所运行的物理网络的拓扑。因此，虚拟机可以在不中断网络的情况下从逻辑网络的一部分迁移到另一部分。更多的信息请参阅下面**逻辑网络**部分。

封装层可以阻止逻辑网络中的虚拟机和容器同物理网络上的节点通信。对于虚拟机集群和容器集群，这个是可以接受甚至是期望的，但是在许多情况下虚拟机和容器确实需要和物理网络连接。OVN为这种需求提供了多种形式的网关(gateways)。需要更多信息，请参阅下面**Gateways**部分。

一个OVN拓扑包括下面几个部分:
* 云管理系统(CMS)，作为OVN的最终客户(通过它的用户和管理员)。OVN集成需要安装一个特定的CMS插件和相关的软件。OVN最初是将OpenStack作为CMS的。

* 我们一般说这个CMS，但是在确实存在多个云管理系统管理一个OVN部署的不同部分的场景。

* 在架构的中心位置有
