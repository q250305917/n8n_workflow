# Veo 3.1 TikTok带货视频自动生成工作流

## 工作流概述

**名称**: Veo 3.1 TikTok带货视频自动生成

**功能**: 自动从Google Sheets读取产品信息，调用kie.ai的Veo 3.1 API生成TikTok带货视频，上传到Google Drive，并更新表格状态。支持失败通知到企业微信机器人。

**触发方式**: 每天晚上23:00定时执行

---

## 数据源

### Google Sheets 表格结构

| 列名 | 说明 | 示例 |
|------|------|------|
| ID | 唯一标识符 | 1, 2, 3... |
| 产品图 | 产品图片URL | `https://example.com/product.jpg` |
| 拍摄风格 | 视频风格描述 | 现代、轻松, 专业、高端 |
| status | 处理状态 | pending/success/fail |
| 视频链接 | 生成的视频链接 | 自动填充 |

**表格ID**: `1o9xX2NXScJIOIcceWSnywgmd-AS9ErnANc6bntIFJfs`
**工作表名称**: `gid=0` (视频广告列表)

---

## 工作流节点结构

### 1. 定时触发 - 每天23:00
- **节点类型**: `scheduleTrigger`
- **配置**: 每天23:00触发执行

### 2. 读取未完成记录
- **节点类型**: `googleSheets`
- **操作**: 读取表格中所有记录
- **需要凭证**: Google Sheets OAuth2 API

### 3. 循环处理每条记录
- **节点类型**: `splitInBatches`
- **功能**: 逐条遍历表格记录

### 4. 准备Veo请求数据
- **节点类型**: `set`
- **功能**: 提取并转换数据格式
- **输出字段**:
  - `product_image_url`: 产品图URL
  - `shooting_style`: 拍摄风格
  - `row_number`: 行号
  - `prompt`: AI视频生成提示词

**提示词格式**:
```
TikTok带货视频，产品参考图：{产品图}，拍摄风格：{拍摄风格}。
请生成一段15秒的产品展示视频，风格要符合{拍摄风格}的要求，突出产品特色。
```

### 5. 调用kie.ai生成视频
- **节点类型**: `httpRequest` (POST)
- **API端点**: `https://api.kie.ai/api/v1/veo/generate`
- **请求头**:
  - `Authorization`: `Bearer {API_KEY}`
  - `Content-Type`: `application/json`
- **请求体**:
```json
{
  "prompt": "{prompt内容}",
  "imageUrls": ["{产品图URL}"],
  "model": "veo3_fast",
  "aspectRatio": "16:9",
  "generationType": "REFERENCE_2_VIDEO",
  "enableTranslation": true
}
```

### 6. 判断是否调用生成视频成功
- **节点类型**: `if`
- **判断条件**: `$json.code === 200`
- **TRUE分支**: → 等待60秒
- **FALSE分支**: → 企微通知 → 更新失败状态

### 7. 等待60秒
- **节点类型**: `wait`
- **功能**: 等待视频生成初始化

### 8. 检查视频生成状态
- **节点类型**: `httpRequest` (GET)
- **API端点**: `https://api.kie.ai/api/v1/veo/record-info?taskId={taskId}`
- **请求头**: `Authorization: Bearer {API_KEY}`

### 9. 判断是否完成
- **节点类型**: `if`
- **判断条件**: `$json.data.successFlag === 1`
- **TRUE分支**: → 获取视频URL
- **FALSE分支**: → 等待30秒后重试

### 10. 获取视频URL
- **节点类型**: `httpRequest` (GET)
- **URL**: `$json.data.response.resultUrls[0]`

### 11. 下载视频文件
- **节点类型**: `httpRequest`
- **响应格式**: file

### 12. 上传到Google云盘
- **节点类型**: `googleDrive`
- **文件名**: `{yyyyLLddHHmmss}-tiktok-video.mp4`
- **目标**: Google Drive根目录

### 13. 更新表格状态
- **节点类型**: `googleSheets` (update操作)
- **匹配列**: `ID`
- **更新字段**:
  - `status`: `success`
  - `视频链接`: `{webViewLink}`

### 14. 企微机器人-生成失败通知
- **节点类型**: `httpRequest` (POST)
- **API端点**: `https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key={WEBHOOK_KEY}`
- **消息内容**:
```
⚠️ TikTok视频生成失败

📅 时间: {当前时间}
🏷️ 任务ID: {row_number}
📦 产品图: {product_image_url}
🎬 拍摄风格: {shooting_style}

❌ 错误信息: {错误消息}
🔍 响应码: {code}
```

### 15. 更新表格失败状态
- **节点类型**: `googleSheets` (update操作)
- **匹配列**: `ID`
- **更新字段**:
  - `status`: `fail`

---

## 流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        开始：每天23:00触发                          │
└────────────────────────────┬────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    读取Google Sheets所有记录                        │
└────────────────────────────┬────────────────────────────────────────┘
                             ↓
                    ┌────────────────┐
                    │ 循环处理每条记录 │◄──────────────┐
                    └────────┬───────┘                │
                             ↓                         │
              ┌──────────────────────────┐            │
              │    准备Veo请求数据       │            │
              │  - 提取产品图URL         │            │
              │  - 构建AI提示词         │            │
              └──────────┬───────────────┘            │
                         ↓                            │
              ┌──────────────────────────┐            │
              │  调用kie.ai生成视频      │            │
              │  POST /api/v1/veo/generate│            │
              └──────────┬───────────────┘            │
                         ↓                            │
              ┌──────────────────────────┐            │
              │ 判断调用是否成功 (code==200)│          │
              └──┬───────────────────────┬┘            │
                 │ TRUE                  │ FALSE       │
                 ↓                       ↓             │
          ┌────────────┐      ┌──────────────────┐   │
          │  等待60秒   │      │ 企微失败通知     │   │
          └─────┬──────┘      │ 更新状态=fail     │   │
                │             └─────────┬─────────┘   │
                ↓                       │             │
          ┌────────────┐                │             │
          │检查生成状态 │                │             │
          │GET record-info              │             │
          └─────┬──────┘                │             │
                ↓                       │             │
          ┌──────────────────────────┐   │             │
          │判断是否完成 (successFlag==1)│  │             │
          └──┬───────────────────────┬┘   │             │
             │ TRUE                  │ FALSE           │
             ↓                       ↓                 │
      ┌──────────┐           ┌────────────┐          │
      │获取视频URL│           │ 等待30秒   │───────────┤
      └────┬─────┘           └──────┬─────┘
           ↓                        │
    ┌────────────┐                 │
    │下载视频文件│                 │
    └────┬───────┘                 │
         ↓                          │
    ┌────────────┐                 │
    │上传GoogleDrive               │
    └────┬───────┘                 │
         ↓                          │
    ┌────────────┐                 │
    │更新状态=success──────────────┘
    └──────┬─────┘
           ↓
      回到循环处理下一条
```

---

## API配置说明

### kie.ai Veo 3.1 API

**1. 生成视频**
```
POST https://api.kie.ai/api/v1/veo/generate

Headers:
  Authorization: Bearer YOUR_API_KEY
  Content-Type: application/json

Body:
{
  "prompt": "视频描述",
  "imageUrls": ["参考图URL"],
  "model": "veo3_fast",
  "aspectRatio": "16:9",
  "generationType": "REFERENCE_2_VIDEO",
  "enableTranslation": true
}

Response:
{
  "code": 200,
  "msg": "success",
  "data": {
    "taskId": "xxx"
  }
}
```

**2. 查询状态**
```
GET https://api.kie.ai/api/v1/veo/record-info?taskId={taskId}

Response:
{
  "code": 200,
  "data": {
    "successFlag": 1,  // 1=完成, 0=处理中
    "response": {
      "resultUrls": ["视频URL"]
    }
  }
}
```

### 企业微信机器人

```
POST https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key={KEY}

Body:
{
  "msgtype": "text",
  "text": {
    "content": "通知内容"
  }
}
```

---

## 所需凭证

| 服务 | 凭证类型 | 用途 |
|------|----------|------|
| Google Sheets | OAuth2 API | 读取/更新表格数据 |
| Google Drive | OAuth2 API | 上传视频文件 |
| kie.ai | API Key (Bearer Token) | 调用Veo 3.1 API |
| 企业微信机器人 | Webhook Key | 发送失败通知 |

---

## 错误处理

1. **API调用失败** → 企业微信通知 → 更新status为fail
2. **视频生成超时** → 每30秒轮询一次状态
3. **状态检查失败** → 继续轮询直到成功

---

## 配置参数清单

创建工作流时需要配置的参数：

- [ ] Google Sheets 表格ID
- [ ] Google Sheets 工作表名称
- [ ] kie.ai API密钥
- [ ] 企业微信机器人Webhook密钥
- [ ] Google Drive 上传目录
- [ ] 视频宽高比 (16:9 或 9:16)
- [ ] 视频模型 (veo3 或 veo3_fast)
- [ ] 轮询间隔时间
- [ ] 初始等待时间

---

## 使用说明

1. 在Google Sheets中准备好产品数据（ID、产品图、拍摄风格）
2. 配置n8n工作流所需的凭证
3. 设置定时触发时间
4. 激活工作流
5. 系统将自动生成视频并更新状态

---

## 注意事项

1. 确保产品图URL可公开访问
2. kie.ai API有调用频率限制
3. 视频生成通常需要1-3分钟
4. 建议设置合理的轮询间隔避免超限
5. 企业微信通知需提前配置机器人

---

## 生成命令

将此提示词提供给AI助手，使用以下命令生成工作流：

```
请基于这份提示词，使用n8n工作流格式生成"Veo 3.1 TikTok带货视频自动生成"工作流
```
