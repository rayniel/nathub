# NATHub 架构设计

## 1. 项目定位

NATHub 是一个运行在 x86/Linux 控制器上的多出口网络发布与设备编排平台。

它以 NAT / 端口映射为第一阶段核心能力，但整体目标并不局限于“手工管理 NAT 规则”，而是面向以下场景：

- 多厂商出口网关统一管理
- 多出口 / 多公网地址池统一编排
- 面向 PVE / OpenStack 等计算平台的联动
- 面向虚拟机 / 实例的公网服务发布
- 面向复杂网络拓扑的出口选择与规则下发

当前目标设备类型：

- OpenWrt
- pfSense
- Huawei SRG2210

未来联动对象：

- Proxmox VE（PVE）
- OpenStack

---

## 2. 项目目标

NATHub 的核心目标：

- 统一管理多厂商设备上的 NAT / 端口映射规则
- 支持多设备、多出口、多 WAN / 多公网地址
- 支持面向虚拟机 / 实例的服务发布
- 抽象“实例 -> 网络 -> 出口 -> 公网端口 -> 底层设备规则”的完整链路
- 为 PVE / OpenStack 等平台提供统一网络发布入口
- 统一审计、任务执行、冲突检测、状态同步与回收机制

---

## 3. 为什么不能只做 NAT 管理器

如果系统仅围绕 NAT 规则建设，会很快遇到以下问题：

- 实际业务对象是 VM / 实例，而不是静态内网 IP
- 多个出口网关服务于不同网络域，不能随意选择
- PVE SDN / VLAN / VNet 会影响实例是否可达某个出口
- OpenStack provider network / trunk / segmentation 会影响出口绑定关系
- 一个“服务发布”动作可能同时涉及 NAT、策略、保存配置、回读验证
- 资源删除后需要自动回收公网端口和底层规则

因此，NATHub 必须被设计为：

> **以 NAT / 端口映射为起点的多出口网络发布与设备编排平台**

---

## 4. 总体架构

采用 **x86 单控制器架构**：

- 控制器部署在 PVE VM 或独立 x86 Linux 主机
- 控制器通过三层路由直连目标设备管理地址
- MT7621 仅提供路由可达性，不部署 agent 或业务程序
- 所有设备 Driver、云平台 Provider、任务编排、审计与同步逻辑集中在控制器完成

---

## 5. 架构原则

### 5.1 单控制器
所有控制逻辑集中在一个平台中完成，降低边界设备复杂度。

### 5.2 设备层与计算平台层分离
设备网关控制逻辑与 PVE/OpenStack 集成逻辑必须解耦。

### 5.3 网络域抽象
系统不能只记录实例 IP，还必须抽象实例所属网络域，例如：

- PVE SDN Zone / VNet / VLAN
- OpenStack provider network
- OpenStack tenant network
- trunk subport VLAN

### 5.4 可达性优先
在进行 NAT / 服务发布前，系统必须先判断实例所在网络域是否可达目标出口网关。

### 5.5 发布模型优先
面向用户和上层平台的核心对象应是“服务发布”，而不是底层 NAT 规则。

### 5.6 设备能力解耦
底层设备通过 Driver 框架接入，支持 SSH / HTTP API / Telnet 等多种方式。

### 5.7 期望状态驱动
系统应记录 desired state、actual state、sync status，以支持对账、漂移修复与资源回收。

---

## 6. 分层设计

### 6.1 Web / API 层
负责：

- 登录认证
- 管理后台
- 对外 API
- 对接上层平台或自动化系统

---

### 6.2 Orchestration 层
负责：

- 服务发布编排
- 出口选择
- 网络可达性校验
- 资源池分配
- 多步骤任务执行
- 回收与同步

这是平台核心层。

---

### 6.3 Provider 层
负责对接计算平台，例如：

- PVEProvider
- OpenStackProvider

职责包括：

- 拉取实例信息
- 拉取网络信息
- 识别租户 / 项目 / 节点
- 识别实例所属网络域
- 同步实例生命周期

---

### 6.4 Driver 层
负责对接具体网络设备，例如：

- OpenWrtDriver
- PfSenseDriver
- HuaweiSRGDriver

职责包括：

- NAT 规则管理
- Policy / ACL 管理
- System 配置保存
- 设备能力声明
- 配置会话管理

---

### 6.5 Transport 层
负责设备通信方式：

- SSHTransport
- HTTPTransport
- TelnetTransport

---

### 6.6 Data / State 层
负责保存：

- 设备信息
- 出口网关与 WAN 信息
- 计算平台信息
- 实例信息
- 网络域信息
- 服务发布对象
- NAT / Policy 等底层资源映射
- 任务与审计日志
- 资源池与分配记录

---

## 7. 核心对象模型

系统建议以以下高层对象组织业务：

### 7.1 Compute Provider
表示计算平台，例如 PVE、OpenStack。

### 7.2 Compute Instance
表示虚拟机 / 实例。

### 7.3 Network Domain
表示实例所在的网络域，例如：

- PVE SDN VLAN Zone / VNet / Subnet
- OpenStack provider network
- OpenStack tenant network
- OpenStack trunk VLAN 子口

### 7.4 Gateway
表示出口网关的逻辑抽象。

### 7.5 Gateway Reachability Binding
表示某个网关能够服务哪些网络域，以及通过何种 inside 网络接入。

### 7.6 Public Endpoint / Port Pool
表示可分配的公网 IP / 端口资源池。

### 7.7 Published Service
表示实例上的某个服务对外发布，这是平台的核心业务对象。

### 7.8 Device Rule Mapping
表示高层发布对象与底层 NAT / Policy / ACL 等设备规则的映射关系。

---

## 8. PVE / OpenStack 联动必须考虑的网络因素

### 8.1 PVE 阶段
对于 PVE，系统至少要考虑：

- VM 所属节点
- VM 所属 Linux bridge 或 SDN VNet
- Zone / VNet / Subnet
- VLAN ID
- 是否通过 trunk / VLAN-aware bridge 接入
- 对应出口网关能否到达该网络域

---

### 8.2 OpenStack 阶段
对于 OpenStack，系统至少要考虑：

- 实例所属项目 / tenant
- 实例所属 network / subnet
- provider network 或 tenant network
- segmentation type
- segmentation id（如 VLAN ID）
- trunk parent port / subport
- 出口侧 provider network / physical network 的映射关系

---

## 9. 典型数据流

### 9.1 手工发布服务
1. 用户选择目标实例或手工填写目标地址
2. 系统识别目标所属 network domain
3. 系统根据策略选择可用 gateway / wan / public endpoint
4. 校验 reachability、冲突、容量与权限
5. 创建 Published Service
6. 编排底层 NAT / Policy / Save Config 任务
7. 回写实际状态与资源映射
8. 记录审计日志

---

### 9.2 由 PVE / OpenStack 联动发布
1. Provider 层同步实例与网络
2. 上层平台调用 NATHub API 创建发布请求
3. Orchestration 层解析实例网络域
4. 进行出口解析与资源分配
5. 调用 Driver 下发配置
6. 回写发布状态
7. 后续由 Reconciler 周期性校验实例和底层规则是否一致

---

### 9.3 资源回收
1. 实例删除、服务过期或用户主动释放
2. 系统找到对应 Published Service
3. 回收公网端口 / 公网 IP
4. 删除底层 NAT / Policy
5. 保存配置
6. 更新状态并记录审计日志

---

## 10. 首期 MVP 范围

MVP 首期可以聚焦以下范围：

- 设备管理
- 出口网关管理
- WAN / 公网资源管理
- 手工创建 NAT / 服务发布
- OpenWrt 首个闭环 Driver
- 基础任务日志
- 基础审计日志

但模型设计上必须提前为以下能力留出空间：

- PVE 实例同步
- PVE SDN / VLAN / VNet 建模
- 多出口 reachability 约束
- OpenStack provider network / segmentation 建模
- 自动分配公网端口
- 服务发布生命周期管理

---

## 11. 中期演进方向

- PVEProvider
- OpenStackProvider
- Published Service 工作流
- Port Pool / Public Endpoint 自动分配
- Gateway Reachability Policy
- Huawei SRG Telnet Driver
- Policy / ACL 联动
- Reconciler / 漂移修复
- 过期资源自动回收

---

## 12. 总结

NATHub 不应被定义为一个简单的 NAT 管理后台，而应被定义为：

> **面向多出口网关、多网络域和虚拟化平台联动的网络发布与设备编排平台**

NAT 只是第一阶段能力。  
真正长期稳定的系统边界应包括：

- 计算资源
- 网络域
- 出口网关
- 公网资源池
- 服务发布
- 底层设备编排
- 生命周期与回收
