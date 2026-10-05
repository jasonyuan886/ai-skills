# FreshLock 独立站建站经验包（供 AI 学习/复用）

> 来源：袁小助 2026-05 ~ 2026-09 实操 FreshLock 手持真空封口机独立站（freshlocksealer.com）的全套沉淀
> 用法：按目录顺序读即可。技术栈 Next.js + Vercel + GitHub + Cloudflare CDN，支付 PayPal，多语言 EN/JP/TH。

---

## 📂 目录与阅读顺序

### 01_建站全流程（先看这个，建立全局）
- `FreshLock独立站建设全流程SOP_25页.pdf` —— **核心**，25页全流程：技术选型、域名/部署、三语言结构、支付链路、社媒引流、GA4看板、26条踩坑教训
- `FreshLock独立站从零到一全流程手册_16页.pdf` —— 16页精简手册，同上脉络的另一版
- `全量任务计划与经验手册.md` —— 项目任务全景 + 阻塞项处理方式

### 02_SEO
- `SEO审计报告_20260725.md` —— **核心方法论**：技术SEO/内容SEO/结构化数据评分体系，P0-P2问题清单与优先级打法
- `SEO外链建设计划.md` —— 零广告外链建设（GitHub高权重外链、IndexNow主动推送、Google Shopping Feed）
- `内链映射.md` —— 博客/产品页内链网络设计
- `UTM命名规范.md` —— 全站流量追踪参数标准（GA4数据准确的前提）
- `SEO周报/` —— 3期真实周报，含GSC点击/曝光/排名 + GA4漏斗数据分析模板

### 03_GEO_AI导购（AI时代的SEO，重点）
- `GEO优化与PayPal幂等经验.md` —— **核心**：AI推荐如何带来真实订单（Grok案例）、品牌实体一致性、llms.txt写法、robots放行AI爬虫；附PayPal订单幂等铁律
- `geo权威页内容.md` / `geo页面内容.md` —— 给AI看的权威页成品
- `交易型问题库.md` —— 拦截AI问答流量的问题清单

### 04_GA4_GSC数据分析
- `流量日报样例_GA4GSC.md` —— GA4/GSC数据日报真实样例（含过滤中国/香港/VPS无效流量的口径）
- `GSC索引诊断方案.md` —— Google收录问题诊断与修复流程

### 05_内容模板
- `SEO文章30篇/` —— 30篇SEO博客成品（选题大纲+正文），可直接参考结构与关键词布局

---

## 🔑 最关键的几条经验（速记）

1. **SEO 已经变成 GEO**：除了Google排名，还要让 ChatGPT/Grok/Perplexity 记对品牌名、给对官网。核心是品牌实体一致性 + llms.txt + 权威页 + robots放行AI爬虫。
2. **技术SEO硬伤一票否决**：meta模板全站复用、description超160字符、薄内容(<200词)、缺hreflang、结构化数据缺字段——这些不修，内容再多也没排名。
3. **结构化数据按页给对**：Product(含price/reviews/availability/brand)、Article给博客、BreadcrumbList、FAQPage独立放FAQ页。
4. **GA4**：埋 gtag + 电商事件(view_item/add_to_payment/begin_checkout/purchase)；purchase走服务端Measurement Protocol补数，过滤CN/HK/机房流量；UTM全站统一。
5. **新站零权威打法**：IndexNow主动推送、GitHub仓库外链、Google Shopping Feed(零广告上架)、Reddit/Quora引流。
6. **PayPal必须自己做幂等**，不能依赖PayPal窗口（18封重复订单邮件的教训）。

## ⚠️ 数据口径
- GA4 Measurement ID：G-N16R0F2B1Y（FreshLock专用，仅作埋点参考，勿在他站复用）
- 部署铁律：代码推 GitHub → Vercel 自动构建 → Cloudflare CDN 分发
