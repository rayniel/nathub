# NATHub 数据模型设计

## 1. 设计原则

NATHub 的数据模型不应只围绕设备与 NAT 规则构建，而应覆盖完整的“实例 -> 网络域 -> 出口 -> 公网资源 -> 底层规则”链路。

设计原则：

- 高层对象优先：优先建模 Published Service，而不是只建模 NAT Rule
- 设备与计算平台分离：Provider 与 Driver 解耦
- 网络域抽象：支持 PVE SDN VLAN / VNet / Subnet、OpenStack provider network / tenant network / trunk
- 期望状态驱动：支持 desired state、actual state、sync status
- 资源池化：支持公网 IP / 端口池分配与回收
- 可映射到底层设备：支持一个发布对象映射多个设备规则

---

## 2. 设备与网关层

### 2.1 devices
保存受管设备信息。

字段建议：

- id
- name
- vendor_type
- mgmt_ip
- mgmt_port
- auth_type
- username
- encrypted_secret
- transport_preference
- enabled
- site
- description
- last_seen
- created_at
- updated_at

其中 `vendor_type` 取值建议：

- openwrt
- pfsense
- huawei_srg

`transport_preference` 可取：

- ssh
- http
- telnet

---

### 2.2 gateways
表示逻辑出口网关。

字段建议：

- id
- name
- device_id
- gateway_type
- site
- enabled
- priority
- health_status
- capabilities_json
- created_at
- updated_at

说明：

- 一个 gateway 通常绑定一个 device
- 但业务上更适合用 gateway 作为出口选择对象

---

### 2.3 wan_interfaces
表示网关出口信息。

字段建议：

- id
- gateway_id
- device_id
- name
- display_name
- ip_addr
- gateway
- isp_name
- enabled
- priority
- is_dynamic
- public_ip
- created_at
- updated_at

---

## 3. 计算平台层

### 3.1 compute_providers
表示外部计算平台。

字段建议：

- id
- name
- provider_type
- api_endpoint
- auth_type
- encrypted_credentials
- enabled
- metadata_json
- created_at
- updated_at

其中 `provider_type` 建议支持：

- pve
- openstack

---

### 3.2 compute_instances
表示来自计算平台的 VM / 实例。

字段建议：

- id
- provider_id
- external_id
- name
- instance_type
- tenant_name
- tenant_id
- project_name
- project_id
- node_name
- primary_ip
- status
- metadata_json
- last_synced_at
- created_at
- updated_at

说明：

- PVE 可使用 VMID 作为 external_id
- OpenStack 可使用 instance UUID 作为 external_id

---

## 4. 网络域层

### 4.1 network_domains
统一抽象实例所属网络域。

用于表达：

- PVE Linux bridge
- PVE SDN Zone / VNet / Subnet
- PVE VLAN
- OpenStack provider network
- OpenStack tenant network
- OpenStack trunk subport
- 其他可达域

字段建议：

- id
- provider_id
- domain_type
- name
- external_id
- zone_name
- vnet_name
- subnet_name
- bridge_name
- segmentation_type
- segmentation_id
- cidr
- vrf_name
- site
- metadata_json
- created_at
- updated_at

其中 `domain_type` 可支持：

- pve_bridge
- pve_sdn_vnet
- pve_vlan
- openstack_provider_network
- openstack_tenant_network
- openstack_trunk_subport
- manual_network

其中 `segmentation_type` 可支持：

- vlan
- vxlan
- flat
- trunk
- none

说明：

- PVE SDN 的 Zone / VNet / Subnet、OpenStack 的 provider network / trunk VLAN 等都可统一映射到 network domain 抽象中
- 对 VLAN 型网络应记录 segmentation_id

---

### 4.2 instance_network_bindings
表示实例与网络域的绑定关系。

字段建议：

- id
- instance_id
- network_domain_id
- ip_addr
- mac_addr
- nic_name
- is_primary
- metadata_json
- created_at
- updated_at

说明：

- 一个实例可绑定多个网络域
- trunk 或多网卡场景必须允许多条记录

---

## 5. 网关可达性层

### 5.1 gateway_reachability_bindings
表示某个网关能够服务哪些 network domain。

这是多出口环境非常关键的表。

字段建议：

- id
- gateway_id
- network_domain_id
- inside_interface_name
- inside_ip
- inside_vlan_id
- route_mode
- requires_trunk
- requires_subinterface
- priority
- enabled
- metadata_json
- created_at
- updated_at

说明：

- 用于表达某出口网关能否到达某个 PVE VLAN / VNet / OpenStack network
- `route_mode` 可表示：
  - l2_direct
  - l3_routed
  - trunk_vlan
  - subinterface
  - overlay

这样系统在分配出口时就可以先判断 reachability，而不是盲选。

---

## 6. 公网资源池层

### 6.1 public_endpoints
表示可用公网地址资源。

字段建议：

- id
- gateway_id
- wan_interface_id
- public_ip
- endpoint_type
- enabled
- status
- metadata_json
- created_at
- updated_at

其中 `endpoint_type` 可支持：

- ip
- ip_range
- vip

---

### 6.2 port_pools
表示端口池。

字段建议：

- id
- name
- public_endpoint_id
- gateway_id
- protocol
- start_port
- end_port
- allocation_strategy
- enabled
- metadata_json
- created_at
- updated_at

---

### 6.3 port_allocations
表示端口分配记录。

字段建议：

- id
- pool_id
- public_endpoint_id
- gateway_id
- protocol
- public_port
- allocated_to_type
- allocated_to_id
- status
- allocated_at
- released_at
- metadata_json

其中 `allocated_to_type` 可支持：

- published_service
- manual_rule

---

## 7. 发布层

### 7.1 published_services
这是平台最核心的业务对象。

用于表示某个实例或目标地址的服务被对外发布。

字段建议：

- id
- name
- source_type
- source_instance_id
- source_manual_ip
- network_domain_id
- tenant_id
- project_id
- protocol
- target_ip
- target_port
- public_endpoint_id
- public_ip
- public_port
- gateway_id
- wan_interface_id
- publish_mode
- desired_state
- actual_state
- sync_status
- expires_at
- created_by
- description
- metadata_json
- created_at
- updated_at

其中 `source_type` 可支持：

- manual
- pve_vm
- openstack_instance

其中 `publish_mode` 可支持：

- static_assign
- auto_allocate

说明：

- 如果来源是手工地址，则用 `source_manual_ip`
- 如果来源是实例，则关联 `source_instance_id`
- `network_domain_id` 用于做出口可达性判断
- 发布对象是高层业务对象，NAT 规则只是其底层实现之一

---

## 8. 底层规则层

### 8.1 nat_rules
表示底层 NAT 规则。

字段建议：

- id
- published_service_id
- name
- device_id
- gateway_id
- wan_interface_id
- nat_type
- protocol
- external_port_start
- external_port_end
- internal_ip
- internal_port_start
- internal_port_end
- source_cidr
- enabled
- description
- desired_state
- actual_state
- sync_status
- last_sync_at
- last_error
- extra_options_json
- created_at
- updated_at

其中 `nat_type` 可支持：

- port_forward
- static_nat
- source_nat
- destination_nat

---

### 8.2 policy_rules
表示安全策略 / ACL / zone policy 等。

字段建议：

- id
- published_service_id
- device_id
- gateway_id
- policy_type
- name
- enabled
- desired_state
- actual_state
- sync_status
- last_sync_at
- last_error
- extra_options_json
- created_at
- updated_at

说明：

- 某些设备发布服务时需要 NAT + Policy 联动
- 该表必须预留，即使 MVP 尚未 fully implement

---

### 8.3 device_rule_mappings
用于统一记录高层发布对象与底层资源的映射。

字段建议：

- id
- published_service_id
- resource_type
- resource_id
- device_id
- gateway_id
- created_at

其中 `resource_type` 可支持：

- nat_rule
- policy_rule
- address_object
- acl_object

---

## 9. 任务与审计层

### 9.1 jobs
字段建议：

- id
- job_type
- target_type
- target_id
- device_id
- gateway_id
- status
- payload_json
- result_json
- error_message
- started_at
- finished_at
- created_at

---

### 9.2 job_steps
建议新增。

字段建议：

- id
- job_id
- step_name
- step_order
- status
- request_json
- result_json
- error_message
- started_at
- finished_at
- created_at

建议 `step_name` 示例：

- resolve_network_domain
- select_gateway
- allocate_public_port
- create_nat_rule
- create_policy_rule
- save_device_config
- verify_publish_result
- release_public_port

---

### 9.3 job_logs
字段建议：

- id
- job_id
- step_id
- level
- message
- created_at

---

### 9.4 audit_logs
字段建议：

- id
- user_id
- action
- target_type
- target_id
- before_json
- after_json
- result
- created_at

---

## 10. 同步与快照层

### 10.1 sync_states
可选增强表，用于记录对象同步状态。

字段建议：

- id
- object_type
- object_id
- desired_state
- actual_state
- sync_status
- drift_reason
- last_checked_at
- created_at
- updated_at

---

### 10.2 device_backups
字段建议：

- id
- device_id
- backup_type
- content
- created_at

用于后期：

- 配置回滚
- 差异比对
- 漂移校验

---

## 11. 未来扩展建议

后期还可以继续扩展：

- tenants / projects
- address_objects
- dns_records
- notifications
- approval_requests
- quotas

---

## 12. 建模总结

NATHub 的核心链路应当是：

> Compute Provider  
> → Compute Instance  
> → Network Domain  
> → Gateway Reachability Binding  
> → Public Endpoint / Port Pool  
> → Published Service  
> → NAT / Policy Device Rules

这样才能适配：

- 多出口网关
- PVE SDN VLAN / VNet
- OpenStack provider network / trunk
- Huawei CLI 复杂动作
- 服务发布生命周期管理
- 自动回收与漂移修复
