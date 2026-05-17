# NATHub Driver 设计

## 1. 设计目标

NATHub 虽然以 NAT / 端口映射为第一阶段核心能力，但系统整体不应被限制为“只管理 NAT”的工具。

考虑到以下现实约束：

- OpenWrt 主要通过 SSH / UCI 管理
- pfSense 可能通过 HTTP API 管理，也可能需要 SSH 辅助
- Huawei SRG 很可能仅支持 Telnet / CLI
- 后续操作不仅包括创建 NAT，还包括删除 NAT、查询规则、保存配置、策略控制等动作

因此，NATHub 的设备接入层应设计为：

> **多厂商网络设备 Driver 框架**

而不是仅围绕 NAT CRUD 构建的窄接口 Adapter。

---

## 2. 设计原则

### 2.1 面向设备能力，而不是单一功能
Driver 负责描述一类设备可以完成哪些管理动作，而不仅仅是“创建 NAT”。

### 2.2 传输方式与设备能力分离
设备通信协议与设备配置逻辑必须解耦。  
例如同样是 Huawei SRG，不同环境可能走 Telnet 或 SSH；pfSense 可能走 API 或 SSH。

### 2.3 资源服务化
设备能力按资源域拆分，而不是平铺成无限增长的方法列表。

例如：

- NATService
- PolicyService
- SystemService
- InterfaceService

### 2.4 命令模板化
所有 CLI 设备操作必须模板化，禁止在业务层直接拼接任意命令。

### 2.5 为复杂会话保留结构
CLI 型设备通常需要：

- 登录
- 进入配置模式
- 执行多条命令
- 保存配置
- 回读校验

因此必须考虑 ConfigSession / CommandBatch 这类抽象。

### 2.6 支持能力声明
不同设备支持的协议、资源、动作不同，Driver 应显式声明能力，供服务层和前端使用。

---

## 3. 分层模型

Driver 体系建议拆分为三层：

### 3.1 Transport 层
负责“如何连到设备”。

典型实现：

- SSHTransport
- HTTPTransport
- TelnetTransport

职责：

- 建立连接
- 登录认证
- 执行命令或请求
- 管理超时、重试、关闭连接
- 清理回显与基础结果

---

### 3.2 Driver 层
负责“设备本身支持哪些能力”。

典型实现：

- OpenWrtDriver
- PfSenseDriver
- HuaweiSRGDriver

职责：

- 暴露能力声明
- 暴露资源服务
- 管理配置会话
- 组织设备级动作执行
- 调用 Parser / Template 解释和生成设备语法

---

### 3.3 Resource Service 层
负责“某类资源如何被查询与修改”。

建议资源域至少包括：

- NATService
- PolicyService
- SystemService
- InterfaceService

首期 MVP 可以先实现：

- NATService
- SystemService

后续再扩展：

- PolicyService
- InterfaceService

---

## 4. 为什么不用单纯 Adapter 设计

如果只设计：

- create_nat_rule()
- delete_nat_rule()
- list_nat_rules()

短期看简单，但后续会有几个问题：

### 4.1 方法膨胀
后面增加策略、地址对象、系统保存、接口查询时，BaseAdapter 会迅速膨胀。

### 4.2 难以表示不同资源能力
某些设备支持 NAT 创建但不支持更新，某些设备支持查询但不支持 API 修改。

### 4.3 难以处理 CLI 会话
像 Huawei SRG 这种设备通常不是“单方法执行单命令”，而是一个配置会话中执行多条命令。

### 4.4 不利于扩展更多厂商
后续如果接入更多设备，单一 Adapter 模型会越来越难维护。

因此建议升级为 Driver 设计。

---

## 5. Driver 顶层接口设计

每个 Driver 应至少具备以下能力：

- 获取设备能力声明
- 测试连通性
- 获取系统信息
- 获取资源服务对象
- 创建配置会话
- 保存配置（如果设备需要）

建议概念接口包括：

- `get_capabilities()`
- `test_connectivity()`
- `system_service`
- `nat_service`
- `policy_service`
- `open_config_session()`

---

## 6. 资源服务设计

### 6.1 NATService
负责 NAT / 端口映射相关操作。

建议支持的基本能力：

- list_rules()
- get_rule()
- create_rule()
- delete_rule()
- update_rule()
- validate_rule()

首期可以允许部分 Driver 不实现完整 update，而用 delete + create 替代。

---

### 6.2 PolicyService
用于后续扩展安全策略或 ACL。

建议未来支持：

- list_policies()
- create_policy()
- delete_policy()
- update_policy()

Huawei SRG 这类设备很可能需要 NAT 和策略联动，因此该服务需要预留。

---

### 6.3 SystemService
负责设备系统级操作。

建议未来支持：

- get_version()
- get_hostname()
- get_uptime()
- save_config()
- get_running_config()
- get_startup_config()

---

### 6.4 InterfaceService
负责设备接口与出口信息读取。

建议未来支持：

- list_interfaces()
- get_interface()
- list_wan_interfaces()

---

## 7. Transport 设计

### 7.1 BaseTransport
提供统一抽象，负责底层通信过程。

核心职责：

- connect()
- close()
- execute()
- execute_batch()

不同协议的细节由子类实现。

---

### 7.2 SSHTransport
适用于：

- OpenWrt
- 支持 SSH 的 Huawei 设备
- 其他 CLI 型网络设备

建议能力：

- 执行单条命令
- 执行命令批次
- 控制超时
- 捕获 stdout / stderr

---

### 7.3 HTTPTransport
适用于：

- pfSense API
- 后续其他基于 HTTP API 的设备

建议能力：

- get / post / put / delete
- session 管理
- token / basic auth 支持
- TLS 校验开关
- 重试控制

---

### 7.4 TelnetTransport
适用于：

- Huawei SRG 等老旧设备

这是必须纳入一等支持的 transport，而不是临时补丁。

TelnetTransport 必须考虑以下能力：

- 登录提示识别
- 用户名/密码输入
- prompt 检测
- 分页关闭
- 命令回显清洗
- read_until / expect 机制
- 超时与错误恢复

Telnet 的实现复杂度明显高于普通 SSH 命令执行，因此必须单独封装。

---

## 8. ConfigSession 设计

对于 CLI 型设备，建议引入配置会话概念。

配置会话的典型流程：

1. 建立连接
2. 进入配置模式
3. 执行命令批次
4. 提交或保存配置
5. 回读确认
6. 退出会话

建议保留以下抽象能力：

- enter()
- execute(command)
- execute_batch(commands)
- commit()
- save()
- rollback()（未来可选）
- exit()

不是所有设备都支持事务性回滚，但结构上应提前预留。

---

## 9. 命令模板与解析器

### 9.1 命令模板
所有 CLI 型设备命令必须集中管理。

例如 Huawei SRG 不应在业务逻辑中散落：

- nat server 命令
- undo nat server 命令
- policy 命令
- save 命令

而应集中放在：

- templates.py
或
- commands.py

这样便于：

- 统一维护语法
- 支持不同版本差异
- 做 dry-run
- 做单元测试

---

### 9.2 解析器 Parser
CLI 返回结果需要解析为结构化数据。

例如：

- NAT 规则列表
- 系统版本
- 接口状态
- 命令执行成功/失败

因此建议每类 Driver 配套 Parser，而不是在 service 层到处写字符串判断。

---

## 10. Driver 能力声明

每个 Driver 都应显式声明 capabilities，至少包括：

### 协议能力
- supports_ssh
- supports_telnet
- supports_http_api

### 资源能力
- supports_nat
- supports_policy
- supports_interface_query
- supports_system_info

### 动作能力
- supports_nat_create
- supports_nat_delete
- supports_nat_update
- supports_config_save
- supports_batch_commands

### 会话能力
- supports_config_session
- supports_transaction
- supports_dry_run

这样服务层可以根据能力决定：

- 是否展示某个按钮
- 是否允许某类操作
- 是否采用替代方案（如 delete + create）

---

## 11. Huawei SRG 特殊设计要求

Huawei SRG 是当前设计里最需要提前考虑扩展性的设备。

因为它可能具备以下特点：

- 只支持 Telnet 或 CLI
- 没有标准 API
- 命令语法复杂
- 配置操作常常需要多个步骤配合
- NAT 与策略可能联动
- 保存配置是显式动作

因此对 Huawei SRG 的 Driver 设计建议是：

- 独立目录
- 独立 Telnet/CLI 处理逻辑
- 独立模板模块
- 独立解析器
- 独立 SystemService / NATService / PolicyService

不建议把 Huawei 的 CLI 逻辑直接写在一个简单的 `huawei_srg.py` 文件里。

---

## 12. 推荐目录结构

建议从原来的 `adapters/` 调整为 `drivers/`：

```text
app/
├─ drivers/
│  ├─ base.py
│  ├─ openwrt/
│  │  ├─ driver.py
│  │  ├─ nat.py
│  │  ├─ system.py
│  │  └─ parser.py
│  ├─ pfsense/
│  │  ├─ driver.py
│  │  ├─ nat.py
│  │  ├─ system.py
│  │  └─ api.py
│  └─ huawei_srg/
│     ├─ driver.py
│     ├─ nat.py
│     ├─ policy.py
│     ├─ system.py
│     ├─ parser.py
│     └─ templates.py
├─ transports/
│  ├─ base.py
│  ├─ ssh.py
│  ├─ http.py
│  └─ telnet.py
```

---

## 13. 推荐演进路径

### 阶段 1
先实现：

- BaseTransport
- SSHTransport
- HTTPTransport
- BaseDriver
- OpenWrtDriver
- PfSenseDriver（基础能力）

### 阶段 2
补充：

- TelnetTransport
- HuaweiSRGDriver
- ConfigSession
- 命令模板与解析器

### 阶段 3
补充：

- PolicyService
- dry-run
- 批量命令执行
- 保存配置与回读校验
- 更细粒度能力声明

---

## 14. 总结

NATHub 在架构上不应被限制为单纯的 NAT 工具，而应设计成：

> **以 NAT / 端口映射为起点的多厂商网络设备 Driver 平台**

核心设计方向应为：

- Driver 替代窄接口 Adapter
- Resource Service 替代单一 NAT CRUD
- SSH / HTTP / Telnet 并列支持
- 配置会话与命令模板提前纳入结构
- 为后续策略、系统配置、批量操作保留扩展能力

这将显著降低后续在 Huawei SRG 等复杂 CLI 设备上的重构成本。
