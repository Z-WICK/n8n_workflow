# n8n Workflows Collection

本仓库收集了基于 [n8n](https://n8n.io/) 自动化平台构建的工作流，用于日常效率提升和数据同步。

## 目录结构

```
n8n_workflow/
├── feishu-accounting/          # 飞书记账工作流
│   ├── feishu-accounting-workflow.json
│   ├── feishu-accounting-workflow-v2.json
│   ├── feishu-token-refresh.json
│   └── README.md
└── telegram-notion-sync/       # Telegram 同步 Notion 工作流
    ├── telegram-notion-sync-guide.md
    └── telegram-notion-sync-credentials.md
```

---

## 工作流列表

### 1. Telegram 频道同步到 Notion

**目录**: `telegram-notion-sync/`

自动将 Telegram 频道/私聊消息同步到 Notion 数据库。

#### 功能特性

| 功能 | 说明 |
|------|------|
| 实时同步 | Telegram 频道新消息自动同步 |
| 私聊支持 | 支持私聊机器人转发消息 |
| 图片嵌入 | 图片直接嵌入 Notion 页面 |
| 多图合并 | 同一消息多张图片合并为一条记录 |
| 链接保留 | 保留超链接、粗体、斜体等格式 |
| 转发消息 | 自动提取转发消息的原始来源 |

#### 技术栈

- **n8n** - 工作流引擎
- **Redis** - 多图消息缓存
- **Telegram Bot API** - 消息接收
- **Notion API** - 数据写入

#### 文档

- [完整部署指南](telegram-notion-sync/telegram-notion-sync-guide.md)
- [凭证配置说明](telegram-notion-sync/telegram-notion-sync-credentials.md)

---

### 2. 飞书记账工作流

**目录**: `feishu-accounting/`

通过飞书机器人实现快捷记账功能。

#### 功能特性

- 飞书消息触发记账
- 自动解析金额和分类
- 数据存储到飞书多维表格
- Token 自动刷新

#### 文档

- [工作流说明](feishu-accounting/README.md)

---

## 快速开始

### 环境要求

- Docker & Docker Compose
- n8n 实例 (自托管或云端)
- 相关平台的 API 凭证

### 部署步骤

1. **克隆仓库**
   ```bash
   git clone https://github.com/Z-WICK/n8n_workflow.git
   ```

2. **导入工作流**
   - 登录 n8n 控制台
   - 点击 `Import from File`
   - 选择对应的 `.json` 文件

3. **配置凭证**
   - 根据文档配置各平台的 API Token
   - 测试连接确保凭证有效

4. **激活工作流**
   - 开启工作流开关
   - 验证 Webhook 是否正常

---

## 自托管 n8n

推荐使用 Docker Compose 部署：

```yaml
version: '3.8'
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    ports:
      - "5678:5678"
    volumes:
      - ./data/n8n:/home/node/.n8n
    environment:
      - N8N_HOST=your-domain.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://your-domain.com/
```

详细部署配置请参考各工作流的文档。

---

## 备份与恢复

### 备份

```bash
# 备份 n8n 数据目录
tar -czvf n8n-backup-$(date +%Y%m%d).tar.gz data/
```

### 恢复

```bash
# 解压备份
tar -xzvf n8n-backup-YYYYMMDD.tar.gz

# 修复权限
chown -R 1000:1000 data/n8n

# 重启服务
docker compose restart n8n
```

---

## 安全提示

- **不要**将 API Token、Bot Token 等敏感信息提交到仓库
- 使用环境变量或 n8n 内置凭证管理存储敏感数据
- 定期轮换 API 密钥

---

## License

MIT

---

## 相关链接

- [n8n 官方文档](https://docs.n8n.io/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Notion API](https://developers.notion.com/)
- [飞书开放平台](https://open.feishu.cn/)
