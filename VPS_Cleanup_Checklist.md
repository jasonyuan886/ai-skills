# VPS 清理操作清单

> **目标**：磁盘从 96% 降至安全水位（<80%），清理 Claude Code 遗留物，保留所有业务资产。
>
> **⚠️ 操作原则**：先备份再删，不确定就保留。共享基础设施服务（xray/socat/socks5_relay）一律不动。

---

## 一、【保留】— 不要动

### 1.1 核心业务目录（/root/projects/）
```bash
# 以下全部保留，不删除任何文件
/root/projects/manager/         # 总管事（除 .last_* 和 .bak-* 外）
/root/projects/chinapal/        # ChinaPal 项目
/root/projects/freshlock/       # B2B 获客
/root/projects/freshlock-web/   # 独立站
/root/projects/xuanpin/         # 跨境选品
/root/projects/infra/           # 基础设施文档
/root/projects/design/          # 产品设计
/root/projects/chinapal_world/  # 静态站
```

### 1.2 系统服务（全部保留运行）
```bash
# 不要 stop/disable 任何以下服务
nginx, xray, socks5-relay, res-socks, res-http, socat-2082,
tinyproxy, postfix, dovecot, tracker, bazi-api, agent-chat,
freshlock-gateway, proxy-monitor, warp-svc, sshd, cron
```

### 1.3 凭据和配置
```bash
/root/credentials_env.sh        # 核心凭据
/root/.claude/CLAUDE.md         # 全局规则
/root/.claude/CLOUD_PHONE.md    # 云手机手册
/root/.claude/uploads/          # 规格文档
/root/.claude/projects/         # 记忆文件
```

### 1.4 自动化脚本（仍在运行的）
```bash
/root/scripts/                  # health_check, resource_monitor, cloudphone_watchdog
/root/automation/               # socks5_relay.py
/usr/local/bin/relay_watchdog.sh
/usr/local/bin/proxy_keepalive.sh
/root/projects/manager/backup.sh
/root/projects/chinapal/story_pipeline.py
/root/projects/chinapal/era_membership_daily.py
/root/projects/xuanpin/baopin_radar/radar.py
/root/projects/freshlock/automation/    # B2B 获客全套
/root/make_video.py
```

### 1.5 正在使用的 cron（保留）
```bash
# 以下 cron 条目保留不动
relay_watchdog.sh, proxy_keepalive.sh, resource_monitor.sh,
cloudphone_watchdog.sh, logrotate syslog-cap, backup.sh,
story_pipeline.py, era_membership_daily.py, b2b_cold_sender.py,
radar.py, health_check.sh, evolution_check.sh, fetch_trends.py,
domain_warmup_chinapal.sh, direct_send.py
```

---

## 二、【备份后清理】— 先确认再删

### 2.1 Claude Code 二进制（约 50-100MB）
```bash
# 确认：账号已封，不再需要
which claude
ls -lh /usr/bin/claude

# 清理：
rm -f /usr/bin/claude
```
**原因**：Claude 账号封禁，二进制无用

### 2.2 __pycache__ 目录
```bash
# 查看大小
du -sh /root/projects/*/__pycache__/ 2>/dev/null

# 清理（安全，Python 会自动重建）
find /root/projects/ -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null
```
**原因**：Python 字节码缓存，删除无影响

### 2.3 chrome_profile_* 目录（可能数 GB）
```bash
# 查看大小
du -sh /root/projects/*/chrome_profile_* 2>/dev/null

# 清理
rm -rf /root/projects/*/chrome_profile_* 2>/dev/null
```
**原因**：Playwright/Chromium 浏览器缓存，不再使用浏览器自动化时可清理

### 2.4 distill 项目 venv（实验项目，可能较大）
```bash
# 查看大小
du -sh /root/projects/distill/venv/

# 清理（实验项目，Claude 已封不再运行）
rm -rf /root/projects/distill/venv/
# 如需保留项目文档，只删虚拟环境：
# rm -rf /root/projects/distill/venv/  # 仅删 venv，保留 CLAUDE.md 等文档
```
**原因**：AI 蒸馏是实验性项目，Claude 封禁后不再运行

### 2.5 freshlock-web venv（可重建）
```bash
# 查看大小
du -sh /root/projects/freshlock-web/venv/

# 清理（需要时可 pip install 重建）
rm -rf /root/projects/freshlock-web/venv/
```
**原因**：Python 虚拟环境，可从 requirements 重建

### 2.6 automation_backup_* 目录
```bash
# 查看
ls -la /root/projects/freshlock-web/automation_backup_*/

# 清理（旧备份，已有 git 历史保障）
rm -rf /root/projects/freshlock-web/automation_backup_*/
```
**原因**：旧的自动化备份，git 仓库已有完整历史

---

## 三、【直接清理】— 无风险，直接删

### 3.1 Claude 机制状态文件
```bash
rm -f /root/projects/manager/.last_skillcheck_*
rm -f /root/projects/manager/.last_alert_*
rm -f /root/projects/manager/.last_nudge
rm -f /root/projects/manager/.last_resume_*
```
**原因**：queue_check.sh / skill_check.sh 的冷却计时器，Claude 停用后无意义

### 3.2 TASK_QUEUE.md 旧备份
```bash
rm -f /root/projects/manager/TASK_QUEUE.md.bak-*
```
**原因**：历史快照，git 历史中已有

### 3.3 0 字节空日志
```bash
find /root/projects/ -name "*.log" -empty -delete 2>/dev/null
find /root/ -maxdepth 1 -name "*.log" -empty -delete 2>/dev/null
```
**原因**：空文件无价值

### 3.4 Claude settings.json
```bash
rm -f /root/.claude/settings.json
```
**原因**：Claude Code 专属权限配置，新 AI 不需要

### 3.5 临时文件
```bash
rm -f /tmp/ai-skills-clone/test_write.txt
rm -rf /tmp/ai-skills-clone/
```

---

## 四、需要停用的 cron 条目

```bash
# 备份当前 crontab
crontab -l > /tmp/crontab_before_cleanup.txt

# 需要注释掉的条目（在行首加 #）：
# 1. */30 * * * * /root/projects/manager/queue_check.sh     ← Claude 续接
# 2. 5 5 * * * /root/projects/manager/trend_scan.sh         ← Claude 热点扫描
# 3. 30 6 * * * /root/projects/manager/skill_check.sh       ← Claude 技能盘点
# 4. 30 7 * * * /root/projects/chinapal/daily_task.sh       ← 调 claude -p
# 5. 37 11 * * 1,3,5 /root/projects/freshlock/automation/weekly_lead_gen.sh  ← 含 claude -p
```

**操作命令**：
```bash
crontab -l > /tmp/crontab_new.txt
# 编辑 /tmp/crontab_new.txt，注释以上 5 行
crontab /tmp/crontab_new.txt
```

---

## 五、需要停用的 systemd 服务

### 5.1 可考虑停用
```bash
# distill-api（实验项目，Claude 封禁后无意义）
systemctl stop distill-api.service 2>/dev/null
systemctl disable distill-api.service 2>/dev/null

# res-http（端口 9181，曾用于代理转发，评估是否仍需）
# 注意：先确认没有脚本依赖 9181 端口
# systemctl stop res-http.service
# systemctl disable res-http.service
```

### 5.2 已知的僵尸服务
```bash
# res-http（TASK_QUEUE 记录中提到应清理，但未执行）
# 先检查是否有流量：
ss -tlnp | grep 9181
# 如果无业务流量，可停用
```

---

## 六、清理后验证

```bash
# 1. 检查磁盘释放了多少
df -h /
# 目标：< 80%（< 15.2G used）

# 2. 检查内存
free -h

# 3. 确认关键服务仍在运行
systemctl list-units --type=service --state=running | grep -E "nginx|xray|postfix|socks5|backup"

# 4. 确认 cron 已更新
crontab -l | grep -v "^#" | grep -v "^$"

# 5. 确认备份仍正常
tail -3 /var/log/backup.log
```

---

## 七、清理优先级（按释放空间排序）

| 优先级 | 操作 | 预计释放 |
|---|---|---|
| 1 | chrome_profile_* | 可能 1-3 GB |
| 2 | distill/venv + freshlock-web/venv | 可能 500MB-1GB |
| 3 | Claude 二进制 | ~50-100MB |
| 4 | __pycache__/ | ~10-50MB |
| 5 | automation_backup_* | ~10-50MB |
| 6 | .last_* / .bak-* / 空日志 | <10MB |

**总计预计释放**：2-5 GB，磁盘使用率从 96% 降至 ~75-85%

---

*本清单由袁小助生成，操作前请确认每项内容。不确定就保留，优先保障业务连续性。*
