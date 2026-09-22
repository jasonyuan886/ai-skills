---
name: vps-watchdog-pattern
description: 需要给VPS上某个资源/服务加"自动探测故障+自愈/预警"机制时用——比如按分钟计费的外部资源(云手机)可能因会话撞额度/异常退出而忘记善后一直烧钱，或者某个共享服务(xray/socks5-relay等)有已知的偶发故障模式需要自动恢复。也是VPS共享铁律"禁止重启xray/socks5-relay"这条规则**已验证例外**的边界说明。排查磁盘/内存紧张前也先看这里——很可能已经有现成监控在跑，不用重新诊断。来自2026-09-13云手机看门狗+现有proxy_keepalive/resource_monitor/health_check实战经验。
---

# VPS看门狗(Watchdog)脚本模式

这台VPS上已经有5个在跑的监控/看门狗脚本，覆盖了大部分常见场景。**新需求先看这里有没有现成的能扩展，不要重新发明；排查磁盘/内存/服务异常前，先看这些脚本的日志，可能已经有记录，不用从头手动诊断。**

## 现有实现总览
| 脚本 | 频率 | 类型 | 做什么 |
|---|---|---|---|
| `/root/scripts/health_check.sh` | 每天8:07 | 只读监控 | 磁盘/内存/xray存活/tmux会话(cc-chinapal/cc-freshlock/cc-freshlock-web/cc-xuanpin/cc-infra/cc-manager)/claude进程/22端口，写`/root/logs/health_<日期>.log`，异常置`/root/logs/LAST_ALERT.flag` |
| `/root/scripts/resource_monitor.sh` | 每5分钟 | 只读监控+告警 | 磁盘≥80%或可用内存≤10%时，tmux推消息给cc-manager（1小时冷却，状态存`/root/projects/manager/.last_alert_<key>`），日志`/root/logs/resource_monitor.log` |
| `/usr/local/bin/proxy_keepalive.sh` | 每5分钟 | 自愈(会重启服务) | 检查xray/tinyproxy/socat-2082/res-socks/res-http/socks5-relay是否active，down了就`systemctl restart`；还会实际curl测试代理能不能连通(不只看进程活着)，连不通也重启；CLOSE-WAIT连接数>50也重启relay。日志`/var/log/proxy_keepalive.log` |
| `/usr/local/bin/relay_watchdog.sh` | 每2分钟 | 自愈(重启单个服务) | 专门盯1082端口的socks5-relay fd泄漏(fd>800或无监听→重启)，是`proxy_keepalive.sh`之外**额外的一层**，不是唯一的 |
| `/root/scripts/cloudphone_watchdog.sh` | 每10分钟 | 自愈(调用外部API) | GeeLark云手机开机超60分钟强制`/phone/stop`关机，防止会话撞额度忘关机白烧钱；EXCLUDE名单排除不归自己管的设备 |

**排查前先看**：
```bash
tail -30 /root/logs/resource_monitor.log        # 磁盘/内存紧张，是不是已经告警过/什么时候开始的
cat /root/logs/LAST_ALERT.flag 2>/dev/null      # health_check有没有标过异常
tail -30 /var/log/proxy_keepalive.log           # 代理服务是不是刚被自动重启过
tail -30 /root/logs/cloudphone_watchdog.log     # 云手机计费异常先看这个
```

## 通用模式（新场景照这个抄）
```
cron高频轮询(2-10分钟，看资源变化速度定)
  → 查询当前真实状态(API/ss/ps/实际连通性测试，不要只看进程存在就当健康)
  → 跟状态文件里记的上次状态比较，判断"是否已超阈值"
  → 超阈值才动手remediate(关机/重启/清理)，没超就只记状态或什么都不做
  → 只读监控用冷却时间(如1小时)防止重复刷屏告警
  → 动作发生后写日志(时间戳+对象+动作+结果)
  → 如果动作会影响到具体项目/人，主动推送通知(tmux消息/SendMessage)，不要静默处理
  → 状态文件要及时清理(如设备关机后清除计时状态)，避免下次误判
  → 自己的日志文件也要做体积保护(如resource_monitor.sh超5MB自动截断)，别自己变成占盘大户
```

## 关键边界：不是"可以随便重启共享服务"的许可
VPS共享铁律明确禁止**手动**kill/重启xray/socks5-relay等共享代理服务（见项目CLAUDE.md）。`proxy_keepalive.sh`+`relay_watchdog.sh`是**已验证、袁总认可的自动化例外**，边界很窄：
- 只有**探测到确凿的故障特征**才触发——服务真的down了(`systemctl is-active`为false)、代理实测连不通(curl测试非200/302)、fd数远超正常值(>800)、CLOSE-WAIT连接数异常(>50)。不是"看着不顺眼"或"猜测可能有问题"就重启
- 只做`systemctl restart`（重启服务本身），**绝不碰配置文件**（`/usr/local/etc/xray/config.json`等仍然禁止改动），不碰iptables/ufw，不reboot
- 这个例外只覆盖已有这两个脚本管的服务列表；新增任何"自动重启共享服务"的看门狗，必须先把触发条件的确凿性想清楚、最好找袁总确认一次，不能照抄这个例外就随便扩大范围到没验证过的服务

## 教训：陌生资源默认当"别人的"，别当异常处理
2026-09-13 ENOE设备事件：云手机看门狗一开始把一个不认识的GeeLark设备当异常在管，后来袁总确认是同事的设备，不归自己这边管。**遇到监控范围内出现陌生/未登记的资源，默认先当"是团队里别人在用的"，不要自动关停/处理/报警**，先确认归属或加白名单排除，避免误伤别人的工作。这条不止适用于云手机，任何"批量扫描到不认识的东西"的场景都适用（比如排查重复Claude会话时，manager那对也是同一个教训的变体——两边都可能是"别人正在用的"，不能凭表面规律批量处理，见[[vps-resource-triage]]）。

## 已知缺口（发现但还没修，供下次处理）
`health_check.sh`监控的tmux会话清单是`cc-chinapal/cc-freshlock/cc-freshlock-web/cc-xuanpin/cc-infra/cc-manager`，**没有`cc-design`和`cc-distill`**——这两个项目是后来加的，监控清单没同步更新，意味着这两个会话意外消失不会被health_check发现。下次改这个脚本时记得补上。
