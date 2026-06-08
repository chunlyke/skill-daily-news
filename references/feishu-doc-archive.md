# 飞书云盘存档 — 每日日报

## 前置条件

飞书应用需要开通以下权限：
- `drive:drive` — 云盘读写（创建/移动文件到文件夹）
- `docx:document` — 文档读写
- `bitable:app` — 多维表格读写（如需创建/管理 Bitable）

**权限开通路径：** https://open.feishu.cn/app/cli_aa94e3cf58385cc4/permission
在当前 Hermes 飞书应用 `cli_aa94e3cf58385cc4` 中添加权限后需要**发布新版本**才能生效。

## 认证流程

所有 Feishu API 调用共用同一认证流程：

```python
import requests, os
r = requests.post('https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal',
    json={"app_id": "cli_aa94e3cf58385cc4", "app_secret": os.environ["FEISHU_APP_SECRET"]})
token = r.json()["tenant_access_token"]
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
```

**记住**：token 有效期 3600 秒（1 小时），长期操作需刷新。

## 核心 API — Docx 文档

### 1. 创建文档

```
POST /open-apis/docx/v1/documents
{"title": "📰 今日新闻日报 · 2026年6月X日"}
```

返回值中的 `document.document_id` 即后续操作的 `doc_id`。

### 2. 写入内容（追加到文档根块）

文档创建后默认有一个根块（block_id = `doc_id`）。在根块下追加子块：

```
POST /open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children
```

请求体为块数组，最常用的块类型：

| 块类型 | type_id | content_key | 说明 |
|--------|---------|-------------|------|
| 文本段落 | 2 | `text` | 普通段落 |
| 标题1 | 3 | `heading1` | 一级标题 |
| 标题2 | 4 | `heading2` | 二级标题 |
| 无序列表 | 12 | `bullet` | 圆点列表项 |

**注意**：飞书 Docx API 的 block content 格式要求每个块有一组 `children`，每个 child 是一个 text_element。不能直接用 Markdown 字符串写入，必须用结构化的 text_element 数组。

**简化写法**（Python requests 示例）：

```python
import requests

headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

blocks = [
    {
        "block_type": 2,
        "text": {
            "elements": [
                {
                    "text_run": {"content": "🔥 芯海直击"},
                    "text_element_style": {"bold": True}
                }
            ],
            "style": {}
        }
    },
    {
        "block_type": 14,
        "text": {
            "elements": [
                {
                    "text_run": {"content": "新闻摘要 https://example.com"},
                    "text_element_style": {}
                }
            ],
            "style": {}
        }
    }
]

r = requests.post(
    f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children",
    headers=headers,
    json={"children": blocks}
)
```

### 3. 移动文档到「每日日报」文件夹

```
PATCH /open-apis/drive/v1/files/{file_token}
{"parent_token": "每日日报文件夹的token"}
```

其中 `file_token` 即创建文档时返回的 `document.document_id`。

### 4. 获取「每日日报」文件夹 token

```
GET /open-apis/drive/v1/files?page_size=100
```

遍历返回的 `files` 数组，找到 `type="folder"` 且 `name="每日日报"` 的条目，取其 `token`。

如果文件夹不存在，可通过创建文档后移动的方式绕过，或使用 API 创建：

### 创建文件夹（API）

```
POST /open-apis/drive/v1/files
{
  "name": "每日日报",
  "type": "folder",
  "parent_token": "根目录token"
}
```

根目录 token 可通过 `GET /open-apis/drive/v1/files` 的第一个返回结果的 `parent_token` 获得（所有根目录文件的 parent_token 相同）。当前环境根目录 token 为 `nodcnbQrUmOEN7oiNIS7Q0RnROc`。

验证：文件夹创建后，再次 `GET /open-apis/drive/v1/files` 应看到 `type=folder` 且 `name="每日日报"` 的条目。

## 完整工作流（Python）

```python
import requests, json, os
from datetime import date

today = date.today()

# 1. 获取 token
r = requests.post('https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal',
    json={"app_id": "cli_aa94e3cf58385cc4", "app_secret": os.environ["FEISHU_APP_SECRET"]})
token = r.json()["tenant_access_token"]
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# 2. 创建文档
r = requests.post('https://open.feishu.cn/open-apis/docx/v1/documents',
    headers=headers, json={"title": f"📰 今日新闻日报 · {today.year}年{today.month}月{today.day}日"})
doc_id = r.json()["data"]["document"]["document_id"]

# 3. 写入内容（此步较复杂，要构造 text_element 数组）

# 4. 移动文件夹（先查文件夹 token）
rr = requests.get('https://open.feishu.cn/open-apis/drive/v1/files', headers=headers, params={"page_size": 100})
folder_token = None
for f in rr.json()["data"]["files"]:
    if f["name"] == "每日日报" and f["type"] == "folder":
        folder_token = f["token"]
        break

if folder_token:
    requests.patch(f"https://open.feishu.cn/open-apis/drive/v1/files/{doc_id}",
        headers=headers, json={"parent_token": folder_token})
```

## 已知陷阱

### Token 在 shell 命令中被拦截
在 terminal 工具中直接写 `Bearer <token>` 会被安全系统替换为 `***`，导致 99991668 错误。**解决方案**：使用 Python `requests` 库（通过 `execute_code`）代替 shell curl。

### 日期字段创建失败（2026-06-07）
飞书 Bitable 日期字段创建时，`property` 中格式参数为 `"date_formatter": "yyyy/MM/dd"` 而非 `"format": 1`。且**不带 property 也能创建成功**（自动使用默认格式）。

### Field PATCH 返回 404
PATCH 方式更新字段名返回 404。需使用 PUT 方式并传入完整的 `type` + `property`：
```python
requests.put(f"{base}/fields/{field_id}", headers=headers, json={
    "field_name": "新名称",
    "type": 1,
    "property": {"type": 1, "text_options": {}}
})
```

### 设置主字段
设置表的 primary_field_id 的 PATCH endpoint 可能返回 "request body cannot be empty"。需通过飞书 UI 手动设置主字段。

## 存档格式

与飞书推送格式一致，紧凑排列：

```
📰 **今日新闻日报 · YYYY年M月D日**
**🔥 芯海直击**
- 摘要 链接
**📋 政策雷达**
- 摘要 链接
...
```

---

## Bitable 多维表格创建（扩展）

适用于为 Kevin 构建飞书在线表格的场景（如人脉管理系统）。

### 创建多维表格

```python
r = requests.post('https://open.feishu.cn/open-apis/bitable/v1/apps',
    headers=headers, json={"name": "表名"})
app_token = r.json()["data"]["app"]["app_token"]
default_table_id = r.json()["data"]["app"]["default_table_id"]
```

### 字段类型参考

| 中文名 | type | property 关键参数 |
|--------|------|------------------|
| 文本 | 1 | `{"type":1,"text_options":{}}` |
| 数字 | 2 | `{"type":2,"num_options":{"decimal_count":0,"formatter":"0"}}` |
| 单选 | 3 | 需预置 options 数组 |
| 日期 | 5 | 不带 property 也能创建，自动使用默认格式 |
| 附件 | 17 | 无额外 property |

### 创建字段（通用模式）

```python
body = {"field_name": "字段名", "type": 1, "property": {"type": 1, "text_options": {}}}
r = requests.post(f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields", headers=headers, json=body)
```

### 单选字段（预置选项）

```python
body = {
    "field_name": "圈层标签",
    "type": 3,
    "property": {
        "options": [
            {"name": "核心圈", "color": 0},
            {"name": "商界", "color": 1},
            # color 0-7 可用
        ]
    }
}
```

### 更新字段名（PUT 而非 PATCH）

```python
requests.put(f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields/{field_id}",
    headers=headers, json={"field_name": "新名称", "type": 1, "property": {"type": 1, "text_options": {}}})
```

### 删除字段

```python
requests.delete(f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields/{field_id}", headers=headers)
```

### 设置主字段（⚠️ 已知问题）

飞书 API **不支持通过 API 设置 primary_field_id**。PATCH table endpoint 返回 "The request body cannot be empty" 错误。**解决方案：** 重命名默认主字段（"文本"→"编号"，用 PUT 完整字段定义），或通过飞书 UI 手动设置。

### 添加记录

```python
r = requests.post(f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records",
    headers=headers, json={
    "fields": {
        "姓名": "张三",
        "圈层标签": "核心圈",    # 单选字段用选项名
        "亲密度(SVI)": 8,       # 数字字段用数值
        "上次联系": 1780000000  # 日期字段用 Unix 时间戳
    }
})
```

### 删除多维表格

```python
requests.delete(f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}", headers=headers)
```

---

## 文件夹状态（2026-06-08 更新）

- 名称：每日日报
- token：`AaXwfDRWHlGMY9d8soac5cuenab`
- 位置：飞书云盘根目录
- 访问链接：https://kcnuyvc64l5g.feishu.cn/drive/folder/AaXwfDRWHlGMY9d8soac5cuenab
- 状态：**已创建** ✅
- drive:drive 权限已验证可用
- **每次 cron 执行日报后，必须自动创建飞书 Docx 文档并移至该文件夹**