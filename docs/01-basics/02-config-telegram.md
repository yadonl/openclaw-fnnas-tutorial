# 打通 Telegram 通道

有了 Web 界面能聊天还不够——你总不能一直开着网页吧？把 Telegram 接上，就可以随时随地用手机跟你的 AI 助手聊天了。

## 2.1 注册 Telegram Bot

Telegram Bot 是 OpenAI 的「中间人」——你的消息发给 Bot，Bot 转给 OpenClaw，OpenClaw 处理完再让 Bot 回你。

操作步骤：

1. **打开 Telegram**，搜索 `@BotFather`（官方机器人父亲，认准蓝色勾✓）
2. 输入 `/newbot`，回车
3. BotFather 会问你两个问题：
   - **Bot 名字**——随便起，比如「小月」
   - **Bot 用户名**——必须以 `bot` 结尾，比如 `yue_ai_bot`。取一个没人用过的
4. 搞定了你会看到一长串东西，长这样：

```
888431234:AAHxyz...一堆字母数字
```

这串就是 **Bot Token**，相当于你家大门钥匙，**千万别泄露给任何人**。谁拿到 Token 谁就能控制你的 Bot。

## 2.2 在 OpenClaw 里配置 Telegram

拿到 Token 之后，打开 OpenClaw 的配置文件（如果你用飞牛安装，文件在 `/vol1/@apphome/trim.openclaw/config.yaml`）。

找到 `channels` 部分，把 Telegram 配置加上。不会配的直接照着下面抄：

```yaml
channels:
  telegram:
    botToken: "888431234:AAHxyz...你刚拿到的Token"
    dmPolicy: "pairing"   # ← 这个最重要！不设的话 Bot 不理你
```

### 关键：dmPolicy 是什么鬼？

简单说，Telegram Bot 默认是「谁都能找它聊天」。但 OpenClaw 比较谨慎——它不会自动回复每个陌生人。`dmPolicy` 就是过滤规则：

- `pairing`（推荐）——只有你在 OpenClaw 配置文件里配过的用户才能跟 Bot 对话
- `all`——谁都能聊（不太安全，不推荐）
- `none`——私聊完全不给回复（基本不用）

**小白最容易踩的坑就在这里：** 很多人配了 Token 就觉得完事了，发现 Bot 不说话，90% 是因为没写 `dmPolicy: "pairing"`。

### 中国大陆用户：代理配置

如果你在国内，Telegram Bot 需要梯子才能连上。两种方法：

**方法一：直接写在配置里**

```yaml
channels:
  telegram:
    botToken: "你的Token"
    dmPolicy: "pairing"
    proxy: "http://127.0.0.1:7890"
```

**方法二：用环境变量**

```bash
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
```

方法二的优点是所有网络请求都会走代理，不止 Telegram。看你喜好。

## 2.3 测试连接

配置改完，重启 OpenClaw Gateway 让配置生效：

```bash
openclaw gateway restart
```

等它启动好（大概十几秒），去 Telegram 里找到你的 Bot：

1. 搜索你的 Bot 用户名
2. 点 **Start** 或发送 `/start`
3. Bot 回复你了 → ✅ 搞定！
4. 没反应 → 看下面的「常见问题」

## 2.4 常见问题

**Q：Bot 完全不回复我**

最常见的原因是 **没配 `dmPolicy`**。回去检查配置文件，确保写的是 `dmPolicy: "pairing"`，然后重启。

**Q：我能回复 Bot，但 Bot 回的消息我收不到**

这不是 Telegram 的问题，是 OpenClaw 的 channel 配置没生效。重启 Gateway 试试：

```bash
openclaw gateway restart
```

**Q：配了代理还是连不上**

检查代理地址对不对。如果你用 Clash，默认端口多数是 `7890`。也可以试试换成 `socks5://127.0.0.1:7890`。还不行就去看 OpenClaw 的日志：

```bash
openclaw gateway logs
```

日志里会有详细的错误信息，能帮你定位问题。

---

**下一步：** 配置好 Telegram 之后，你就可以随时随地用手机跟你的 AI 助手聊天了。接下来我们建议你看看 [配置飞牛NAS通道](03-config-fnnas.md)，把聊天记录持久化存起来。
