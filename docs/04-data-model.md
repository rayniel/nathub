# NATHub 数据模型设计

## 1. 设备表 devices

用于保存受管设备信息。

字段建议：

- id
- name
- vendor_type
- mgmt_ip
- mgmt_port
- auth_type
- username
- encrypted_secret
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

---

## 2. 出口表 wan_interfaces

用于描述设备上的多个出口。

字段建议：

- id
- device_id
- name
- display_name
- ip_addr
- gateway
- isp_name
- enabled
- priority
- is_dynamic
- created_at
- updated_at

---

## 3. NAT 规则表 nat_rules

字段建议：

- id
- name
- device_id
- wan_interface_id
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
- created_by
- created_at
- updated_at

---

## 4. 任务表 jobs

字段建议：

- id
- job_type
- target_type
- target_id
- device_id
- status
- payload_json
- result_json
- error_message
- started_at
- finished_at
- created_at

状态建议：

- pending
- running
- success
- failed
- partial

---

## 5. 任务日志表 job_logs

字段建议：

- id
- job_id
- level
- message
- created_at

---

## 6. 审计日志表 audit_logs

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

## 7. 配置快照表 device_backups

字段建议：

- id
- device_id
- backup_type
- content
- created_at

可用于后期：

- 配置回滚
- 规则差异比对
- 同步校验
