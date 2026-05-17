# NATHub 适配器设计

## 1. 设计原则

- 设备差异通过 Adapter 层隔离
- 传输方式与设备能力分离
- 平台内部使用统一规则模型
- Adapter 只负责把统一规则翻译成设备可执行操作

---

## 2. 基础接口

建议定义统一抽象接口：

- test_connectivity(device)
- get_capabilities(device)
- list_nat_rules(device)
- validate_rule(device, rule)
- create_nat_rule(device, rule)
- update_nat_rule(device, rule)
- delete_nat_rule(device, rule)

---

## 3. OpenWrtAdapter

推荐方式：

- SSH
- uci 读写 firewall
- fw4 reload

原则：

- 优先修改持久化配置
- 避免首期直接写 nftables 运行时规则

典型流程：

1. 读取现有 firewall redirect
2. 根据统一模型构造 redirect 配置
3. `uci commit firewall`
4. `fw4 reload`
5. 回读验证规则存在

---

## 4. PfSenseAdapter

优先方式：

- HTTP API
- 备选 SSH

原则：

- 若环境提供可用 API，则优先 API
- 若必须 SSH，则所有操作必须模板化

能力：

- 查询 NAT Port Forward
- 创建规则
- 删除规则
- 启用/禁用规则

---

## 5. HuaweiSRGAdapter

推荐方式：

- SSH CLI

原则：

- 不允许任意命令执行
- 只允许模板化的 NAT / 策略命令
- 执行后要有回读校验

典型流程：

1. 进入配置模式
2. 生成 nat server / 对应策略命令
3. 提交保存
4. 查询当前配置确认下发结果

---

## 6. 传输层接口

### SSHTransport
能力：

- connect
- exec
- close

### HTTPTransport
能力：

- get
- post
- put
- delete
- session 管理
- TLS 配置
- token 认证

---

## 7. 适配器与传输层关系

- OpenWrtAdapter → SSHTransport
- HuaweiSRGAdapter → SSHTransport
- PfSenseAdapter → HTTPTransport（优先）

---

## 8. 幂等性建议

创建规则前应先查询已有规则：

- 相同规则已存在：视为成功
- 同名规则内容不同：视为冲突
- 不存在：执行创建
