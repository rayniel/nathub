# NATHub

NATHub 是一个运行在 x86/Linux 控制器上的统一 NAT / 端口映射管理平台，用于集中管理多设备、多出口场景下的端口映射规则。

当前目标设备类型：

- OpenWrt
- pfSense
- Huawei SRG2210

NATHub 的核心目标：

- 统一管理多厂商设备上的端口映射规则
- 支持多设备、多 WAN / 多出口
- 统一抽象 NAT 规则模型
- 提供统一的 Web 管理界面和 API
- 通过三层路由直连设备，无需在 MT7621 等边界路由器上部署 agent
- 保留审计、任务执行、规则同步、冲突检测等扩展能力

## 架构原则

- **单控制器架构**：所有业务逻辑集中在 x86 控制器
- **网络直连**：通过三层路由访问目标设备管理地址
- **边界设备无代理**：MT7621 只做路由/转发，不运行管理程序
- **统一规则模型**：以平台数据模型为中心，向各厂商设备适配
- **可扩展适配器**：OpenWrt / pfSense / Huawei SRG 独立适配器实现
- **尽量轻量**：首期使用 Flask + SQLAlchemy + Jinja2 + AdminLTE

## 推荐技术栈

### 后端

- Python 3.11+
- Flask
- SQLAlchemy
- Alembic
- Flask-Login
- WTForms / Flask-WTF

### 设备通信

- Paramiko（SSH）
- requests 或 httpx（HTTP/API）

### 数据库

- MVP：SQLite
- 正式部署：PostgreSQL / MariaDB

### 前端

- Jinja2 模板渲染
- AdminLTE 作为后台 UI 模板
- 尽量避免前端重型 SPA

## 典型部署位置

推荐部署在：

- PVE 虚拟机（Debian / Ubuntu）
- 或独立 x86 Linux 主机

不推荐部署在：

- MT7621/OpenWrt 本机

原因：

- 资源限制
- Python 运行时和依赖维护成本高
- 多设备适配逻辑更适合集中部署

## 目标网络模型

- x86 控制器通过三层路由访问各设备管理网段
- MT7621 仅提供路由可达性
- 必要时可在边界侧使用静态路由或 SNAT 解决回程路径问题

## 核心能力规划

- 设备管理
- 多出口管理
- NAT / 端口映射规则管理
- 规则冲突检测
- 统一任务下发
- 执行日志记录
- 审计日志
- 后续支持配置快照、规则对比、回滚

## 开发文档

详细设计文档见 `docs/` 目录。
