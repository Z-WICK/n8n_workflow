# 飞书记账工作流使用说明

## 工作流架构

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│   Webhook   │ ──▶ │  解析数据   │ ──▶ │ 写入飞书多维表格 │ ──▶ │  返回响应   │
│  POST请求   │     │  验证格式   │     │    创建记录     │     │  成功/失败  │
└─────────────┘     └─────────────┘     └─────────────────┘     └─────────────┘
```

## 文件说明

| 文件 | 说明 |
|------|------|
| `feishu-accounting-workflow.json` | 主工作流，接收记账数据并写入飞书 |
| `feishu-token-refresh.json` | Token 刷新工作流，每小时自动刷新 |

---

## 配置步骤

### 1. 创建飞书应用

1. 访问 [飞书开放平台](https://open.feishu.cn/app)
2. 创建企业自建应用
3. 获取 `App ID` 和 `App Secret`
4. 添加权限：
   - `bitable:app` - 多维表格应用权限
   - `bitable:app:readonly` - 读取权限
   - `bitable:app:write` - 写入权限

### 2. 创建飞书多维表格

在飞书中创建多维表格，包含以下字段：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 日期 | 日期 | 记账日期 |
| 金额 | 数字 | 金额（正数收入，负数支出） |
| 类型 | 单选 | 收入/支出 |
| 类别 | 单选 | 餐饮/交通/购物/娱乐/其他 |
| 备注 | 文本 | 备注信息 |
| 创建时间 | 日期时间 | 记录创建时间 |

### 3. 获取表格信息

从多维表格 URL 中获取：
```
https://xxx.feishu.cn/base/{APP_TOKEN}?table={TABLE_ID}
```

- `APP_TOKEN`: 多维表格的应用 Token
- `TABLE_ID`: 数据表 ID

### 4. 配置 n8n 环境变量

在 n8n 中设置以下环境变量：

```bash
FEISHU_BASE_URL=https://open.feishu.cn
FEISHU_APP_ID=cli_a9c4cc39bd385cda
FEISHU_APP_SECRET=yW4bQAPHweSae98KnfmkydzEdh1LXgg0
FEISHU_APP_TOKEN=RqfMbH3Pna5z8YsO82Ucszp4n0d
FEISHU_TABLE_ID=tblLa5dU4grEaNiE
```

### 5. 配置 HTTP Header Auth

在 n8n 凭证中创建 `HTTP Header Auth`：
- Name: `Feishu Token`
- Header Name: `Authorization`
- Header Value: `Bearer {your_tenant_access_token}`

### 6. 导入工作流

1. 在 n8n 中点击 "Import from File"
2. 选择 `feishu-accounting-workflow.json`
3. 同样导入 `feishu-token-refresh.json`
4. 激活两个工作流

---

## API 使用方法

### Webhook 地址

```
POST https://your-n8n-domain/webhook/accounting
```

### 请求格式

```json
{
  "amount": -35.5,
  "category": "餐饮",
  "remark": "午餐",
  "date": "2025-12-26"
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| amount | number | ✅ | 金额，负数为支出，正数为收入 |
| category | string | ❌ | 类别，默认"其他" |
| remark | string | ❌ | 备注 |
| date | string | ❌ | 日期，默认当天，格式 YYYY-MM-DD |

### 响应示例

成功：
```json
{
  "success": true,
  "message": "记账成功",
  "data": {
    "date": "2025-12-26",
    "amount": -35.5,
    "category": "餐饮"
  }
}
```

失败：
```json
{
  "success": false,
  "message": "记账失败",
  "error": "Invalid amount"
}
```

---

## 调用示例

### cURL

```bash
curl -X POST https://your-n8n-domain/webhook/accounting \
  -H "Content-Type: application/json" \
  -d '{
    "amount": -35.5,
    "category": "餐饮",
    "remark": "午餐"
  }'
```

### JavaScript

```javascript
fetch('https://your-n8n-domain/webhook/accounting', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    amount: -35.5,
    category: '餐饮',
    remark: '午餐'
  })
});
```

### iOS 快捷指令

1. 创建新快捷指令
2. 添加"获取 URL 内容"操作
3. 设置 URL 为 Webhook 地址
4. 方法选择 POST
5. 请求体选择 JSON
6. 添加输入框获取金额和备注

---

## 类别建议

| 类别 | 适用场景 |
|------|----------|
| 餐饮 | 早餐、午餐、晚餐、外卖、饮料 |
| 交通 | 打车、公交、地铁、加油 |
| 购物 | 日用品、服装、电子产品 |
| 娱乐 | 电影、游戏、旅游 |
| 居住 | 房租、水电、物业 |
| 医疗 | 看病、买药 |
| 教育 | 课程、书籍 |
| 工资 | 工资收入 |
| 其他 | 未分类 |

---

## 常见问题

### Q: Token 过期怎么办？
A: `feishu-token-refresh.json` 工作流会每小时自动刷新 Token。

### Q: 如何查看记账数据？
A: 直接在飞书多维表格中查看，支持筛选、排序、统计。

### Q: 如何添加更多字段？
A:
1. 在飞书多维表格中添加新字段
2. 修改工作流中"写入飞书多维表格"节点的 JSON Body
