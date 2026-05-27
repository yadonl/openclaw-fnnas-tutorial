# 第1章：飞牛NAS上部署OpenClaw

## 1.1 飞牛应用商店的一键安装

Coming soon—this one's straightforward, but we'll note the quirks.

---

# 第5章：VPS代理搭建

## 5.4 ⚠️ Tailscale --accept-routes 血泪教训

这是整个搭建过程中**最惨痛的一次事故**。

### 事故经过

在VPS上安装了Tailscale，为了能访问家中NAS的内网（192.168.5.0/24），执行了：

```bash
tailscale up --accept-routes
```

这条命令让VPS**接受所有Tailscale子网路由**。随后NAS路由器上的路由表被Tailscale注入，导致：

1. NAS的Docker容器路由混乱，部分容器无法联网
2. mihomo代理节点全部失联
3. Tailscale的DERP中继在全球乱跳
4. 最终：**OpenClaw超时、FN CONNECT断联、全部服务挂掉**

### 根因

Tailscale的`--accept-routes`会把所有对端advertise的路由全部注入到宿主机的路由表中。如果VPS上advertise了默认路由（0.0.0.0/0），那相当于把整个VPS的流量都指向了Tailscale接口——包括原本直连的互联网出口。

### 正确做法

VPS和NAS严格分离：
- **VPS**：只负责翻墙（sing-box + Shadowsocks）
- **NAS**：用Tailscale P2P直连回家
- **不要**让VPS参与NAS的路由决策

---

# 第7章：n8n工作流迁移实录

## 7.1 Code节点沙箱问题

### 症状

n8n 2.20.11 的所有工作流全部报错：

```
"errorMessage":"Module 'fs' is disallowed [line 1]"
```

### 根因

n8n 2.20.11 的Code节点（v2）使用task runner沙箱执行JavaScript，**完全禁止 `require('fs')`**。所有之前依赖文件读写的Code节点全部崩溃。

### 修复方案

弃用Code节点直接读文件，改为HTTP Request节点从本地数据服务器获取数据：

```
流程：
Trigger → HTTP Request(本地数据服务器) → Code节点(只处理$json输入)
```

本地启动一个Python HTTP服务器（data_server.py，端口18888），提供静态JSON/TXT文件服务。

---

## 7.2 n8n URL表达式别用$today

### 症状

HTTP Request节点URL写成：

```
="https://api.example.com/data?date=" + $today.format("YYYY-MM-DD")
```

n8n不执行求值，直接把字面量当URL发出去，报"URL must start with http"。

### 根因

n8n HTTP Request节点v4.2的URL字段不支持复杂JS表达式，`$today`在该上下文不可用。

### 修复

删除HTTP Request数据获取节点 → 替换为Code节点v2 + `fetch()`。

Code节点用 `new Date()` 计算日期，调用 `await fetch('http://...')` 获取数据，返回 `[data]`（JSON）或 `[{data: textContent}]`（TXT，兼容`$json.data`）。

**教训：** 不要在n8n参数里写复杂JS表达式。需要动态计算 → 先用Code节点算好再传。

---

## 7.3 n8n cron调度器的坑

### 症状

定时推送工作流运行几天后全部静默失效，手动触发正常。

### 根因

n8n 2.20.11 的cron调度器有bug——它会反复注销已注册的cron job，导致定时任务在运行几天后悄然停止。

### 修复

彻底放弃n8n调度，迁移到系统crontab + Python直推：

```bash
# 采集守护进程（Python，持续运行）
@reboot python3 /path/to/collector_daemon.py

# 推送脚本（crontab定时触发）
0 7 * * *  python3 push_early_report.py   # 早报
0 19 * * * python3 push_hotsearch.py      # 热搜
30 20 * * * python3 push_gold.py          # 贵金属
0 23 * * * python3 push_daily_review.py   # 复盘
0 12,21 * * * python3 push_circle_intel.py # 圈层情报(2次/天)
0 22 * * 4 python3 push_weekly_movie.py   # 每周影视
0 20 * * 5 python3 push_weekly_report.py  # 每周综合报告
```
