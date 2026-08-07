---
title: "Python 环境变量：作用域与存储"
date: 2026-05-11
description: Python 环境变量的作用域规则、os.environ 操作与 .env 文件管理的完整指南
tags:
  - Python
  - 环境变量
  - 配置
---

# Python 环境变量：作用域与存储

> **最后更新**: 2026-05-11

---

## 一、环境变量的本质

环境变量是操作系统层面的键值对，每个进程都有一份从父进程继承的环境副本。Python 通过 `os.environ` 访问和修改当前进程的环境变量。

**关键特性**：

- 进程级隔离：子进程继承父进程的环境变量快照，修改互不影响
- 字符串类型：所有值都是字符串，需要手动转换类型（int/bool/path 等）
- 生命周期：随进程存在，进程退出即消失

---

## 二、os.environ 操作

### 2.1 读取

```python
import os

# 读取（不存在则 KeyError）
value = os.environ["HOME"]

# 安全读取（不存在返回 None）
value = os.environ.get("HOME")

# 带默认值
value = os.environ.get("PORT", "8080")

# 批量获取
env = {k: os.environ[k] for k in ["HOME", "PATH", "SHELL"] if k in os.environ}
```

### 2.2 写入

```python
# 设置
os.environ["MY_VAR"] = "hello"

# 修改后子进程可见
os.environ["PATH"] = "/usr/local/bin:" + os.environ["PATH"]
os.system("echo $MY_VAR")  # hello
```

**注意**：`os.environ` 的修改只影响当前进程及其子进程，不会回写到父进程的 shell 环境。

### 2.3 删除

```python
# 删除单个变量
del os.environ["MY_VAR"]

# 安全删除（不存在不报错）
os.environ.pop("MY_VAR", None)
```

---

## 三、作用域层级

环境变量的优先级从低到高：

```
系统级 < 用户级 < Shell 配置 < 终端会话 < 进程继承 < 代码中设置
```

| 层级 | 位置 | 影响范围 | 持久性 |
|------|------|---------|--------|
| 系统级 | `/etc/environment`、`/etc/profile` | 所有用户 | 永久 |
| 用户级 | `~/.bashrc`、`~/.zshrc`、`~/.profile` | 当前用户 | 永久 |
| 会话级 | `export VAR=value` | 当前终端会话 | 临时 |
| 进程级 | Python `os.environ["VAR"]` | 当前进程及子进程 | 临时 |
| .env 文件 | 项目目录 | 加载到的进程 | 持久（文件） |

### .env 文件加载

```bash
# .env 文件示例
DATABASE_URL=postgresql://localhost/mydb
SECRET_KEY=abc123
DEBUG=true
```

```python
# 方式一：python-dotenv
from dotenv import load_dotenv
load_dotenv()  # 默认加载当前目录的 .env
db_url = os.environ["DATABASE_URL"]

# 方式二：指定路径
from dotenv import load_dotenv
load_dotenv("/path/to/.env")

# 方式三：不覆盖已有变量（生产环境已有环境变量时）
load_dotenv(override=False)
```

> **安全提醒**：`.env` 文件应加入 `.gitignore`，不要提交到版本控制。可以提交 `.env.example` 作为模板。

---

## 四、常见数据类型转换

环境变量的值始终是字符串，需要手动转换：

```python
import os

# 布尔值
debug = os.environ.get("DEBUG", "false").lower() in ("true", "1", "yes")

# 整数
port = int(os.environ.get("PORT", "8080"))

# 列表（逗号分隔）
allowed_hosts = os.environ.get("ALLOWED_HOSTS", "").split(",") if os.environ.get("ALLOWED_HOSTS") else []

# JSON
import json
config = json.loads(os.environ.get("CONFIG", "{}"))

# 路径
from pathlib import Path
data_dir = Path(os.environ.get("DATA_DIR", "/tmp/data"))
```

---

## 五、子进程的环境继承

子进程继承父进程 `os.environ` 的快照：

```python
import subprocess
import os

# 修改当前进程环境
os.environ["MY_VAR"] = "parent_value"

# 子进程继承修改后的环境
result = subprocess.run(["echo", "$MY_VAR"], capture_output=True, text=True)
# 但这样无效：shell 变量替换由 shell 做，不是子进程
# 正确方式：
result = subprocess.run(["env"], capture_output=True, text=True)
# 输出中会包含 MY_VAR=parent_value

# 额外传入环境变量（不影响 os.environ）
result = subprocess.run(
    ["env"],
    env={**os.environ, "EXTRA_VAR": "extra"},
    capture_output=True, text=True
)
```

---

## 六、最佳实践

1. **所有配置通过环境变量注入**：不硬编码敏感信息，遵循 12-Factor App 原则
2. **提供合理默认值**：`os.environ.get("KEY", default)` 避免部署时遗漏配置
3. **启动时集中读取**：在应用入口统一解析，而非分散在各模块
4. **类型安全**：使用 Pydantic 等库进行校验和类型转换
5. **区分环境**：通过 `APP_ENV` 等变量区分 dev/staging/prod

```python
# 推荐：Pydantic Settings 集中管理
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "postgresql://localhost/dev"
    secret_key: str
    debug: bool = False
    port: int = 8080

    class Config:
        env_file = ".env"

settings = Settings()  # 自动从环境变量和 .env 加载并校验类型
```

---

*最后更新: 2026-05-11*
