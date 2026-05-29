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

## 5.3 首次连接VPS

买完之后，你会得到一个IP、root密码。连接：

```bash
ssh root@你的IP
# 输入密码
```

> ⚠️ 拿到VPS后第一步：先装fail2ban、改SSH端口、设密钥登录。公网IP被扫只是时间问题。

## 5.4 安装甬哥Sing-box一键脚本

不推荐手动配，直接用甬哥(yonggekkk)的一键脚本，省心且支持五协议：

```bash
bash <(curl -Ls https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)
```

回车三次（全部默认），脚本会自动：
- 装sing-box最新版
- 生成五协议配置：Vless-Reality / Vmess-WS / Hysteria2 / TUIC-v5 / AnyTLS
- 生成自签证书 + Warp出站
- 设置iptables防火墙

装完后你会看到类似这样的信息：

```
Vless-reality端口：25316
Vmess-ws端口：2052  
Hysteria-2端口：34660
Tuic-v5端口：64648
Anytls端口：53394
UUID：aaf5e571-71d0-41a1-ae93-c35a1a6ffaf8
```

脚本快捷方式：`sb`（随时运行管理）

## 5.5 关键优化：Reality端口改443

默认端口25316太显眼。运营商扫到非标端口会标记。改到443（HTTPS标准端口）：

```bash
# 手动改配置
python3 -c "
import json
c = json.load(open('/etc/s-box/sb.json'))
for i in c['inbounds']:
    if i.get('type')=='vless':
        i['listen_port'] = 443
json.dump(c, open('/etc/s-box/sb.json','w'), indent=4)
"
systemctl restart sing-box
```

## 5.6 加伪装站（让服务器看起来正常）

扫描器访问80端口看到一个正常页面，而不是"连接被拒绝"：

```bash
apt install nginx -y
systemctl enable --now nginx
# 访问 http://你的域名 会显示Nginx默认页
```

## 5.7 防火墙白名单

甬哥脚本会装 iptables-persistent 替代 UFW。手动锁端口：

```bash
iptables -F INPUT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT    # SSH
iptables -A INPUT -p tcp --dport 80 -j ACCEPT       # 伪装站
iptables -A INPUT -p tcp --dport 443 -j ACCEPT      # Reality
iptables -A INPUT -s 100.64.0.0/10 -p tcp --dport 1080 -j ACCEPT  # NAS SOCKS5
iptables -A INPUT -j DROP                           # 其余全封
netfilter-persistent save
```

> ⚠️ 重要：iptables 规则有顺序。先放行再DROP，**新规则一定要用 `-I` 插到DROP前面**，别用 `-A` 追加到DROP后面，否则连不上！

## 5.8 加固（SSH + fail2ban）

### SSH密钥登录（改端口到2222）

Ubuntu 24.04用systemd socket激活SSH，改端口不能用传统的Port指令，要用socat转发：

```bash
# 先生成本机密钥对，把公钥放到VPS ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# 禁用密码登录
cat >> /etc/ssh/sshd_config.d/99-hardening.conf << "CONF"
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password
MaxAuthTries 3
CONF

# socat转发实现端口2222→22
cat > /etc/systemd/system/ssh-port-2222.service << "EOF"
[Unit]
Description=SSH Port 2222 Forwarder
After=network.target
[Service]
Type=simple
ExecStart=/usr/bin/socat TCP-LISTEN:2222,reuseaddr,fork TCP:127.0.0.1:22
Restart=always
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now ssh-port-2222.service
systemctl restart ssh
```

以后SSH：`ssh -p 2222 -i 私钥文件 root@你的IP`

### fail2ban防护

```bash
apt install fail2ban -y
cat > /etc/fail2ban/jail.local << "F2B"
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 24h
F2B
systemctl restart fail2ban
```

实战数据：部署后半小时就拦截了一个扫描IP。

## 5.9 Cloudflare隐藏源站

把域名迁到Cloudflare（改NS），可以做两件事：
- 隐藏VPS真实IP（防运营商扫端口）
- DNS解析走CF全球网络

### DNS记录配置

```
dddlin.cyou     A → 你的VPS IP  灰云(DNS only)  ← 不要开橙云！
reality.dddlin.cyou  A → 你的VPS IP  灰云(DNS only)
```

> ⚠️ Reality/Hy2不走Cloudflare CDN，必须灰云。开橙云会断连。

### 证书

Cloudflare生成Origin Certificate（免费，长期有效），上传到VPS：

```bash
# 把CF给的cert.pem和key.pem放到
# /etc/sing-box/cert/ 或 /etc/s-box/ （取决于甬哥脚本版本）
```

## 5.10 手机上配置Shadowrocket

小火箭配置Reality（主用，走TCP/443，不会被运营商限速）：

```
vless://你的UUID@你的域名:443?encryption=none&security=reality&flow=xtls-rprx-vision&sni=apple.com&fp=chrome&pbk=公钥&sid=短ID#节点名
```

Hy2备用（如果运营商不限UDP的话）：

```
hysteria2://密码@你的VPS_IP:34660?alpn=h3&insecure=1&sni=apple.com#Hy2
```

> 💡 Reality 443是主用，Hy2当备用。Reality走TCP很稳，Hy2快但容易被运营商限UDP。

### 为什么伪装成apple.com？

```text
你在中国 → 连apple.com → 正常应该到苹果服务器
苹果的CDN(Akamai/Cloudfront)在全球有大量节点，LA是很常见的苹果服务器位置
iPhone/iCloud流量大量走到LA → 运营商看到443端口的apple.com TLS流量 → 完全正常
```

国内主流网站（baidu.com）反而不合适：
```text
你在中国 → 连baidu.com → 正常应该到北京/上海的服务器
结果你的流量跑到LA去了 → 运营商一看：这流量不对劲
```

## 5.11 让NAS容器走VPS代理

NAS上的容器（采集器、n8n、Jellyfin）需要翻墙访问外网。

方案：NAS上跑mihomo（代理客户端），把所有流量转发到VPS。

### mihomo配置

```yaml
# /vol2/1000/docker/mihomo/config.yaml
port: 7890
socks-port: 1080
allow-lan: true
mode: rule

proxies:
  - name: "VPS"
    type: socks5
    server: 100.105.217.124  # VPS的Tailscale IP
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

### 容器环境变量

```bash
HTTP_PROXY=http://172.17.0.1:7890
HTTPS_PROXY=http://172.17.0.1:7890
```

> ⚠️ mihomo要配置正确，不然国内站也走VPS，白白浪费VPS流量。

## 5.12 最终安全架构

```text
┌──────────────────────────────────────────────────────────────┐
│                        外网访问链路                           │
│                                                              │
│  手机 → Reality(443/TCP/HTTPS伪装) → VPS(唯一公网入口)       │
│                                          ↘                   │
│  手机 → Tailscale P2P → NAS 回家          100.105.217.124:   │
│                                          1080(SOCKS5)        │
│                                            ↓                 │
│                                         NAS mihomo           │
│                                            ↓                 │
│                                         外网                 │
├──────────────────────────────────────────────────────────────┤
│                        安全措施                               │
│                                                              │
│  SSH: 2222(socat) + 密钥登录(fail2ban防守)                   │
│  DNS: Cloudflare(灰云) + CF Origin Cert(2041年到期)          │
│  防火墙: iptables白名单(仅2222/80/443/1080)                  │
│  伪装站: Nginx:80(dddlin.cyou)                               │
│  旧服务停用: 独立Hy2、Shadowsocks、UFW                       │
└──────────────────────────────────────────────────────────────┘
```

## 5.13 ⚠️ 最大的坑：iptables规则顺序导致SSH锁死

搭建过程中最惨痛的事故😅

### 错误操作

想加SSH端口2222放行时用了 `-A`（追加）：

```bash
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT  # ❌ 追加到DROP后面
iptables -A INPUT -j DROP                         # ← DROP在前面
```

2222规则加在了DROP**后面**，永远不会被匹配到！

### 结果

SSH断连，只能从RackNerd面板进VNC控制台修复。

### 正确操作

```bash
iptables -I INPUT 3 -p tcp --dport 2222 -j ACCEPT  # ✅ 插到DROP前面
```

### 教训

1. ✅ 有DROP规则的链，新规则用 `-I`（insert）插到前面，不用 `-A`（append）
2. ✅ 改防火墙前先 `iptables -L INPUT -n --line-numbers` 确认规则顺序
3. ✅ Ubuntu 24.04用systemd socket管理SSH，改端口用socat转发更简单

## 5.14 ⚠️ 第二个坑：SOCKS5监听127.0.0.1导致NAS断连

### 错误操作

想把SOCKS5从0.0.0.0改为内网地址：

```bash
"listen": "127.0.0.1"  # ❌ NAS通过Tailscale访问，127.0.0.1只接受本机连接
```

### 正确做法

```bash
TS_IP=$(tailscale ip -4)  # 获取VPS的Tailscale IP（如100.105.217.124）
"listen": "$TS_IP"        # ✅ 只接受Tailscale内网连接，安全且NAS能连
iptables -A INPUT -s 100.64.0.0/10 -p tcp --dport 1080 -j ACCEPT
```

## 5.15 你现在有什么

- ✅ VPS上sing-box五协议（Reality 443主用 + Hy2 34660备用）
- ✅ Cloudflare DNS + 灰云隐藏源站
- ✅ SSH 2222端口 + 密钥登录 + fail2ban防护
- ✅ iptables白名单（只开放必要端口）
- ✅ Nginx伪装站（80端口，服务器看起来正常）
- ✅ SOCKS5代理（仅Tailscale内网，NAS专用）
- ✅ CF Origin Certificate（2041年到期）
- ✅ NAS mihomo全局走VPS（隐藏家庭IP）

> 💡 **一句话总结**：从裸奔的Lucky DDNS，进化到VPS公网边界+Tailscale内网回家+Cloudflare隐藏源站的完整安全架构。值得每一个折腾AI的人照做。
