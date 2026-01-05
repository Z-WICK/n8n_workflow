# Telegram 频道同步到 Notion - 项目文档

## 项目概述
自动将 Telegram 频道消息同步到 Notion Database，支持文字、图片（多图合并）、视频、文件等消息类型。

## 服务器信息
- **服务器 IP**: 23.19.230.202
- **SSH 端口**: 22
- **部署目录**: `/srv/n8n-compose/`

### 目录结构
```
/srv/n8n-compose/
├── docker-compose.yml          # Docker Compose 配置
├── .env                        # 环境变量配置
├── local-files/                # n8n 本地文件存储
└── data/                       # 所有持久化数据
    ├── n8n/                    # n8n 核心数据
    │   ├── database.sqlite     # n8n 数据库（工作流、凭证等）
    │   ├── config/             # n8n 配置
    │   ├── nodes/              # 自定义节点
    │   │   └── package.json
    │   ├── binaryData/         # 二进制数据存储
    │   ├── ssh/                # SSH 密钥
    │   ├── git/                # Git 配置
    │   ├── crash.journal       # 崩溃日志
    │   └── n8nEventLog*.log    # 事件日志
    ├── traefik/                # Traefik 反向代理数据
    │   └── acme.json           # SSL 证书（Let's Encrypt）
    └── redis/                  # Redis 缓存数据
        └── dump.rdb            # Redis 持久化文件
```

---

## n8n 配置
- **n8n 地址**: https://n8n.johnsion.com
- **n8n API Key**: `<YOUR_N8N_API_KEY>`
- **工作流 ID**: `mex4ERmqLLacDfol`
- **工作流名称**: Telegram频道同步到Notion
- **工作流状态**: 已激活

---

## Telegram Bot 配置
- **Bot Token**: `<YOUR_TELEGRAM_BOT_TOKEN>`
- **n8n 凭证 ID**: `NWqf2cts29NAnfeW`
- **n8n 凭证名称**: Telegram Bot - TG频道同步

### 监控的频道
- **频道名称**: 折腾啥
- **频道 ID**: `-1001248042100`

---

## Notion 配置
- **Integration Token**: `<YOUR_NOTION_TOKEN>`
- **Database ID**: `2de666823c4080ecb0d3f76688d78096`
- **Database URL**: https://www.notion.so/2de666823c4080ecb0d3f76688d78096

### Database 字段结构
| 字段名 | 类型 | 说明 |
|--------|------|------|
| 名称 | Title | 消息标题（前100字）|
| 来源频道 | Rich Text | Telegram 频道名称 |
| 发布时间 | Date | 消息发布时间 |
| 消息类型 | Select | 文字/图片/视频/文件 |
| 原文链接 | URL | Telegram 原文链接 |

---

## Redis 配置
- **n8n 凭证 ID**: `eBjJMqyqFcChRMQf`
- **Host**: redis (Docker 内部网络)
- **Port**: 6379
- **用途**: 多图消息合并缓存（TTL 30秒）

---

## 工作流功能说明

### 触发方式
- Telegram Webhook 监听 `channel_post` 事件

### 核心功能
1. **消息解析**: 提取文字、图片、视频、文件等内容
2. **多图合并**: 使用 Redis 缓存同一 media_group_id 的图片，等待 5 秒后合并写入
3. **图片嵌入**: 通过 Telegram getFile API 获取图片链接，嵌入 Notion 页面
4. **元数据记录**: 来源频道、发布时间、消息类型、原文链接

### 工作流节点
```
TelegramTrigger → ParseMsg → IsMediaGroup
                                ├─ Yes → RedisGet → MergeData → RedisSet → IsFirst
                                │                                            ├─ Yes → Wait(5s) → RedisFinal → PrepGroup → SplitFiles
                                │                                            └─ No → (结束)
                                └─ No → PrepSingle → SplitFiles
                                                          ↓
                                              GetFilePath → CollectUrls → BuildNotion → WriteNotion
```

---

## 迁移指南

### 备份
```bash
cd /srv/n8n-compose
tar -czvf n8n-backup-$(date +%Y%m%d).tar.gz data/ docker-compose.yml .env
```

### 恢复
```bash
# 解压备份
tar -xzvf n8n-backup-YYYYMMDD.tar.gz

# 修复权限
chown -R 1000:1000 data/n8n

# 启动服务
docker compose up -d
```

---

## 注意事项
1. Bot 需要是频道管理员才能接收消息
2. Notion Integration 需要连接到目标 Database
3. 图片通过 Telegram API 获取临时链接嵌入 Notion
4. 多图消息会等待 5 秒收集所有图片后再写入
5. Redis 数据 TTL 为 30 秒，用于多图消息去重

---

## 更新日志
- **2026-01-04**: 初始创建工作流
- **2026-01-04**: 添加多图消息合并功能（Redis）
- **2026-01-04**: 修复图片获取、元数据丢失等问题
- **2026-01-05**: 迁移 Docker volumes 到 bind mounts，便于备份迁移
