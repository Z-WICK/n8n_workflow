# Telegram 频道同步到 Notion 工作流指南

## 目录

- [概述](#概述)
- [功能特性](#功能特性)
- [架构设计](#架构设计)
- [环境要求](#环境要求)
- [部署配置](#部署配置)
- [工作流详解](#工作流详解)
- [Notion 数据库结构](#notion-数据库结构)
- [常见问题](#常见问题)
- [维护与备份](#维护与备份)

---

## 概述

本工作流实现了 **Telegram 频道/私聊消息自动同步到 Notion 数据库** 的功能。支持文字、图片、视频等多种消息类型，并能保留消息中的超链接格式。

### 工作流信息

| 项目 | 值 |
|------|-----|
| 工作流 ID | `mex4ERmqLLacDfol` |
| 工作流名称 | Telegram频道同步到Notion |
| n8n 地址 | https://n8n.johnsion.com |
| 状态 | 已激活 |

---

## 功能特性

### 核心功能

- ✅ **实时同步** - Telegram 频道新消息自动同步到 Notion
- ✅ **私聊支持** - 支持通过私聊机器人转发消息同步
- ✅ **图片嵌入** - 图片直接嵌入 Notion 页面
- ✅ **多图合并** - 同一条消息的多张图片合并为一条 Notion 记录
- ✅ **链接保留** - 保留消息中的超链接、粗体、斜体等格式
- ✅ **转发消息** - 支持转发的消息，自动提取原始来源

### 支持的消息类型

| 类型 | 支持状态 | 说明 |
|------|---------|------|
| 文字消息 | ✅ | 完整保留格式和链接 |
| 图片消息 | ✅ | 嵌入 Notion 页面 |
| 视频消息 | ✅ | 记录视频信息 |
| 文件消息 | ✅ | 记录文件信息 |
| 多图消息 | ✅ | 自动合并为单条记录 |

### 支持的文本格式

| 格式 | Telegram | Notion |
|------|----------|--------|
| 超链接 | `[文字](URL)` | 可点击链接 |
| 粗体 | `**文字**` | **粗体** |
| 斜体 | `_文字_` | *斜体* |
| 代码 | `` `代码` `` | `代码` |
| URL | 自动识别 | 可点击链接 |

---

## 架构设计

### 整体架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Telegram      │     │      n8n        │     │     Notion      │
│   频道/私聊     │────▶│   工作流处理    │────▶│    数据库       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌─────────────────┐
                        │     Redis       │
                        │  (多图缓存)     │
                        └─────────────────┘
```

### 工作流节点流程

```
ChannelTrigger (Telegram Webhook)
        │
        ▼
    ParseMsg (解析消息、提取实体)
        │
        ▼
   IsMediaGroup? ─────────────────────┐
        │ Yes                         │ No
        ▼                             ▼
    RedisGet                      PrepSingle
        │                             │
        ▼                             │
    MergeData                         │
        │                             │
        ▼                             │
    RedisSet                          │
        │                             │
        ▼                             │
    IsFirst? ─────┐                   │
        │ Yes     │ No (丢弃)         │
        ▼         │                   │
      Wait (5s)   │                   │
        │         │                   │
        ▼         │                   │
    RedisFinal    │                   │
        │         │                   │
        ▼         │                   │
    PrepGroup     │                   │
        │         │                   │
        ▼◀────────┴───────────────────┘
    SplitFiles
        │
        ▼
    GetFilePath (获取图片URL)
        │
        ▼
    CollectUrls
        │
        ▼
    BuildNotion (构建Notion页面)
        │
        ▼
    WriteNotion (写入Notion)
```

---

## 环境要求

### 服务器配置

| 组件 | 版本/配置 |
|------|----------|
| Docker | 20.10+ |
| Docker Compose | v2+ |
| 内存 | 最低 1GB |
| 存储 | 最低 1GB |

### 服务组件

| 服务 | 用途 | 端口 |
|------|------|------|
| n8n | 工作流引擎 | 5678 (内部) |
| Redis | 多图消息缓存 | 6379 (内部) |
| Traefik | 反向代理 & SSL | 80, 443 |

---

## 部署配置

### 目录结构

```
/srv/n8n-compose/
├── docker-compose.yml      # Docker Compose 配置
└── data/
    ├── n8n/                # n8n 数据 (工作流、凭证、执行记录)
    ├── redis/              # Redis 数据
    └── traefik/            # Traefik 配置和 SSL 证书
```

### Docker Compose 配置

```yaml
version: '3.8'

services:
  traefik:
    image: traefik
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=your-email@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik:/letsencrypt

  n8n:
    image: docker.n8n.io/n8nio/n8n
    environment:
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.yourdomain.com/
    volumes:
      - ./data/n8n:/home/node/.n8n
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`n8n.yourdomain.com`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"

  redis:
    image: redis:alpine
    volumes:
      - ./data/redis:/data
```

### 凭证配置

#### 1. Telegram Bot 凭证

| 字段 | 值 |
|------|-----|
| 凭证名称 | Telegram Bot - TG频道同步 |
| 凭证 ID | `NWqf2cts29NAnfeW` |
| Bot Token | `<YOUR_TELEGRAM_BOT_TOKEN>` |

**创建 Bot 步骤：**

1. 在 Telegram 中搜索 `@BotFather`
2. 发送 `/newbot` 创建新机器人
3. 按提示设置名称和用户名
4. 获取 Bot Token
5. 将 Bot 添加为频道管理员

#### 2. Redis 凭证

| 字段 | 值 |
|------|-----|
| 凭证名称 | Redis |
| 凭证 ID | `eBjJMqyqFcChRMQf` |
| Host | `redis` |
| Port | `6379` |

#### 3. Notion 配置

| 字段 | 值 |
|------|-----|
| API Token | `<YOUR_NOTION_TOKEN>` |
| Database ID | `2de666823c4080ecb0d3f76688d78096` |

**获取 Notion Token：**

1. 访问 https://www.notion.so/my-integrations
2. 创建新的 Integration
3. 复制 Internal Integration Token
4. 在 Notion 数据库页面，点击 `...` → `Add connections` → 选择你的 Integration

---

## 工作流详解

### 节点说明

#### 1. ChannelTrigger

**类型：** Telegram Trigger

**功能：** 接收 Telegram Webhook 消息

**配置：**
```json
{
  "updates": ["channel_post", "message"],
  "additionalFields": {}
}
```

**监听事件：**
- `channel_post` - 频道新消息
- `message` - 私聊消息

---

#### 2. ParseMsg

**类型：** Code 节点

**功能：** 解析 Telegram 消息，提取文本、实体（链接、格式）、媒体信息

**核心逻辑：**

```javascript
// 解析消息实体，保留链接格式
function parseEntities(text, entities) {
  // 处理 text_link (超链接)
  // 处理 url (纯URL)
  // 处理 bold, italic, code (格式)
}
```

**输出字段：**

| 字段 | 说明 |
|------|------|
| `text` | 纯文本内容 |
| `richText` | Notion 格式的富文本 JSON |
| `isoDate` | ISO 格式时间戳 |
| `chatTitle` | 来源频道/聊天名称 |
| `mediaType` | 消息类型 (文字/图片/视频/文件) |
| `fileId` | Telegram 文件 ID |
| `sourceUrl` | 原消息链接 |
| `mediaGroupId` | 多图消息组 ID |

---

#### 3. IsMediaGroup

**类型：** IF 节点

**功能：** 判断是否为多图消息

**条件：** `hasMediaGroup === true`

---

#### 4. RedisGet / RedisSet / RedisFinal

**类型：** Redis 节点

**功能：** 缓存多图消息数据

**Key 格式：** `mg:{mediaGroupId}`

**TTL：** 30 秒

---

#### 5. MergeData

**类型：** Code 节点

**功能：** 合并同一组多图消息的数据

**逻辑：**
- 首条消息：创建新缓存
- 后续消息：追加图片到缓存

---

#### 6. IsFirst

**类型：** IF 节点

**功能：** 判断是否为多图组的第一条消息

**逻辑：**
- 是第一条 → 继续等待其他图片
- 不是第一条 → 丢弃（由第一条统一处理）

---

#### 7. Wait

**类型：** Wait 节点

**功能：** 等待 5 秒，让所有图片消息到达

---

#### 8. PrepGroup / PrepSingle

**类型：** Code 节点

**功能：** 准备数据供后续处理

---

#### 9. SplitFiles

**类型：** Code 节点

**功能：** 将多个文件拆分为独立项，逐个获取下载链接

---

#### 10. GetFilePath

**类型：** HTTP Request 节点

**功能：** 调用 Telegram API 获取文件路径

**API：** `https://api.telegram.org/bot{token}/getFile?file_id={fileId}`

---

#### 11. CollectUrls

**类型：** Code 节点

**功能：** 收集所有文件的下载 URL

**URL 格式：** `https://api.telegram.org/file/bot{token}/{file_path}`

---

#### 12. BuildNotion

**类型：** Code 节点

**功能：** 构建 Notion API 请求体

**核心逻辑：**
- 标题：取消息第一行
- 正文：第一行之后的内容（保留链接格式）
- 图片：嵌入为 Image Block

---

#### 13. WriteNotion

**类型：** HTTP Request 节点

**功能：** 调用 Notion API 创建页面

**API：** `POST https://api.notion.com/v1/pages`

---

## Notion 数据库结构

### 必需字段

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 名称 | Title | 消息第一行（标题） |
| 来源频道 | Rich Text | 频道/聊天名称 |
| 发布时间 | Date | 消息发送时间 |
| 消息类型 | Select | 文字/图片/视频/文件 |
| 原文链接 | URL | Telegram 原消息链接 |

### 创建数据库

1. 在 Notion 中创建新数据库
2. 添加上述字段
3. 复制数据库 ID（URL 中 `?v=` 之前的 32 位字符串）
4. 将 Integration 连接到数据库

---

## 常见问题

### Q1: Webhook 返回 404

**原因：** 节点名称包含中文，URL 编码后无法匹配

**解决：** 确保触发器节点名称为纯英文（如 `ChannelTrigger`）

---

### Q2: Webhook 返回 403

**原因：** 工作流未激活或 webhook 未注册

**解决：**
1. 重新激活工作流
2. 重启 n8n 服务
3. 检查 Telegram webhook 设置

---

### Q3: 多图消息生成多条记录

**原因：** Redis 未正确配置或连接失败

**解决：**
1. 检查 Redis 服务是否运行
2. 检查 Redis 凭证配置
3. 查看 n8n 执行日志

---

### Q4: 链接丢失

**原因：** 消息实体未正确解析

**解决：** 检查 `ParseMsg` 节点的 `entities` 解析逻辑

---

### Q5: Too Many Requests 错误

**原因：** Telegram API 频率限制

**解决：** 等待几秒后重试，避免频繁操作

---

## 维护与备份

### 数据目录

```
/srv/n8n-compose/data/
├── n8n/      (24MB) - 工作流、凭证、执行记录
├── redis/    (8KB)  - 缓存数据
└── traefik/  (8KB)  - SSL 证书
```

### 备份命令

```bash
# 创建备份
cd /srv/n8n-compose
tar -czvf n8n-backup-$(date +%Y%m%d).tar.gz data/

# 恢复备份
tar -xzvf n8n-backup-20260105.tar.gz
```

### 定时备份 (Cron)

```bash
# 每天凌晨 3 点备份
0 3 * * * cd /srv/n8n-compose && tar -czvf /backup/n8n-$(date +\%Y\%m\%d).tar.gz data/
```

### 日志查看

```bash
# 查看 n8n 日志
docker logs n8n-compose-n8n-1 --tail 100

# 查看 Redis 日志
docker logs n8n-compose-redis-1 --tail 100

# 实时跟踪日志
docker logs -f n8n-compose-n8n-1
```

### 服务管理

```bash
# 启动服务
cd /srv/n8n-compose && docker compose up -d

# 停止服务
cd /srv/n8n-compose && docker compose down

# 重启服务
cd /srv/n8n-compose && docker compose restart

# 重启单个服务
cd /srv/n8n-compose && docker compose restart n8n
```

---

## 附录

### Telegram Bot API 参考

- [getWebhookInfo](https://core.telegram.org/bots/api#getwebhookinfo) - 获取 webhook 信息
- [setWebhook](https://core.telegram.org/bots/api#setwebhook) - 设置 webhook
- [getFile](https://core.telegram.org/bots/api#getfile) - 获取文件下载路径

### Notion API 参考

- [Create a page](https://developers.notion.com/reference/post-page) - 创建页面
- [Rich text](https://developers.notion.com/reference/rich-text) - 富文本格式

### 相关链接

- [n8n 官方文档](https://docs.n8n.io/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Notion API](https://developers.notion.com/)

---

*文档更新时间：2026-01-05*
