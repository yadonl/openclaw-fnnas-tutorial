# 🦞 飞牛NAS上玩转OpenClaw：真实踩坑到生产环境的全记录

> 一个普通人在FeiNiu NAS上从零搭建OpenClaw + 自动化推送 + 三机联动的真实经历  
> 没有理论，全是实战——踩过的坑、修复的方案、当前的稳定架构

[![GitHub stars](https://img.shields.io/github/stars/yadonl/openclaw-fnnas-tutorial?style=social)](https://github.com/yadonl/openclaw-fnnas-tutorial)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

---

## 📖 关于这个教程

这不是官方文档的复读，而是一个"自己摸出来的"实战记录。

作者在完全不懂的情况下，靠折腾把OpenClaw跑成了自己的员工——能自动采集、定时推送、24小时不关机。

**总字数：** ~12,000字  
**实战经验：** 30+个真实踩坑记录  
**适合人群：** NAS玩家、折腾党、想用AI自动化的个人用户

---

## 📚 目录

### 第一篇：环境搭建篇

| 章节 | 内容 | 适合谁看 |
|------|------|---------|
| [第1章：飞牛NAS上部署OpenClaw](docs/01-basics/01-deploy-fnnas.md) | 应用商店安装、配置面板介绍、飞牛版特殊坑 | 刚下载完OpenClaw的新手 |
| [第2章：打通Telegram通道](docs/01-basics/02-config-telegram.md) | Bot注册、dmPolicy配置、代理设置、常见问题 | 想让Bot回复你的人 |
| [第3章：模型选择与配置](docs/01-basics/03-model-config.md) | DeepSeek V4 Flash、API Key配置、预算控制 | 想选最省钱方案的人 |

### 第二篇：架构篇

| 章节 | 内容 | 适合谁看 |
|------|------|---------|
| [第4章：三机联动架构](docs/02-core/01-architecture.md) | AI算力机+调度机+NAS的角色分工 | 有多台设备想联动的人 |

### 第三篇：网络篇

| 章节 | 内容 | 适合谁看 |
|------|------|---------|
| [第5章：VPS代理搭建](docs/03-network/01-vps-proxy.md) | VPS选购、sing-box配置、Shadowsocks、mihomo代理、血泪教训⚠️ | 想自建代理、替代机场的人 |
| [第6章：Shadowrocket + iOS快捷指令](docs/03-network/02-shadowrocket.md) | 小火箭配置、出门/在外/回家三个一键切换 | 手机用户 |

### 第四篇：自动化篇

| 章节 | 内容 | 适合谁看 |
|------|------|---------|
| [第7章：n8n工作流迁移实录](docs/02-key-lessons.md) | Code节点禁fs、URL不能用$today、cron调度器bug | 用n8n做自动化的人 |
| [第8章：自建采集与推送系统](docs/04-auto/01-push-system.md) | collector_daemon持续采集、data_server中转、7个crontab推送 | 想做信息聚合推送的人 |

### 第五篇：运维篇

| 章节 | 内容 | 适合谁看 |
|------|------|---------|
| [第9章：系统健康检查](docs/05-ops/02-health-check.md) | 自动检查清单、服务重启方案、小白避坑 | 担心系统稳定性的人 |
| [第10章：备份与恢复](docs/05-ops/01-backup.md) | 飞牛备份限制、OpenClaw数据目录、自动脚本、恢复方法 | 怕数据丢了的人 |

---

## 🛠 踩坑合集（快速索引）

| 坑 | 症状 | 原因 | 修复 |
|----|------|------|------|
| Tailscale炸网 | 全部服务挂掉 | `--accept-routes` 路由污染 | VPS和NAS路由分离 |
| n8n Code崩 | 工作流报fs模块禁止 | 沙箱环境限制 | HTTP Request替代 |
| n8n URL表达式失效 | 报"URL must start with http" | HTTP Request节点不支持JS | Code节点fetch()替代 |
| n8n定时器失踪 | 推送静默失效 | cron调度器bug | 迁移到系统crontab |
| Bot不回复 | 发消息没反应 | dmPolicy没配 | 设成"pairing" |
| 金价显示0 | 数据全零 | data_server缓存旧数据 | 手动触发采集刷新 |
| Telegram连接失败 | 国内网络不通 | 代理没配置 | 填channels.telegram.proxy |

---

## ⭐ 如果这个教程对你有帮助

- 点个Star ⭐ 让更多人看到
- 提Issue分享你的踩坑经历
- Fork后改成你自己的版本

---

## 📝 许可证

[MIT License](LICENSE)
