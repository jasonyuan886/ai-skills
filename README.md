# AI Skills Library 🤖

实战经验技能库 — 供 AI Agent 调用执行的可复用经验包。

## 📗 成功经验技能

| 技能 | 说明 |
|------|------|
| `dtc-site-operations` | DTC独立站全链路运营（建站/部署/SEO/GEO/PayPal/信任体系） |
| `b2b-cold-email` | 外贸B端冷邮件获客全流程（线索库/预热/批量发送/跟进） |
| `social-media-matrix` | 社媒矩阵运营（LinkedIn/Telegram/X，GeeLark云手机ADB） |
| `short-video-batch` | 短视频批量制作与多平台发布（Ken Burns/热点快速响应） |
| `ai-pcb-generation` | AI PCB自动生成（Altium Designer逆向/OLE2格式/自动布局布线） |
| `enterprise-email-setup` | 企业邮箱系统搭建（Zoho/DNS/DKIM/SMTP配置） |
| `chinapal-ai-operations` | AI聚合平台运营（SEO/博主合作/流量获取） |
| `cloud-phone-adb-publishing` | 云手机ADB视频发布全链路（ADB就绪时序/深度链接/只读核验/成本控制） |
| `content-queue-scheduling` | 内容队列排期与防重发（not_before闸门/pushed不重试/归档不删除/字段契约） |
| `publish-verification` | 发布结果核验方法论（公开RSS优先/主页截图/published_hint不可信） |
| `ops-gateway-restricted-actions` | 受限动作网关设计（白名单/realpath围栏/ZIP导入防御/上传链路对齐） |
| `shared-vps-service-deploy` | 共享VPS新服务部署（只加不改/低权限systemd/nginx追加location/部署后验证） |
| `vps-resource-triage` | VPS资源紧张排查与重复会话清理（磁盘vs内存分诊/交叉验证禁止表面规律判断） |
| `vps-watchdog-pattern` | VPS看门狗脚本设计模式（自愈/预警/共享服务重启的验证边界） |
| `customer-inquiry-response` | 客户询盘回复审批流程（冷邮件可直发 vs 客户回信必须确认再发） |
| `outreach-hypothesis-review` | 假设驱动的外联/内容效果复盘方法论（真实数据核验/排除噪音/持续迭代假设） |
| `deepseek-task-routing` | DeepSeek任务分流策略（结构化/批量任务分流省额度，API连通已验证，效果数据积累中） |

## 📕 避坑指南技能

| 技能 | 说明 |
|------|------|
| `dtc-site-pitfalls` | DTC独立站踩坑记录与预防（支付/SEO/国际化/运费） |
| `doudian-pitfalls` | 抖店运营避坑指南（恶意下单/投放异常/体验分） |
| `infra-pitfalls` | 技术基础设施避坑（VPS代理/NAT/fd泄漏/续费管理） |
| `social-media-pitfalls` | 社媒运营避坑指南（登录/代理/风控/通道铁律） |
| `email-system-pitfalls` | 邮箱系统搭建避坑（Zoho假报错/免费版限制/DNS遗漏） |
| `crossborder-platform-pitfalls` | 跨境平台避坑指南（主体限制/审核/运费/退货） |
| `compliance-risk-pitfalls` | 合规风控避坑指南（PayPal合规/隐私政策/冷邮件合规） |

## 使用方式

每个技能目录下包含 `SKILL.md`，AI Agent 可直接读取并遵循其中的执行流程。

## 入库标准（提交前必读）

**不是什么内容都值得做成技能。技能必须是实战验证过、100%有效的经验，宁缺毋滥。**

- 只试过一次、没有真实验证、还停留在假设/猜测阶段的东西，不要提炼进库
- 已经证实走不通/放弃的路线，不要写成"经验"（可以写进对应的"避坑指南"，但要明确标注结论是"此路不通"）
- 提炼前问自己：如果另一个Agent直接照抄这份SKILL.md去执行，会不会踩坑或者做错？不确定就先不提交
- 每个SKILL.md需要标准YAML frontmatter（`---`包裹，含`name`/`description`字段），方便被正确解析
- 提交前检查是否与库内已有技能重复——同一个知识点没必要在两个文件里各写一遍

## License

Private - Internal Use Only
