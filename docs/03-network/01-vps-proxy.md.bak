# VPS代理搭建：从翻墙到NAS容器全局走自己节点

> ⚠️ 长篇警告：这篇是整个教程最长最详细的。因为这是我们踩过最深的坑，也是最有价值的内容。

## 5.1 为什么要自建代理

之前NAS上的容器（n8n、Jellyfin、采集器）要通过翻墙访问外网时，用的是机场订阅（第三方代理服务）。机场的问题：

- ❌ 跑路风险：付了年费，可能两个月就关站
- ❌ 节点质量不稳定：高峰期卡成狗
- ❌ 限速限流：说是不限，实际用多了就慢

自己的VPS（Virtual Private Server，虚拟专用服务器）：

- ✅ 自己的机器，想怎么用怎么用
- ✅ 固定IP，不会突然变
- ✅ 一年$10左右，比机场便宜

## 5.2 选购VPS

我们买的是 RackNerd 洛杉矶机房的VPS：

```
配置：1核CPU、1GB内存、20G SSD硬盘
价格：约$10/年（折合人民币约¥7/月）
机房：洛杉矶（LA）
```

选购建议：

- 新手别买CN2 GIA线路（太贵，一个月$20+）
- 洛杉矶/圣何塞机房到中国大陆延迟能接受（~200ms）
- 便宜货够用就行，翻墙不需要高配置

购买后你会得到一个IP地址、root密码、SSH端口（默认22）。

## 5.3 连接VPS

Windows用户用 **PuTTY** 或 **Terminal**（Windows10及以上自带）：

```bash
ssh root@你的VPS_IP地址
# 输入密码（输入时不会显示，正常敲完回车就行）
```

Mac/Linux用户直接终端：

```bash
ssh root@你的VPS_IP地址
```

第一次连接会提示确认主机指纹，输入 `yes` 回车。

## 5.4 安装sing-box（核心代理程序）

sing-box 是一个"统一代理客户端"，支持多种翻墙协议。我们用它来搭建代理服务。

登录VPS后，执行：

```bash
# 下载sing-box
bash <(curl -fsSL https://sing-box.app/install.sh)

# 安装完成后检查版本
sing-box version
```

### 配置sing-box

创建配置文件 `/etc/sing-box/config.json`：

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-reality",
      "listen": "0.0.0.0",
      "listen_port": 52345,
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.microsoft.com",
            "server_port": 443
          },
          "private_key": "（生成的私钥）",
          "short_id": ["（生成的短ID）"]
        }
      }
    },
    {
      "type": "socks5",
      "tag": "socks5-in",
      "listen": "0.0.0.0",
      "listen_port": 1080
    }
  ]
}
```

> 对于小白：inbounds 就是"入口"，客户端的连接从这里进入VPS。
> - 端口52345：手机/电脑用小火箭连接（VLESS+Reality协议）
> - 端口1080：NAS上的容器走SOCKS5代理

### 生成Reality密钥

```bash
sing-box generate reality-keypair
# 会输出一个 PrivateKey 和 PublicKey
# 把 PrivateKey 填到配置里，PublicKey 给小火箭配置用
```

### 启动sing-box

```bash
systemctl enable sing-box
systemctl start sing-box
systemctl status sing-box  # 检查是否运行成功
```

## 5.5 Shadowsocks（最简单通用的协议）

有些客户端不支持VLESS+Reality，所以我们再加一个Shadowsocks。

Shadowsocks是经典的翻墙协议，几乎所有客户端都支持。

在sing-box的配置里加一个SS入站：

```json
{
  "type": "shadowsocks",
  "tag": "ss-in",
  "listen": "0.0.0.0",
  "listen_port": 52348,
  "method": "aes-256-gcm",
  "password": "（设置一个密码，像 b503bcd5-... 这样长一点）"
}
```

重启sing-box：

```bash
systemctl restart sing-box
```

## 5.6 开启系统转发（重要！）

这是刚装完VPS最容易忘的一步。不开启的话，流量进来就断了：

```bash
# 临时开启
sysctl -w net.ipv4.ip_forward=1

# 永久开启（重启也有效）
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

## 5.7 在手机上配置Shadowrocket

iOS手机用Shadowrocket（小火箭）连接VPS：

1. 打开Shadowrocket
2. 点右上角 `+` 添加节点
3. 类型选 Shadowsocks
4. 地址填VPS的IP，端口填52348
5. 加密方式选 aes-256-gcm
6. 密码填上面设置的密码
7. 保存，连接

如果连接成功，就可以上Google、YouTube了。

> 💡 更详细的Shadowrocket配置见第6章

## 5.8 让NAS容器走VPS代理

NAS上的容器（采集器、n8n等）需要翻墙才能访问海外网站。

方案：在NAS上跑mihomo（一个代理客户端），把所有流量转发到VPS。

### 安装mihomo

mihomo是Clash.Meta的延续，配置简单。

配置文件 `/vol2/1000/docker/mihomo/config.yaml`：

```yaml
port: 7890
socks-port: 1080
allow-lan: true
mode: rule

proxies:
  - name: "VPS"
    type: socks5
    server: 100.105.217.124  # VPS的Tailscale内网IP
    port: 1080

proxy-groups:
  - name: Proxy
    type: select
    proxies:
      - VPS

rules:
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,100.64.0.0/10,DIRECT
  - MATCH,Proxy
```

启动mihomo：

```bash
/mihomo -d /path/to/config
```

然后在容器的环境变量里设置代理：

```
HTTP_PROXY=http://172.17.0.1:7890
HTTPS_PROXY=http://172.17.0.1:7890
```

这样所有容器流量 → mihomo(:7890) → VPS(Tailscale SOCKS5:1080) → 外网。

## 5.9 ⚠️ 最大的坑：Tailscal这条命令差点炸了整个网络

这是搭建过程中最惨痛的事故，单独写一节强调：

### 错误操作

为了让手机能通过VPS访问家里NAS，在VPS上执行了：

```bash
tailscale up --accept-routes
```

### 造成的结果

这条命令让VPS**接受所有Tailscale子网路由**。然后：

1. NAS的路由表被污染，Docker容器路由全部混乱
2. mihomo代理节点连不上外网
3. Tailscale的DERP中继在全球乱跳
4. 最终OpenClaw超时、FN CONNECT断联、全部服务挂掉

### 根因

`--accept-routes` 会把所有Tailscale对端机器通告的路由全部注入到宿主机的路由表。如果VPS上通告了默认路由（0.0.0.0/0），那所有流量都指向Tailscale接口——互联网出口就没了。

### 正确做法

**VPS和NAS严格分离：**
- VPS：只负责翻墙（sing-box + Shadowsocks/SOCKS5）
- NAS：用Tailscale P2P直连回家
- 不要让VPS参与NAS的路由决策

## 5.10 最终稳定架构

```
┌────────────────────────────────────────────────────┐
│                    外网访问链路                      │
│                                                    │
│  手机Shadowrocket ──→ VPS(SS:52348) ──→ 翻墙       │
│  手机Tailscale ──→ NAS P2P直连 回家                │
│                                                    │
├────────────────────────────────────────────────────┤
│                   容器访问链路                      │
│                                                    │
│  NAS容器 ──→ mihomo(:7890) ──→ VPS(SOCKS5:1080)   │
│                                        │           │
│                                        └──→ 外网   │
└────────────────────────────────────────────────────┘
```

## 5.11 你现在有什么

- ✅ VPS上跑了sing-box（Reality + SOCKS5双协议）
- ✅ Shadowsocks兜底（兼容所有客户端）
- ✅ NAS上的mihomo只走VPS节点（不用机场了）
- ✅ 手机小火箭+电脑V2rayN都能用
- ✅ 锚点配置已存档，随便折腾都不怕

> 💡 **一句话总结**：VPS是你自己的机场，$10/年随便用，不怕跑路不限速。值得每一个折腾AI的人拥有。
