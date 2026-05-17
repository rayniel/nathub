# NATHub 技术选型

## 1. 语言选择

首选：Python

原因：

- 开发效率高
- 适合设备自动化、SSH、HTTP API 编排
- 适合快速构建 Web 管理后台
- 适合先做 MVP 再逐步扩展

---

## 2. 后端框架

推荐：Flask

理由：

- 轻量
- 容易组织后台管理系统
- 配合 Jinja2 适合服务端渲染
- 插件生态成熟

建议组件：

- Flask
- Flask-Login
- Flask-WTF
- SQLAlchemy
- Alembic

---

## 3. 设备通信

### SSH
推荐：

- Paramiko

可选：

- Netmiko（后期如 CLI 交互复杂可引入）

### HTTP/API
推荐：

- httpx
或
- requests

---

## 4. 数据库

### MVP
- SQLite

### 生产建议
- PostgreSQL
或
- MariaDB

---

## 5. 前端

推荐：

- Jinja2
- AdminLTE

原则：

- 以后台表单和表格为主
- 尽量不做重型 SPA
- 页面优先服务端渲染

---

## 6. 后台任务

MVP 推荐：

- 数据库任务表
- Python 后台线程池

后期可选：

- Celery + Redis

---

## 7. 加密与安全

推荐：

- cryptography
- Fernet 对称加密保存设备凭据

---

## 8. 部署环境

推荐：

- Debian 12
- Ubuntu 24.04 LTS

运行方式：

- systemd
- gunicorn / waitress
- nginx 反向代理（可选）
