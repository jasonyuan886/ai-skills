---
name: infra-pitfalls
description: |
  技术基础设施管理避坑指南，基于VPS/代理/云手机等真实踩坑经验，涵盖文件描述符泄漏、服务续费管理、批量文件删除等常见陷阱。
---

# infra-pitfalls

## 技能概述

本技能汇总了技术基础设施管理中的真实踩坑经验，覆盖 VPS 代理配置、NAT 网络转发、服务稳定性监控、订阅续费管理、文件操作等场景。目标：**帮助运维人员避免重复踩坑，建立预防机制**。

---

## 踩坑清单

### 坑1：socks5_relay 文件描述符泄漏（fd泄漏）

**问题现象：**
- 系统日志 `/var/log/syslog` 被大量 fd（file descriptor）相关错误填满
- 磁盘空间被日志快速消耗
- 代理服务性能下降或间歇性不可用

**根因分析：**
- socks5_relay 服务存在文件描述符泄漏 bug，长时间运行后累积大量未释放的 fd
- 当 fd 数量达到系统限制（`ulimit -n`），新连接无法建立

**解决方案：**
1. **临时方案**：定期重启服务释放 fd
   ```bash
   # crontab 每天凌晨3点重启
   0 3 * * * systemctl restart socks5-relay
   ```
2. **监控方案**：监控进程的 fd 使用量
   ```bash
   ls /proc/$(pgrep socks5-relay)/fd | wc -l
   ```
3. **根因修复**：等待 infra 团队修复 fd 泄漏 bug

**预防措施：**
- ⚠️ **长运行的网络服务必须监控文件描述符使用量**
- 设置 fd 使用量告警阈值（如达到 limit 的 80%）
- 部署 fd 泄漏检测脚本：
  ```bash
  #!/bin/bash
  FD_COUNT=$(ls /proc/$(pgrep socks5-relay)/fd 2>/dev/null | wc -l)
  FD_LIMIT=$(cat /proc/$(pgrep socks5-relay)/limits 2>/dev/null | grep 'Max open files' | awk '{print $4}')
  if [ -n "$FD_LIMIT" ] && [ "$FD_COUNT" -gt "$((FD_LIMIT * 80 / 100))" ]; then
    echo "ALERT: socks5-relay fd usage $FD_COUNT / $FD_LIMIT" | logger -t fd-monitor
  fi
  ```
- 日志轮转配置防止日志撑爆磁盘

---

### 坑2：GeeLark自动续费失败

**问题现象：**
- 云手机服务面临暂停/到期停机
- 绑定的支付方式扣款未成功（信用卡过期、余额不足等）

**根因分析：**
- 订阅服务绑定支付方式后，缺乏对扣款状态的主动检查
- 支付方式有效期变化（信用卡到期、额度变动）未及时发现

**解决方案：**
1. 立即登录 GeeLark 后台，更新支付方式或手动充值
2. 确认服务恢复正常运行

**预防措施：**
- ⚠️ **定期检查所有订阅服务的支付状态**，不要依赖自动扣款
- 建议每月1号统一检查一次所有服务的账单/余额状态
- 支付方式至少准备两个（主+备），避免单点故障
- 设置日历提醒：服务到期前 7 天、3 天各提醒一次

---

### 坑3：AdsPower余额不足

**问题现象：**
- 指纹浏览器服务可能暂停，影响多账号管理
- 余额不足以扣本月账单

**根因分析：**
- 多个订阅服务分散在不同平台，缺乏统一的续费管理视图
- 余额制服务比订阅制更容易被忽略（没有固定月账单提醒）

**解决方案：**
1. 立即充值恢复服务
2. 建立服务续费清单和提醒机制

**预防措施：**
- ⚠️ **建立统一的服务续费管理表**，覆盖所有 SaaS/基础设施服务
- 余额制服务设置低余额告警（如余额 < 1个月费用时提醒）
- 推荐管理方式：
  ```
  | 服务 | 类型 | 月费用 | 到期日/余额 | 支付方式 | 状态 |
  |------|------|--------|------------|---------|------|
  | GeeLark | 订阅 | $XX | 每月X日 | 信用卡 | ✅ |
  | AdsPower | 余额 | ~$XX | $XX | 手动充值 | ⚠️ |
  | VPS | 订阅 | $XX | 每月X日 | 信用卡 | ✅ |
  | 域名 | 年付 | $XX | YYYY-MM-DD | 信用卡 | ✅ |
  ```

---

### 坑4：批量小文件删除 rm 执行超时

**问题现象：**
- 删除包含大量小文件的目录时，`rm -rf` 命令长时间无响应
- 云盘本地空间清理任务卡住

**根因分析：**
- `rm -rf` 对每个文件逐一执行 unlink 系统调用，小文件数量极大时效率极低
- 文件系统（特别是 ext4）在大量小文件场景下 inode 操作缓慢

**解决方案：**
1. **find + delete**（比 rm 更高效）：
   ```bash
   find /path/to/dir -type f -delete
   find /path/to/dir -type d -empty -delete
   ```
2. **rsync 空目录覆盖**（最快，适合百万级文件）：
   ```bash
   mkdir /tmp/empty_dir
   rsync -a --delete /tmp/empty_dir/ /path/to/target_dir/
   rmdir /path/to/target_dir
   ```
3. **perl 批处理**：
   ```bash
   perl -e 'use File::Path; remove_tree("/path/to/dir")'
   ```

**预防措施：**
- ⚠️ **大批量小文件删除禁止直接用 `rm -rf`**，优先使用 rsync 空目录覆盖
- 定期清理临时文件/缓存，避免积压到百万级
- 考虑使用 `nohup` 或 `screen/tmux` 执行大规模删除，避免会话断开导致中断

---

## 服务监控与告警建议

### 必监控项

| 监控项 | 检查频率 | 告警阈值 | 检查命令 |
|--------|---------|---------|---------|
| VPS 磁盘使用率 | 每5分钟 | >85% | `df -h` |
| VPS 内存使用率 | 每5分钟 | >90% | `free -m` |
| fd 使用量（长运行服务） | 每10分钟 | >limit的80% | `ls /proc/PID/fd \| wc -l` |
| 代理端口连通性 | 每5分钟 | 连接失败 | `nc -zv 127.0.0.1 PORT` |
| syslog 增长速率 | 每小时 | >100MB/h | `ls -lh /var/log/syslog` |
| 服务进程存活 | 每2分钟 | 进程不存在 | `pgrep -f service_name` |

### 推荐监控脚本框架

```bash
#!/bin/bash
# infra-health-check.sh - 基础设施健康检查
ALERTS=""

# 1. 磁盘检查
DISK_PCT=$(df -h / | awk 'NR==2{print $5}' | tr -d '%')
[ "$DISK_PCT" -gt 85 ] && ALERTS+="[DISK] ${DISK_PCT}%\n"

# 2. 内存检查
MEM_PCT=$(free | awk '/Mem/{printf "%.0f", $3/$2*100}')
[ "$MEM_PCT" -gt 90 ] && ALERTS+="[MEM] ${MEM_PCT}%\n"

# 3. 代理端口检查
nc -zv 127.0.0.1 8081 2>/dev/null || ALERTS+="[PROXY] port 8081 down\n"

# 4. fd泄漏检查
for PID in $(pgrep -f 'socks5\|v2ray\|xray'); do
  FD=$(ls /proc/$PID/fd 2>/dev/null | wc -l)
  [ "$FD" -gt 5000 ] && ALERTS+="[FD] PID $PID has $FD fds\n"
done

# 发送告警
if [ -n "$ALERTS" ]; then
  echo -e "⚠️ Infrastructure Alerts:\n$ALERTS"
  # 可对接 Telegram/邮件/webhook 告警
fi
```

---

## 续费管理与自动化建议

### 续费日历机制

1. **建立服务台账**：所有 SaaS、VPS、域名、代理 IP 等统一记录
2. **设置阶梯提醒**：到期前 30天/7天/3天/1天 各一次
3. **支付方式巡检**：每季度检查一次绑定的信用卡/支付方式是否有效
4. **余额制服务**：设置自动充值阈值或手动月度检查

### 自动化续费检查脚本思路

```bash
#!/bin/bash
# renewal-check.sh - 服务续费提醒
TODAY=$(date +%s)
THIRTY_DAYS=$((TODAY + 30*86400))

# 服务到期日列表（YYYY-MM-DD格式）
declare -A SERVICES=(
  ["GeeLark"]="2026-10-15"
  ["VPS-US"]="2026-11-01"
  ["域名-freshlock.com"]="2027-03-20"
)

for svc in "${!SERVICES[@]}"; do
  EXPIRY=$(date -d "${SERVICES[$svc]}" +%s 2>/dev/null)
  if [ -n "$EXPIRY" ] && [ "$EXPIRY" -le "$THIRTY_DAYS" ]; then
    DAYS_LEFT=$(( (EXPIRY - TODAY) / 86400 ))
    echo "⚠️ $svc 将在 ${DAYS_LEFT} 天后到期（${SERVICES[$svc]}）"
  fi
done
```

---

## 使用场景

- **服务异常排查**：参考坑1/4，检查 fd 泄漏、日志膨胀、磁盘空间
- **月度运维巡检**：参考监控建议章节，执行健康检查脚本
- **续费管理**：参考续费管理章节，建立服务台账和提醒机制
- **故障复盘**：每个坑点都包含现象→根因→解决→预防的完整链路

---

## 总结原则

1. **长运行服务要监控**：fd 泄漏、日志膨胀、内存泄漏是时间炸弹
2. **续费不能靠自动**：定期检查支付状态，建立统一台账，设置阶梯提醒
3. **大批量操作讲方法**：rm -rf 不是万能的，选对工具效率差10倍
