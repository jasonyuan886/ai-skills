# 项目交接总览 / Master Handover

> **文档性质**：跨平台 AI 协作交接资料。任何 AI Agent 可直接读取本文件，快速理解业务全貌、项目状态与操作边界。
> **最后更新**：2026-09-27
> **维护人**：袁总（深圳市七力科技有限公司）

---

## 0. 安全约定（先读这条）

- **本文件不含任何密码、Token、密钥、API Key、账号口令。** 所有凭证集中保存在私有 `SECRET` 存储与 VPS 本地配置中，仅在执行时按需读取，禁止打印、外传或写入公开仓库。
- 如需凭证：向项目主理人申请，或在被授权的执行环境内通过环境变量 / 凭据服务读取。
- 仓库 `jasonyuan886/ai-skills` 为**公开仓库**，任何提交都不得携带敏感信息。

---

## 1. 主理人与公司

- **主理人**：袁总
- **主体**：深圳市七力科技有限公司 / Shenzhen Qili Technology Co., Ltd
- **地址**：深圳市龙华区大浪创艺路安宏基工业园 C 栋 3 楼
- **主营**：手持真空封口机（PVS02）及配套真空袋；Mason Jar 真空封口机；面向俄罗斯及全球外贸 B 端 + 欧美日 DTC C 端
- **关联主体**：ENO TECHNOLOGY CO., LTD（品牌 VAKENZA，Mason Jar Vacuum Sealer 制造商）

### 核心产品参数（不可擅改）

- 手持真空封口机 PVS02：-60kPa / USB-C 充电 / 1200mAh / 约 210g / BPA-free / CE+FCC+RoHS
- 产品图**只用实拍**，禁止 AI 生成产品图；视频不真人出镜，仅手 + 产品 + 操作特写
- 违禁/高风险词零容忍：patent、exclusive、waterproof、commercial-grade 等

---

## 2. 主理人工作偏好（重要）

- 极度厌恶无意义过程汇报，**只要最终结果**；短句、直接
- 运营全自动化：养号 / 发帖 / 互动 / 文案全部由 AI 完成，禁止要求主理人参与执行
- 对外品牌统一「七力 / Qili」，**禁止出现 "Coze"**
- 建议必须有数据支撑，禁止猜测配置值
- 没出单前禁止付费广告，全走免费流量
- 严控积分 / 成本：批量任务优先脚本与 VPS，临时脚本用完即删
- 事实性问题先检索验证，禁止纯推理下结论

---

## 3. 协作架构与边界

```
袁总（决策）
   │
   ├── 袁小助（参谋 / 建机制 / 转达决策，不接执行）
   │
   └── VPS Claude Code 编队（cc-manager 总管 + 各项目组）
            └── 所有项目执行、代码、服务、cron 全部编队内闭环
```

**铁律：**
1. 袁小助只当参谋、出谋划策、建机制，机制建完即退出，不接活
2. VPS 上一切代码、服务、cron 由 Claude Code 编队自管；外部 Agent 默认不直接操作 VPS（除非主理人临时授权特定任务）
3. 对编队的唯一动作：把主理人定的决策写成指令，转达总管室 cc-manager
4. 零手动：任何事不得让主理人手动操作（生成 token、点确认、复制粘贴等全禁）

---

## 4. 项目清单与状态

### 项目 1 — FreshLock DTC 独立站

- **定位**：手持真空封口机 DTC，面向欧美日 C 端
- **站点**：
  - EN 主站 https://www.freshlocksealer.com （USD）
  - JP 站 https://jp.freshlocksealer.com （JPY）
  - B2B 站 https://go.freshlocksealer.com
- **技术栈**：Next.js + Vercel 构建 + Cloudflare CDN（NS kaiser/veronica.ns.cloudflare.com）
- **支付**：PayPal Business Live（POST /api/paypal 创建订单 → PUT /api/paypal 收款验证）
- **代码仓库**：`jasonyuan886/freshlock-store`（EN）、`freshlock-jp`、`freshlock-th`；订单私有库 `freshlock-orders`
- **部署铁律**：代码推 GitHub → Vercel 自动构建 → Cloudflare 分发，无需手动部署
- **已完成**：GSC 结构化数据、hreflang、canonical、301、sitemap、42 篇博客 128 内链、GEO 页面、Google Merchant Center、Trustpilot 企业页、ScamAdviser 认领
- **真实订单**：
  - 首单 Michael Pryor（纽约）Starter Kit $94.99
  - 第二单 Jennifer Dalton（英国伦敦）FreshLock Pro $80.98（2026-09-17，PayPal 已收款）
- **待办/风险**：
  - 订单持久化链路疑似失效（第二单未写入 GitHub 订单库，邮件归档兜底正常）
  - 国际运费倒贴（英国仅收 $5.99，实际约 $15-25），需分区运费 + 税费透明化
  - 品牌词 "FreshLock" 被 10+ 竞品占用、0 外链，品牌词搜索不可见
  - Stripe HK 永久搁置（无香港身份证/地址/银行账户）
  - PayPal 合规审核（材料已备齐）

### 项目 2 — 外贸 B2B 冷邮件获客

- **定位**：手持真空机全球 B 端批发（品牌 Edelweiss 俄罗斯 / QILI 通用 / SealQ B2B）
- **成果**：
  - 有效线索库 100 条，分层 Tier1/2/3
  - 全球冷邮件 48 封覆盖 19 国（含俄罗斯 10 封）
  - **2 个真实询盘：AENO、KODAECS**
- **执行铁律**：
  - 过滤所有中国域名邮箱
  - 新域名梯度预热（首日 ≤10 封，周累计 ≤50 封）
  - 发前背景筛查（商标/公司归属，中港背景排除，同行不白发）
  - 禁无差别群发；发送后校验存证
  - B 端仅保留批发商/分销商/跨境卖家/小B，纯 C 端转独立站
- **待办**：送达率与进箱监控、WhatsApp 渠道、KODAECS 样品发货跟进

### 项目 3 — AI PCB 自动生成

- **定位**：AI 自动生成 Altium Designer `.PcbDoc`，面向电源/单片机工程师，100W 以内双面板
- **护城河**：可输出 AD 原生 PcbDoc 格式
- **技术突破**：
  - AD 13.3 PCB 文件格式全量逆向（OLE2 Compound Document）
  - 纯 Python OLE2 写入器自研（规避 olefile mini-stream 写入问题）
  - 关键发现：AD 优先从 `WideStrings6` 流读取元件名；13 个强制流全部合规
  - 146 个标准封装入库 + 53 组定制封装提取
  - DelphiScript 备用方案（调 AD 原生 API）
- **现状**：BP2836 LED 驱动板持续迭代，可输出 669×2165mil 样板
- **待办**：DelphiScript 实机验证、模板扩展（反激隔离/LM2596/TP4056/7805）、后端 API + 小程序 MVP

### 项目 4 — ChinaPal AI（aihubai.cn）

- **定位**：来华外国人 AI 助手（翻译 / 导航 / 文化解读）+ AI 聚合能力展示
- **状态**：
  - 网站上线运营，品牌/翻译/信任要素全修复
  - AIHub 极简聊天前端，13 个助手（6 P0 + 7 P1）角色感知
  - Web Speech API 语音翻译、翻译付费墙
  - 内容：SEO 页 + 工具页、Medium/Dev.to 技术博客矩阵
  - 6 个旅游博主付费合作外联、4 份报价送达
- **流量策略**：聚焦 4 条自有可控路径，关停高风险获客渠道
- **待办**：外链建设、短视频持续发布、博主合作转化

### 项目 5 — 海外短视频矩阵

- **定位**：多平台（YouTube / TikTok / Pinterest / Instagram）短视频批量生产
- **成果**：
  - YouTube 8 条 Shorts 公开发布
  - Pinterest 每日 2 Pin（EN/JP，v3 风格，产品无变形校验）
  - 热点快速响应：赵家驹夺冠热点 10 条 Ken Burns 竖版视频，成本约 ¥9
- **SOP**：选题 → 素材（实拍/即梦）→ Ken Burns 动效 → 剪映二次编辑（字幕/BGM/转场/AI 声明）→ 多平台适配发布
- **待办**：发布流程剧集/系列化分类、持续排期

### 项目 6 — ENOETECH / VAKENZA B2B 官网

- **定位**：Mason Jar Vacuum Sealer 制造商 B2B 官网
- **站点**：https://www.enoetech.com
- **技术栈**：Next.js 16 + Cloudflare Pages；域名 Namecheap，Cloudflare NS 已激活
- **状态**：线上可访问，首页/产品页/Insights 完整；Next.js metadata、sitemap、robots.txt、llms.txt 已配
- **待办**：确认线上代码仓库、补全 Organization/Product 结构化数据、图片 alt 与独立 Meta、提交 GSC

---

## 5. 基础设施（仅描述，凭证在私有存储）

- **US VPS**：RackNerd，Debian 12，部署多类业务；资源偏紧（磁盘/内存余量小），新增服务前先评估
- **代理节点**：Reality / VMess / SOCKS5 等，各项目代理互相隔离、禁止挪用
- **GeeLark 云手机（4 台）**：
  - US01 旅游（ChinaPal）
  - US02 独立站（FreshLock）
  - US Reality-B2B（外贸 B2B，三平台主力机）
  - ENOE（B2B）
- **住宅 IP**：IPRoyal Static Residential ISP 类型；ENOE 专属独立 IP 禁止挪用
- **企业邮箱**：Zoho（MX/SPF/DKIM 2048/DMARC 全配置）；免费版无 SMTP，自动化需 Lite 版

### 社媒操作通道铁律

- 社媒/云手机操作只走 **VPS + GeeLark ADB**
- 禁用：扣子平台云手机、扣子云电脑 / agent-browser、扣子沙箱浏览器
- ADB 关键：`setStatus` body 必须 `{"ids":[...],"open":true}`
- 用毕立即关机止损（US Reality 按分钟计费）

### 社媒回复自主权

- TG 群 / 社媒群内回复直接发，无需确认
- 只有客户回复了主理人的帖子/消息后，才需经主理人确认再回复
- 对外私信/发帖先拟稿，主理人确认

---

## 6. AI 技能库（本仓库）

**仓库**：https://github.com/jasonyuan886/ai-skills

- 每个技能一个目录，内含 `SKILL.md`，AI Agent 可直接读取并遵循执行
- 三类技能：运营/获客类、技术/发布类、避坑指南类
- 新增/更新技能：在对应目录编辑 `SKILL.md`，提交到 main
- **提交前自检：不得包含任何凭证或敏感信息**

### 提交规范

- 一次提交聚焦一个主题，commit message 简明（中英文均可）
- 避坑类技能需包含：问题现象 / 根因 / 解决方案 / 预防措施
- 经验类技能需包含：触发场景 / 执行步骤 / 输入输出 / 边界

---

## 7. 当前优先级建议（供参考，最终由主理人定）

1. 修复订单持久化 + 回填第二单，分区运费与税费透明化（防继续倒贴）
2. 跟进 AENO / KODAECS 询盘转化，推进样品发货
3. 服务续费与余额（GeeLark / AdsPower 等）防止停服影响编队
4. PayPal 合规材料保持可用
5. 外链建设破品牌词 SEO 困局
6. 短视频与社媒按 SOP 持续产出，积累自然流量

---

*本文件为活文档，项目状态变化时由被授权的 Agent 或编队更新并提交。*
