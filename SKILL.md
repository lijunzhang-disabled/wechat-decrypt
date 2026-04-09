---
name: wechat-decrypt
description: 微信聊天记录解密与查询 - 解密本地微信数据库，查询和总结聊天记录。
---

# WeChat Decrypt - 微信聊天记录解密与查询

解密 macOS 微信 4.0 的本地数据库，支持查询聊天记录、联系人信息，并生成总结。

## 项目位置

```
/Users/lijunzhang/secagentinfra/wechat-decrypt
```

## 前置条件

- macOS 微信 4.0 客户端已登录
- conda 环境 `crypto` 已配置（包含 pycryptodome 等依赖）

## 触发场景

用户提到以下关键词时触发：
- "微信聊天记录"、"聊天记录"、"微信消息"
- "解密微信"、"wechat decrypt"
- 查询特定日期/联系人/群聊的消息
- "总结聊天"、"整理消息"

## 核心操作流程

### 第 1 步：解密数据库（如果需要更新）

```bash
cd /Users/lijunzhang/secagentinfra/wechat-decrypt
source ~/.zshrc; conda activate crypto; python decrypt_db.py
```

解密后的数据库位于：`/Users/lijunzhang/secagentinfra/wechat-decrypt/decrypted/`

### 第 2 步：检查数据库时间范围

```bash
source ~/.zshrc; conda activate crypto; python -c "
import sqlite3
import time

conn = sqlite3.connect('/Users/lijunzhang/secagentinfra/wechat-decrypt/decrypted/message/message_0.db')
cursor = conn.cursor()
cursor.execute(\"SELECT name FROM sqlite_master WHERE type='table' AND name LIKE 'Msg_%'\")
tables = [t[0] for t in cursor.fetchall()]

max_time = 0
for table in tables:
    try:
        cursor.execute(f'SELECT MAX(create_time) FROM {table}')
        r = cursor.fetchone()
        if r and r[0] and r[0] > max_time:
            max_time = r[0]
    except:
        pass

print(f'数据库最新消息时间: {time.strftime(\"%Y-%m-%d %H:%M:%S\", time.localtime(max_time))}')
print(f'当前时间: {time.strftime(\"%Y-%m-%d %H:%M:%S\", time.localtime())}')
conn.close()
"
```

如果数据库最新时间落后当前时间超过1小时，建议重新运行解密脚本。

### 第 3 步：查询聊天记录

#### 按日期查询所有消息

```python
import sqlite3
import time

conn = sqlite3.connect('/Users/lijunzhang/secagentinfra/wechat-decrypt/decrypted/message/message_0.db')
cursor = conn.cursor()

cursor.execute("SELECT name FROM sqlite_master WHERE type='table' AND name LIKE 'Msg_%'")
tables = [t[0] for t in cursor.fetchall()]

# 设置时间范围（时间戳）
start_time = 1775577600  # 示例：2026-04-08 00:00:00
end_time = 1775663999    # 示例：2026-04-08 23:59:59

all_msgs = []
for table in tables:
    try:
        cursor.execute(f'SELECT create_time, message_content FROM {table} WHERE create_time >= ? AND create_time <= ?', (start_time, end_time))
        for r in cursor.fetchall():
            if r[1] and isinstance(r[1], str):
                all_msgs.append((r[0], r[1]))
    except:
        pass

all_msgs.sort(key=lambda x: x[0])
for ts, content in all_msgs:
    t = time.strftime('%H:%M:%S', time.localtime(ts))
    print(f'[{t}] {content[:200]}')

conn.close()
```

#### 查询特定群聊/联系人

1. 先从联系人数据库查找 username：

```python
import sqlite3

conn = sqlite3.connect('/Users/lijunzhang/secagentinfra/wechat-decrypt/decrypted/contact/contact.db')
cursor = conn.cursor()

# 按昵称或备注搜索
cursor.execute("SELECT username, nick_name, remark FROM contact WHERE nick_name LIKE '%关键词%' OR remark LIKE '%关键词%'")
for r in cursor.fetchall():
    print(r)

conn.close()
```

2. 根据 username 计算消息表名：

```python
import hashlib
username = '20760632071@chatroom'  # 群聊ID
table_hash = hashlib.md5(username.encode()).hexdigest()
table_name = f'Msg_{table_hash}'
print(f'消息表名: {table_name}')
```

3. 查询该表的消息。

## 数据库结构

### 解密后目录结构

```
decrypted/
├── contact/
│   ├── contact.db      # 联系人信息
│   └── contact_fts.db  # 联系人全文索引
├── message/
│   ├── message_0.db    # 主消息数据库
│   ├── biz_message_0.db # 公众号消息
│   └── media_0.db      # 媒体信息
├── session/
│   └── session.db      # 会话列表
└── ...
```

### 联系人表 (contact.db - contact)

| 字段 | 说明 |
|------|------|
| username | 微信ID（wxid_xxx 或 群聊ID@chatroom） |
| nick_name | 昵称 |
| remark | 备注名 |
| alias | 微信号 |

### 消息表 (message_0.db - Msg_xxx)

每个聊天会话对应一个表，表名为 `Msg_` + MD5(username)

| 字段 | 说明 |
|------|------|
| create_time | 消息时间戳（秒） |
| message_content | 消息内容（文本/XML） |
| local_type | 消息类型 |

## 常用群聊/联系人速查

根据历史记录，常用的群聊：
- **闪闪红星**: `20760632071@chatroom` → 表名 `Msg_352bf021a7d6b0629fe797bc88e661ed`

## 注意事项

1. **必须使用 conda crypto 环境**：`source ~/.zshrc; conda activate crypto; python ...`
2. **数据库可能被锁定**：如果微信正在运行，某些表可能被锁
3. **时间戳是秒级**：Python `time.time()` 返回的就是秒级时间戳
4. **消息内容可能是 bytes**：查询时过滤 `isinstance(r[1], str)`

## MCP Server（可选）

项目包含 `mcp_server.py`，可以通过 MCP 协议提供查询接口：
- `mcp__wechat__get_contacts` - 获取联系人
- `mcp__wechat__get_chat_history` - 获取聊天记录

配置已在 `.claude/settings.local.json` 中预设。
