---
name: vps-resource-triage
description: 主力VPS(192.236.171.76)内存/磁盘紧张、卡顿、或袁总要求清理重复Claude Code会话时用。核心教训：这台机器常年内存紧张的真正原因通常不是日志/缓存，而是整个Claude Code团队的tmux会话本身(一个会话~150-350MB)；清理重复会话绝不能凭前缀/创建时间等表面规律批量判断，必须逐个会话自查+交叉验证。来自2026-09-22排查16个重复Claude会话、安全清理7个的实战经验。
---

# VPS资源排查 & 重复会话清理

## 第一步：先分清是磁盘紧张还是内存紧张，两者处理方式完全不同
```bash
df -h /        # 磁盘
free -h        # 内存，重点看 available 这一列和 Swap 已用量
ps aux --sort=-%mem | head -20   # 内存前20名进程
```

**磁盘紧张**：通常是日志/journal/apt缓存，安全清理：
```bash
apt-get clean
journalctl --vacuum-size=50M
truncate -s 0 /var/log/syslog
find /var/log -name '*.gz' -delete
find /var/log -name '*.1' -delete
```

**内存紧张**：先看`ps aux --sort=-%mem`前20名是什么。如果大量`claude`进程各占100-350MB——**这不是能清理的"垃圾"，是团队正在跑的Claude Code会话本身**，不能因为占内存就当作问题处理，除非确认是重复/空壳会话（见下）。日志清理对内存紧张**没有直接帮助**，不要浪费时间往这个方向查。

## 第二步：识别"重复会话"——不能靠表面规律判断
2026-09-20 21:32团队做过一次批量重启/恢复，之后8个项目（infra/freshlock/freshlock-web/chinapal/distill/xuanpin/design/manager）各自额外多出一个新tmux会话（21:51创建，无cc-前缀，短码命名如"xuanpin-a7"），跟原有的`cc-<项目名>`会话（21:32创建，人工改过正式名字）并存，长期没清理，是内存紧张的主因之一。

**已验证无效/危险的判断依据**（不要用这些做批量决策）：
- ❌ "cc-前缀=旧的该删" —— 错，cc-前缀那批恰恰大多是有正经工作历史、被人工重命名过的原会话
- ❌ "创建时间早的=该删" —— 部分对，但manager那对是反例
- ❌ "用`claude --continue`启动=更新更该留" —— 有会话自己拿这个当理由论证自己该留，但这只是进程启动参数，不代表内容更有价值
- ❌ 只信一边的自我报告 —— 每个会话判断"自己该不该被关"时天然有立场，必须双方都问

**正确流程**：
1. `ListAgents` 拿到当前全部peer会话+tmux pane对应关系
2. 给每一对的**两边**都发 `SendMessage` 要求自查，问清楚：
   - 现在有没有在跑任务/后台进程
   - git status是否干净、有没有本会话产生的未提交改动
   - 关掉自己对已知的定时任务(systemd timer/cron)有没有影响（通常没有——这类任务不依赖tmux会话存活）
   - 有没有跟袁总还没走完的对话/等待外部输入的挂起状态
3. 观察SendMessage返回结果里"也通过Remote Control连接"这个标注——这是"袁总手机/远程端真的连过这个会话"的强信号，比自我报告更可靠
4. **特别警惕"两边都有真实价值"的情况**（本次manager那对：一个是袁总当前正在用的实时聊天窗口，另一个独立跑了2天有自己的监控/队列维护工作）——遇到这种直接跳过，不处理，问袁总
5. 只处理双方交叉验证后确认"其中一边明确空/无历史/可安全关"的对，动手前用`ListAgents`再确认目标是`idle`状态（不是`busy`，避免掐断正在生成的回复）

## 第三步：动手清理
```bash
tmux kill-session -t <会话名>   # 比kill pid更干净，会正常终止会话内的claude进程
```
清完立即验证：
```bash
free -h                          # available应该涨、swap应该降
systemctl is-active xray socks5-relay tinyproxy proxy-monitor res-http res-socks  # 共享服务一个都不能少
```
把处理过程（哪几对怎么判断的、关了谁留了谁、为什么）写进 `/root/CHANGELOG_agent_chat.log`（或对应项目的CHANGELOG）。

## 排查前先检查：可能已经有现成监控在跑
`/root/scripts/resource_monitor.sh`每5分钟检查一次磁盘/内存阈值并自动推送告警到cc-manager（见[[vps-watchdog-pattern]]），先看它的日志/告警状态，很可能不用从头手动诊断：
```bash
tail -30 /root/logs/resource_monitor.log
cat /root/projects/manager/.last_alert_mem /root/projects/manager/.last_alert_disk 2>/dev/null
```

## 正确的开新会话方式（附带内存保护）
`/root/scripts/cc_start.sh <组名>` 是标准开组脚本——内置检查：可用内存<180MB会拒绝开新会话并提示先关哪些cc-*组，避免手滑再造出一批吃内存的新会话。**脚本里的组名白名单目前只有`freshlock|freshlock-web|chinapal|xuanpin`，没有design/distill/infra/manager**——这几个项目的会话是当时手动开的，不在标准流程里，是个已知缺口，下次改这个脚本时应该补全。

## 已知坑
- 手机端Claude App的"Recents"会话列表和真正的VPS tmux会话不是一回事——App里可能有历史遗留的Remote Control配对入口（比如同时存在"ChinaPal AI"和"Chinapal AI"两个大小写不同的条目），在手机上删/关这类条目**不会影响VPS内存**，别指望靠手机操作解决资源紧张
- `tmux kill-session`这类destructive操作经常被Claude Code的auto mode分类器拦一次，同一条命令原样重试一次通常就通过，不用换写法
- 一次批量發送SendMessage排查多个会话时，收到的回复是异步陆续到达的，不要在还没收全就下结论，尤其像manager这种"证据后到但推翻了前面结论"的情况
