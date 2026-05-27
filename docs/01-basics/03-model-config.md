# 模型选择与配置：用最便宜的钱跑最强的AI

## 3.1 OpenClaw用什么模型

OpenClaw本身不包含AI模型，它需要通过API调用外部的AI服务。简单理解：

> OpenClaw = 你的大脑（思考+决策）
> 模型API = 你的知识储备（说出来/写出来）

所以模型选对了，体验就好。选错了，又贵又慢。

## 3.2 我们用的方案：DeepSeek V4 Flash

经过多轮对比，最后锁定 DeepSeek V4 Flash：

| 对比项 | GPT-4o | Claude Sonnet | DeepSeek V4 Flash |
|--------|:------:|:-------------:|:-----------------:|
| 价格(输入) | ¥42/百万token | ¥21/百万token | **¥1/百万token** |
| 价格(输出) | ¥168/百万token | ¥63/百万token | **¥2/百万token** |
| 缓存命中 | 半价 | 半价 | **¥0.028/百万token** |
| 中文能力 | 好 | 好 | **极好** |
| 上下文 | 128K | 200K | **1M** |

**结论：** DeepSeek V4 Flash 便宜到离谱，中文能力强，上下文1M（能记住一整本书）。日常对话和复杂任务都够用。

## 3.3 配置API Key

去 [DeepSeek官网](https://platform.deepseek.com) 注册账号 → 点 API Keys → 创建新Key。

然后把Key填到OpenClaw配置里：

```bash
openclaw config set models.providers.deepseek.apiKey "sk-你的key"
openclaw gateway restart
```

或者直接改配置文件（适用于飞牛版）：

找到 `/vol1/@apphome/trim.openclaw/data/home/.openclaw/openclaw.json`，在 `models.providers` 里加上：

```json
"deepseek": {
  "api": "openai-completions",
  "baseUrl": "https://api.deepseek.com/v1",
  "apiKey": "sk-你的key",
  "models": [
    {
      "id": "deepseek-v4-flash",
      "name": "DeepSeek V4 Flash",
      "input": ["text", "image"]
    }
  ]
}
```

## 3.4 控制花销

AI模型调用是按token计费的。1个token大约是1个汉字或3个英文字母。

为了防止用超预算，OpenClaw可以设置每日上限：

```bash
openclaw config set agents.defaults.maxCostPerDay 1.0
```

这就设了每天最多花1美元，超过了就自动停用。我们实测：用DeepSeek V4 Flash，每天频繁聊天+跑任务，一个月也就¥3-5块钱。

## 3.5 上下文窗口（能记住多少）

上下文越大，AI能记住的对话越多。但太大了也浪费。

我们设置的经验值：

```
contextTokens: 400000    ← 允许40万token上下文
reserveTokens: 80000     ← 留8万给新对话用
keepRecentTokens: 100000 ← 保留最近的10万token
```

这样设置的好处是：兼顾长对话能力和响应速度，不会因为历史太多导致"忘记前面说了什么"。

## 3.6 其他模型选择

如果你不想用DeepSeek，也可以：

| 模型 | 适合场景 |
|------|---------|
| 通义千问 Qwen | 中文写作、翻译 |
| 文心一言 ERNIE | 百度生态集成 |
| Ollama本地模型 | 完全免费，不联网（适合隐私场景） |

Ollama需要本地有GPU，我们的AI主机（RTX 3060 Ti）就跑了Ollama，用于本地推理。

## 3.7 小白常见问题

**Q: 为什么AI不回复了？**
A: 看看是不是额度用完了（`openclaw config get agents.defaults.maxCostPerDay`）

**Q: 为什么回复这么慢？**
A: DeepSeek V4 Flash 正常响应在0.5-2秒，如果很慢可能是网络问题

**Q: 怎么换模型？**
A: 改配置里的 `agents.defaults.model`，然后重启Gateway

## 3.8 最终建议

- **日常用**：DeepSeek V4 Flash（便宜够快）
- **复杂任务**：可以切更强的模型（多花的钱也值得）
- **私密数据**：用Ollama本地跑（但不适合长对话）
- **一定要设每日预算！** 忘了设的话，一次失误可能刷掉几十块钱
