# Shadowrocket配置与iOS快捷指令

## 6.1 在手机上配置Shadowrocket

Shadowrocket（俗称"小火箭"）是iOS上最好的代理客户端之一。

### 添加节点

1. 打开Shadowrocket
2. 点右上角的 `+` 号
3. 按以下填写：

```
类型: Shadowsocks
地址: （你的VPS的IP地址）
端口: 52348
加密: aes-256-gcm
密码: （你设置的密码）
备注: VPS
```

4. 点「完成」
5. 点击新添加的节点，出现绿色勾号说明选中了
6. 顶部的开关点一下，变成绿色就是连接成功了

### 配置规则

小火箭的规则决定了哪些网站走代理，哪些直连。

最简单的规则只需要两条：

```
GEOIP,CN,DIRECT    ← 访问国内网站直连
FINAL,PROXY        ← 其余全部走代理
```

怎么设置：打开Shadowrocket → 配置 → 点当前配置 → 编辑纯文本 → 在 [Rule] 下面加这两行。

### 完整配置参考

```
[General]
bypass-system = true
skip-proxy = 192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,localhost,*.local
dns-server = https://dns.alidns.com/dns-query
ipv6 = false
private-ip-answer = true

[Rule]
GEOIP,CN,DIRECT
FINAL,PROXY
```

> `skip-proxy` 的意思是：访问内网IP（192.168.x.x、10.x.x.x等）时跳过代理，因为内网不需要翻墙。

## 6.2 三个快捷指令：出门/在外/回家

iOS的快捷指令可以让手机一键切换网络状态。我们设了三个：

### 出门（离开家 → 开翻墙）

```
操作：
  1. URL: shadowrocket://connect
  2. 打开URL
```

效果：打开小火箭，自动连接之前选好的VPS节点。

### 在外（不用翻墙了 → 全关掉）

```
操作：
  1. URL: shadowrocket://disconnect
  2. 等待2秒
  3. URL: tailscale://
  4. 打开URL
```

效果：先断开小火箭，再打开Tailscale（方便回家时连内网）。

### 回家（到了家 → 开Tailscale连NAS）

```
操作：
  1. URL: tailscale://
  2. 打开URL
  3. 等待3秒
  4. URL: shadowrocket://disconnect
  5. 打开URL
```

效果：打开Tailscale连回家里内网，同时关掉翻墙（在家不需要）。

### 三个指令的核心逻辑

| 场景 | 小火箭 | Tailscale | 说明 |
|------|:------:|:---------:|------|
| 出门 | ✅ 开 | ❌ 关 | 翻墙上网 |
| 在外 | ❌ 关 | ❌ 关 | 啥都不用 |
| 回家 | ❌ 关 | ✅ 开 | 访问NAS |

### 如何设置

1. 打开iPhone的「快捷指令」App
2. 点右上角 `+` 新建
3. 搜索添加「URL」操作 → 填入对应的链接
4. 再添加「打开URL」操作
5. 重命名（出门/在外/回家）
6. 添加到桌面：桌面长按 → 左上角 `+` → 选快捷指令 → 选你创建的

## 6.3 进阶：自动触发（不用手动点）

可以设置地理位置自动化：

1. 打开快捷指令 → 自动化 → 创建个人自动化
2. 选「离开」→ 定位到你家地址 → 运行「出门」
3. 选「到达」→ 定位到你家地址 → 运行「回家」

设好后，出门自动开翻墙，到家自动开内网，全程不用手动操作。
