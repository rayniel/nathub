# NATHub API 设计草案

## 1. 设备管理

### 获取设备列表
- GET /api/devices

### 创建设备
- POST /api/devices

### 更新设备
- PUT /api/devices/{id}

### 删除设备
- DELETE /api/devices/{id}

### 测试连接
- POST /api/devices/{id}/test-connectivity

---

## 2. 出口管理

### 获取设备出口
- GET /api/devices/{id}/wan-interfaces

### 新增出口
- POST /api/devices/{id}/wan-interfaces

### 更新出口
- PUT /api/wan-interfaces/{id}

### 删除出口
- DELETE /api/wan-interfaces/{id}

---

## 3. NAT 规则管理

### 获取规则列表
- GET /api/nat-rules

支持过滤：

- device_id
- wan_interface_id
- protocol
- enabled

### 获取单条规则
- GET /api/nat-rules/{id}

### 创建规则
- POST /api/nat-rules

### 更新规则
- PUT /api/nat-rules/{id}

### 删除规则
- DELETE /api/nat-rules/{id}

### 启用规则
- POST /api/nat-rules/{id}/enable

### 禁用规则
- POST /api/nat-rules/{id}/disable

### 手动同步规则
- POST /api/nat-rules/{id}/sync

---

## 4. 任务接口

### 获取任务列表
- GET /api/jobs

### 获取任务详情
- GET /api/jobs/{id}

### 获取任务日志
- GET /api/jobs/{id}/logs

---

## 5. 审计接口

### 获取审计日志
- GET /api/audit-logs
