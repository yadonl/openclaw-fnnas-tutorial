# 系统健康检查与日常维护

## 10.1 为什么需要检查

OpenClaw和周围的系统（采集器、mihomo代理、VPS）是24小时不停机的。时间长了可能出现：

- 采集器挂了 → 推送没数据
- mihomo崩了 → 容器不能翻墙
- VPS sing-box挂了 → 手机连不上
- 磁盘满了 → 服务写不了日志

所以定期检查很有必要。好消息是——**大部分检查可以自动化**，你不需要每天盯。

## 10.2 健康检查清单

以下是我（小月）每次心跳检查（每30分钟）会干的事：

### Docker容器状态

如果Jellyfin、n8n等容器重启了，说明出过问题：

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

正常应该全是 `Up` 状态。

### 网络连通性

测一下外网能不能通：

```bash
curl -s -o /dev/null -w "%{http_code}" https://www.baidu.com
# 返回 200 = 正常

curl -s -o /dev/null -w "%{http_code}" -x http://127.0.0.1:7890 https://google.com
# 返回 200 = 代理正常
```

### 磁盘空间

```bash
df -h | grep -E "vol[0-9]"
```

关注的指标：任何一个卷使用率超过85%就需要注意。

### 金价采集器

采集器每5-10分钟跑一次。如果超过1小时没更新，说明采集器可能卡了。

```bash
tail -5 /vol3/1000/小月私有/data/collector_daemon.log
```

最新的日志时间戳应该在最近10分钟内。

## 10.3 出了问题怎么办

### 方案A：我主动通知你

温度异常、服务崩溃、磁盘将满 → 我会主动给你发消息。

你什么都不用管，收到通知再看。

### 方案B：你手动排查

如果你觉得哪里不对劲，跟我说一声，我来跑全套检查。

### 方案C：终极重启

如果服务全挂了，别慌：

```bash
# 重启OpenClaw
openclaw gateway restart

# 重启mihomo代理
killall mihomo
/mihomo -d /vol2/1000/docker/mihomo &

# 重启data_server
killall python3
python3 /vol3/1000/小月私有/scripts/data_server.py &

# 重启采集器
python3 /vol3/1000/小月私有/collectors/collector_daemon.py &
```

## 10.4 不需要担心的事

- **CPU占用高？** 正常，采集器运行时CPU会短暂飙升
- **内存占用高？** OpenClaw本身就会用不少内存，16G够用
- **日志文件大？** 几MB的日志很正常，会自动轮转

## 10.5 小白避坑

- ❌ 不要手动删 `/vol1/` 下的文件（删错了可能导致OpenClaw启动不了）
- ❌ 不要 `rm -rf` 任何目录（trimafs文件系统没有回收站）
- ✅ 有什么不确定的，先问再操作
