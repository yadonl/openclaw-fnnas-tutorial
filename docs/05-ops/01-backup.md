# 备份与恢复

## 11.1 飞牛自带的备份能备份什么

飞牛NAS的系统设置里有个「备份和还原」功能，能导出系统配置（`.bin` 文件）。

**✅ 能备份的：** 账号、权限、网络权限、共享文件夹设置这类系统层面的东西。

**❌ 不能备份的：** OpenClaw 内部的工作流、技能、配置、记忆——这些飞牛根本不知道是什么。

所以光靠飞牛系统备份是不够的，得额外把 OpenClaw 的数据单独打包备份。万一哪天NAS挂了、硬盘坏了、或者你想换机器，系统备份能帮你恢复NAS本身，但OpenClaw还得靠手动备份来恢复。

---

## 11.2 OpenClaw 必须备份的目录

OpenClaw 的所有"灵魂"数据都在这两个文件夹里：

| 目录 | 大小（参考） | 说明 |
|------|-------------|------|
| `/vol1/@apphome/trim.openclaw/data/workspace/` | 约 89MB | 技能、脚本、记忆、文档 |
| `/vol1/@apphome/trim.openclaw/data/openclaw/` | 约 710MB | 运行时数据、会话记录、日志 |

再加一个主配置文件（很小，就 1KB）：

```
/vol1/@apphome/trim.openclaw/data/home/.openclaw/openclaw.json
```

**就是这么简单。**

只要这三样东西在，哪怕NAS系统崩溃、换新机器、重装系统，也能100%恢复所有功能。技能、工作流、记忆、对话历史，全部一个不落。

---

## 11.3 自动备份脚本

写个简单的 shell 脚本，每周自动打包备份。新建一个文件，比如 `/vol1/1000/备份/backup_openclaw.sh`：

```bash
#!/bin/bash
# OpenClaw 自动备份脚本

BACKUP_DIR="/vol1/1000/备份"
DATE=$(date +%Y-%m-%d)

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 打包备份
tar -czf "${BACKUP_DIR}/openclaw-backup-${DATE}.tar.gz" \
  /vol1/@apphome/trim.openclaw/data/home/.openclaw/openclaw.json \
  /vol1/@apphome/trim.openclaw/data/workspace \
  /vol1/@apphome/trim.openclaw/data/openclaw

# 删除30天前的旧备份
find "$BACKUP_DIR" -name "openclaw-backup-*.tar.gz" -mtime +30 -delete

echo "备份完成：openclaw-backup-${DATE}.tar.gz"
```

加执行权限：

```bash
chmod +x /vol1/1000/备份/backup_openclaw.sh
```

关于容量你也不用担心：压缩后大概 **167MB** 左右，每周一次，一年下来不到 9GB。放在NAS上完全没压力。

---

## 11.4 设置自动定时备份

用 `crontab` 设置每周日凌晨 3 点自动跑：

```bash
crontab -e
```

添加这一行：

```
0 3 * * 0 bash /vol1/1000/备份/backup_openclaw.sh
```

几个参数说明：
- `0 3` — 凌晨3点执行
- `* * 0` — 每周日（第0天）
- 脚本里的 `-mtime +30` 自动删除超过30天的旧备份

这样设置好之后，你**完全不用管它**。每周自动备份，自动清理旧的，稳得很。

---

## 11.5 恢复方法

重装系统后，三步就能恢复：

**第一步：** 重新在飞牛应用商店安装 OpenClaw（装好就行，先别急着用）

**第二步：** 停止 OpenClaw 服务。在 OpenClaw 后台或命令行里停掉它，避免恢复时文件被占用。

**第三步：** 解压备份文件，覆盖回原目录：

```bash
# 假设备份文件在 /vol1/1000/备份/
tar -xzf /vol1/1000/备份/openclaw-backup-2026-05-25.tar.gz -C /
```

这条命令会把三个备份项原样覆盖回去。

**第四步：** 重新启动 OpenClaw 服务。

等它启动完成，你之前所有的技能、工作流、记忆、对话历史，全部恢复原样，跟备份前一模一样。

如果恢复后发现有些功能不对，检查一下文件权限：

```bash
chown -R 1000:1000 /vol1/@apphome/trim.openclaw/
```

> 💡 **小贴士：** 建议每年至少手动验证一次备份文件能不能正常解压。别等到真需要恢复的时候才发现备份包坏了——那可就尴尬了。
