# VPS → Kimi 接手方案（Claude Code 编队交接）

> **背景**：原 Claude Code 编队已在 VPS 上运行约 1 个月（2026-08 下旬 ~ 2026-09-27），因 Claude 账号封禁停用。本文档整理 VPS 上所有资产、架构、自动化机制、各项目状态，供 Kimi（国内 AI）接手操作。
>
> **最后盘点时间**：2026-09-27
>
> **⚠️ 安全提醒**：本文档不含任何真实密码/密钥/Token。凭据统一引用 `/root/credentials_env.sh`，执行前需先 `source /root/credentials_env.sh` 加载环境变量。

---

## 一、VPS 架构全景

### 1.1 基本信息

| 项目 | 值 |
|---|---|
| IP | 192.236.171.76（RackNerd US） |
| 系统 | Ubuntu, root 用户 |
| SSH 凭据 | `[见 /root/credentials_env.sh]` |
| 磁盘 | 19G 总容量，已用 17G（**96% 告警**） |
| 内存 | 3.8G 总，已用 3.1G，可用 192M |
| Swap | 3.0G，已用 1.3G |

### 1.2 核心目录结构

```
/root/
├── credentials_env.sh          # 所有凭据集中存放（source 后导出环境变量）
├── .claude/
│   ├── CLAUDE.md               # 袁总全局规则/铁律/品牌/偏好
│   ├── CLOUD_PHONE.md          # GeeLark 云手机运营手册（社媒自动化必读）
│   ├── settings.json           # Claude Code 权限配置（参考用）
│   ├── uploads/                # 上传的规格文档（起名spec/PayPal spec/增长策略doc等）
│   └── projects/
│       ├── -root/memory/       # 全局记忆（FreshLock/B2B/抖店/社媒/ChinaPal状态）
│       └── -root-projects-chinapal/memory/  # ChinaPal 详细记忆（10+ 个 md）
├── projects/                   # 编队核心工作目录（git 仓库，每小时备份）
│   ├── manager/                # 总管事（协调所有组）
│   ├── chinapal/               # ChinaPal AI 来华旅游
│   ├── freshlock/              # FreshLock B2B 外贸（冷邮件获客）
│   ├── freshlock-web/          # FreshLock 独立站（C端电商/SEO/社媒）
│   ├── xuanpin/                # 跨境选品（爆品雷达）
│   ├── infra/                  # 基础设施（代理/IP/云手机）
│   ├── design/                 # 产品设计（3D建模/PCB，刚起步）
│   ├── distill/                # AI 蒸馏（实验性质）
│   ├── chinapal_world/         # chinapal.world 静态站点（GitHub Pages）
│   └── apple-charger/          # 充电器项目（几乎空白）
├── automation/                 # 自动化脚本（socks5_relay.py 等）
├── scripts/                    # 通用运维脚本
├── ai-skills/                  # AI 实战技能库（已推 GitHub 公开仓库）
└── make_video.py               # ChinaPal 短视频生产脚本（零成本方案）
```

### 1.3 运行的系统服务

| 服务 | 用途 |
|---|---|
| nginx | Web 服务器（80/443） |
| xray | 代理 |
| socks5-relay | SOCKS5 代理中继（住宅IP共享） |
| res-socks/res-http | 住宅 IP 代理转发（9180/9181） |
| socat-2082 | TCP 端口转发 |
| tinyproxy | HTTP 代理（8080，已加认证） |
| postfix + dovecot | 邮件系统（SMTP + IMAP） |
| tracker | AIHub 访问追踪 |
| bazi-api | ChinaPal 八字起名 API |
| agent-chat | Agent 聊天中转 |
| freshlock-gateway | FreshLock 安全任务网关 |
| proxy-monitor | 代理连接监控 |
| warp-svc | Cloudflare Zero Trust |
| tor | Tor 网络 |

---

## 二、编队架构（原 Claude Code 10 个 tmux 会话）

### 2.1 项目组一览

| tmux 会话 | 项目目录 | 职责 | 当前状态 |
|---|---|---|---|
| cc-manager | /root/projects/manager | 总管事，协调所有组 | ⚠️ 会话冻结 |
| cc-chinapal | /root/projects/chinapal | ChinaPal 社媒获客 | ⚠️ 有脚本仍在运行 |
| cc-freshlock | /root/projects/freshlock | B2B 冷邮件获客 | ⚠️ 有脚本仍在运行 |
| cc-freshlock-web | /root/projects/freshlock-web | FreshLock 独立站 | ⚠️ 有脚本仍在运行 |
| cc-xuanpin | /root/projects/xuanpin | 跨境选品 | ⚠️ cron 日报运行 |
| cc-infra | /root/projects/infra | 基础设施管理 | 纯文档组 |
| cc-design | /root/projects/design | 产品设计 | 刚起步 |
| cc-distill | /root/projects/distill | AI 蒸馏实验 | 实验阶段 |
| chinapal_world | /root/projects/chinapal_world | 静态站 | 无需 AI |
| manager | - | 辅助管理 | 无任务 |

### 2.2 每个项目组核心文件

每个 `/root/projects/<组名>/` 下通常包含：
- **CLAUDE.md** — 项目规则、账号信息、当前任务、踩坑记录（**最重要，必读**）
- **history/** — 详细档案
- **EVOLUTION_LOG.md** — 经验/教训提炼
- **PROJECT_STATE.md** — 部分组有，当前进度
- **decisions/** — 决策记录

---

## 三、当前任务状态

### 3.1 进行中

#### 🔴 FreshLock Web — 两个 P0 施工指令
1. **首页 SEO 修复**（HOMEPAGE-20260918）：沉浸式改版后 SSR 正文空壳，Googlebot 看不到内容。需改回 Server Component，保留视觉但服务端直出文案。详细需求见 `/root/projects/manager/inbound/HOMEPAGE-20260918.md`
2. **运费架构重做**（SHIPPING-20260918）：一刀切 $5.99 运费导致英国单亏损。新方案按分区定价（US $6.99/门槛$99、GB $13.99/$149、EU $15.99/$149 等）。详细需求见 `/root/projects/manager/inbound/SHIPPING-20260918.md`

#### FreshLock B2B
- 冷邮件三连修复已完成（SMTP 直连/跟进定时器/AI 手写文案）
- 等 cron 按新架构跑下一批验证

#### ChinaPal
- FB/IG 自动发布修复中，需连续验证稳定性
- TikTok 主账号放弃，备用账号限流

#### 需要袁总拍板
- FreshLock 落地页 9 条评估，等拍板
- 聊天中转界面：等 GPT API key + 扣子 token

### 3.2 已放弃
- B2B 邮箱扩 30 个 → 失败
- TikTok 数据源 → 需真机，放弃
- LinkedIn 搜索 → 新号受限，放弃
- 多域名扩展 → 不买

---

## 四、自动化脚本清单（Cron）

### 4.1 ✅ 保留运行（纯 Python/Bash，不依赖 Claude）

| 频率 | 脚本 | 用途 |
|---|---|---|
| 每2分钟 | relay_watchdog.sh | SOCKS5 看门狗 |
| 每5分钟 | proxy_keepalive.sh | 代理保活 |
| 每5分钟 | resource_monitor.sh | 资源监控 |
| 每10分钟 | logrotate syslog-cap | 日志轮转 |
| 每10分钟 | cloudphone_watchdog.sh | 云手机看门狗 |
| 每1小时 | backup.sh | Git 备份到 GitHub |
| 每天 07:30 UTC | story_pipeline.py（第一轮） | ChinaPal 短视频发布 |
| 每天 08:07 UTC | health_check.sh | 健康检查 |
| 每天 09:00 UTC | domain_warmup_chinapal.sh | 域名预热 |
| 每天 09:30 UTC | era_membership_daily.py | Era 会员日报 |
| 每天 10:00 UTC | b2b_cold_sender.py | B2B 冷邮件发送 |
| 每天 20:10 UTC | story_pipeline.py（第二轮） | 短视频第二发 |
| 每天 21:00 UTC | radar.py | 爆品雷达日报 |
| 每天 07:00 UTC | evolution_check.sh | 进化追踪汇总 |
| 每6小时 | fetch_trends.py | 趋势数据抓取 |

### 4.2 ⚠️ 需立即停用（依赖 Claude，已不可用）

| 频率 | 脚本 | 原因 |
|---|---|---|
| 每30分钟 | queue_check.sh | 向 tmux 发 claude 命令 |
| 每天 05:05 UTC | trend_scan.sh | 直接调用 claude -p |
| 每天 06:30 UTC | skill_check.sh | 向 tmux 发 claude 命令 |
| 每天 07:30 UTC | daily_task.sh（ChinaPal） | 调用 claude -p |
| 每周一三五 11:37 | weekly_lead_gen.sh | 含 claude -p 研究阶段 |

**停用方法**：
```bash
crontab -l > /tmp/crontab_backup.txt
# 编辑 crontab，在以上 5 条前加 # 注释
crontab -e
```

### 4.3 需确认
- `orchestrator_round2.sh`（20:37 UTC）— 检查是否内含 claude 调用
- `direct_send.py`（09:10 UTC）— 需确认功能

---

## 五、关键资产价值评估

### 5.1 ⭐⭐⭐ 核心资产（必须保留）
- `/root/projects/*/CLAUDE.md` — 全部项目规则/账号/踩坑记录
- `/root/projects/*/history/` — 详细执行档案
- `/root/projects/freshlock/automation/` — B2B 获客脚本流水线
- `/root/projects/manager/TASK_QUEUE.md` — 全公司任务看板
- `/root/projects/manager/inbound/` — 袁总拍板的施工指令
- GitHub 备份仓库 `jasonyuan886/vps-projects-backup`

### 5.2 ⭐⭐ 重要资产（保留）
- `EVOLUTION_LOG.md` — 各项目经验教训
- ChinaPal 短视频产线（make_video.py + story_pipeline.py）
- 爆品雷达（xuanpin/baopin_radar/）
- 社媒 ADB 发布脚本
- `/root/.claude/` 下的记忆文件和规格文档
- 系统服务（nginx/xray/postfix 等）

### 5.3 ⭐ 可清理
- Claude Code 二进制（/usr/bin/claude）
- chrome_profile_* 缓存
- __pycache__/ 目录
- .last_skillcheck_* / .last_alert_* 状态文件
- TASK_QUEUE.md.bak-* 旧备份
- distill/freshlock-web 的 venv（可重建）
- 0 字节空日志文件

---

## 六、接手操作步骤

### Step 1：读取核心文档（30 分钟了解全局）
```bash
cat /root/.claude/CLAUDE.md                    # 袁总规则铁律
cat /root/projects/manager/HANDOFF.md          # 编队自写交接入口
cat /root/projects/manager/TASK_QUEUE.md       # 任务看板
cat /root/.claude/CLOUD_PHONE.md               # 云手机手册
```

### Step 2：停掉 Claude 专属 cron
```bash
crontab -l > /tmp/crontab_backup.txt
crontab -e  # 注释掉 queue_check/trend_scan/skill_check/daily_task/weekly_lead_gen
```

### Step 3：释放磁盘空间（当前 96%）
```bash
find /root/projects/ -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null
rm -rf /root/projects/*/chrome_profile_* 2>/dev/null
rm -f /root/projects/manager/TASK_QUEUE.md.bak-*
rm -f /root/projects/manager/.last_skillcheck_*
rm -f /root/projects/manager/.last_alert_*
rm -f /root/projects/manager/.last_nudge
rm -f /root/projects/manager/.last_resume_*
# 如果还紧张：
du -sh /root/projects/distill/venv/ /root/projects/freshlock-web/venv/
```

### Step 4：检查自动化运行状态
```bash
tail -5 /var/log/backup.log
tail -5 /root/projects/chinapal/story_pipeline.log
tail -5 /root/projects/xuanpin/baopin_radar/radar.log
tail -5 /root/b2b_email_log.txt
```

### Step 5：按优先级处理待办
1. FreshLock 首页 SEO 修复（P0）
2. FreshLock 运费架构重做（亏钱中）
3. 裸域名 DNS 修复
4. B2B 冷邮件验证
5. ChinaPal 社媒稳定性

---

## 七、外部服务与账号索引

> 密码统一 `[见 /root/credentials_env.sh]`

### 域名
| 域名 | 用途 | 托管 |
|---|---|---|
| freshlocksealer.com | FreshLock 独立站 | Vercel + Cloudflare |
| chinapal.world | ChinaPal 官网 | GitHub Pages |
| aihubai.cn | ChinaPal 应用 | VPS |
| enoetech.com | ENOE B2B | VPS |

### 社媒
| 平台 | 账号 | 项目 | 云手机 |
|---|---|---|---|
| X | @JasonYuan198651 | FreshLock B2B | US-Reality |
| Telegram | FreshLock 主体号 | FreshLock B2B | US-Reality |
| Instagram | @chinapalai2026 | ChinaPal | US01 |
| YouTube | FreshLock | FreshLock | US02 |
| Dev.to | chinapal_ai | ChinaPal | - |
| GitHub | jasonyuan886 | 全局 | - |

### 支付
| 服务 | 备注 |
|---|---|
| PayPal (freshlocksealer@gmail.com) | `[见 credentials_env.sh]` |
| 万里汇 USD 提现 | 已开通 |

---

## 八、铁律（必读）

1. **品牌**：对外禁"Coze"，统一七力/Qili/ChinaPal AI
2. **所有动作以出订单为核心**
3. **禁提广告**：没出单前全走免费流量
4. **运营全自动**：禁要求袁总参与执行
5. **事实先行**：事实性问题必须搜索验证
6. **沟通风格**：中文，直接简短，只报结果
7. **费用**：50 元内自主，大消耗需确认
8. **中国邮箱过滤**：B2B 必须过滤中国域名
9. **客户邮件先审后发**
10. **GeeLark 按分钟计费**：用完立刻关机

---

*本文档由袁小助基于 VPS 盘点数据生成，供 Kimi 接手参考。*
