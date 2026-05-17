# NATHub MVP 规划

## 阶段 1：基础框架

目标：

- Flask 项目初始化
- 登录页
- 主页仪表盘
- SQLite 数据库初始化
- 设备管理页面
- NAT 规则页面基础结构

交付：

- 基础项目骨架
- README
- 文档目录
- 数据库迁移

---

## 阶段 2：OpenWrt 接入

目标：

- 实现 SSHTransport
- 实现 OpenWrtAdapter
- 支持：
  - 测试连接
  - 查询 NAT 规则
  - 创建 NAT 规则
  - 删除 NAT 规则

交付：

- OpenWrt 首个完整闭环

---

## 阶段 3：任务和日志

目标：

- 增加任务表 jobs
- 增加任务日志
- 页面显示任务执行结果
- 增加审计日志

---

## 阶段 4：pfSense 接入

目标：

- 实现 PfSenseAdapter
- 支持基础规则查询/创建/删除

---

## 阶段 5：Huawei SRG 接入

目标：

- 实现 HuaweiSRGAdapter
- CLI 模板化命令下发
- 基础回读校验

---

## 阶段 6：增强功能

目标：

- 规则同步
- 冲突检测优化
- 批量下发
- 配置快照
- 差异比对
- 回滚支持
