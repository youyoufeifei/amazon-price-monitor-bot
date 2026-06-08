# Amazon 竞品价格监控 Bot — 交付物说明

基于 **Bright Data + 扣子 Coze** 搭建的全自动竞品情报采集 Bot，实现 Amazon 商品价格、评论数的每日自动监控，并将分析报告推送至钉钉群。

---

## 文件清单

| 文件 | 说明 |
| :--- | :--- |
| `导入文件.txt` | Coze 工作流完整 DSL，复制粘贴即可导入所有节点 |
| `images/workflow-canvas.png` | Coze 工作流画布截图 |
| `images/brightdata-output.png` | Bright Data 数据采集输出截图 |
| `README.md` | 本说明文件 |

---

## 前置条件

| 依赖 | 说明 |
| :--- | :--- |
| 扣子 Coze 账号 | [www.coze.cn](https://www.coze.cn)，需开通工作流功能 |
| Bright Data 账号 | https://get.brightdata.com/mcpserver-fei ，注册后获取 API Token，用于调用爬取服务 |
| 钉钉自定义机器人 | 在目标群聊中添加，获取 Webhook 地址 |

---

## 快速开始

### 第一步：导入工作流

由于 Coze 工作流导入/导出需要会员，本项目采用剪贴板导入方式：

1. 打开 `导入文件.txt`，**全选并复制**所有内容（Ctrl+A → Ctrl+C）
2. 登录 Coze，新建一个**工作流**
3. 在工作流画布内，**直接粘贴**（Ctrl+V）
4. 所有节点将自动还原到画布中

### 第二步：补全必填配置

粘贴后工作流节点已就位，按以下清单逐项填写你自己的配置：

| 序号 | 节点名称 | 需要配置的项 | 说明 |
| :---: | :--- | :--- | :--- |
| 1 | 开始节点 | — | 将输入变量连接到「代码-拼接商品URL」节点 |
| 2 | 结束节点 | — | 输出变量设置为「HTTP 请求-推送结果至钉钉」节点的输出 |
| 3 | 触发爬取请求 | `Authorization` | 填入 `Bearer YOUR_BRIGHTDATA_API_KEY` |
| 4 | 下载 snapshot 结果 | `Authorization` | 填入 `Bearer YOUR_BRIGHTDATA_API_KEY` |
| 5 | 大模型-数据清洗 | 模型选择 | 选择你有权限使用的大模型（如 DeepSeek-V3、GLM 等） |
| 6 | 大模型-数据分析 | 模型选择 | 同上 |
| 7 | 查询上次价格 | 数据库 | 选择你自己在 Coze 中创建的数据库 |
| 8 | 新增数据 | 数据库 | 同上 |
| 9 | HTTP 请求-推送结果至钉钉 | `url` | 填入你的钉钉机器人 Webhook 地址 |

### 第三步：创建数据库

在 Coze 工作台新建数据库，字段结构如下：

| 字段名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `asin` | String | Amazon 商品 ASIN |
| `price` | Float | 当日价格（仅数字） |
| `currency` | String | 货币代码，如 USD |
| `title` | String | 商品标题 |
| `reviews_count` | Integer | 评论数 |

创建完成后，在「查询上次价格」和「新增数据」节点中绑定该数据库。

### 第四步：测试运行

1. 在工作流输入框中填入一个 Amazon ASIN（10 位，如 `B08N5WRWNW`）
2. 点击「试运行」
3. 预期结果：钉钉群收到《今日 Amazon 竞品价格监控日报》

---

## 工作流架构

```
输入 ASIN
  → 清洗 ASIN（代码节点）
  → 拼接商品 URL（代码节点）
  → 触发 Bright Data 爬取（HTTP POST）
  → 判断是否需要轮询（选择器）
      ├─ 是 → 解析 snapshot_id → 循环等待（每次 10 秒，最多 10 次）
      │          → 下载结果 → 判断是否完成
      │              ├─ 完成 → 赋值中间变量 → 终止循环
      │              └─ 未完成 → 继续循环
      └─ 否 → 直接使用同步返回的数据
  → 大模型-数据清洗（结构化 JSON 提取）
  → 查询上次价格（数据库）
  → 新增今日数据（数据库）
  → 获取当前时间（代码节点）
  → 大模型-数据分析（生成日报 Markdown）
  → HTTP 推送至钉钉
```

---

## 常见问题

**Q：粘贴后节点连线丢失怎么办？**
按文章教程中的架构图，手动补连开始节点 → 代码-拼接商品URL，以及结束节点 ← 推送钉钉节点两处连线即可，其余节点连线在 DSL 中已保存。

**Q：运行报错 `401 Unauthorized`？**
检查「触发爬取请求」和「下载 snapshot 结果」两个 HTTP 节点的 `Authorization` 请求头，格式必须为 `Bearer <你的API Key>`，注意 Bearer 后有一个空格。

**Q：轮询 10 次后仍未拿到数据？**
Bright Data 异步任务通常在 30 秒内完成。如遇超时，可在循环节点将最大次数从 10 调大至 20，或将等待时间从 10 秒调整为 15 秒。

**Q：钉钉未收到消息？**
确认 Webhook 地址填写正确，且钉钉机器人「安全设置」未开启 IP 白名单限制（Coze 云端出口 IP 不固定）。建议将安全设置改为「自定义关键词」模式，关键词填 `Amazon`。

**Q：数据库中查不到历史数据（首次运行）？**
首次运行时数据库为空，「查询上次价格」节点会返回空列表，大模型-数据分析会在报告中注明"无历史数据可对比"，属于正常现象，第二次运行起即可进行同环比分析。

---

## 相关链接

- Bright Data 注册领试用额度：[https://get.brightdata.com/mcpserver-fei](https://get.brightdata.com/mcpserver-fei)
- Coze 工作台：[https://www.coze.cn](https://www.coze.cn)
- 项目 GitHub：[https://github.com/youyoufeifei/amazon-price-monitor-bot](https://github.com/youyoufeifei/amazon-price-monitor-bot)
