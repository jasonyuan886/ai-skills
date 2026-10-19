# GEO 技术就绪套件（新 DTC 站 AI 可见性速通）

> 适用：新域名独立站要在 2 周内达到"AI 爬虫畅行、内容可抽取"状态。案例：freshlocksealer.com（2026-10，外部审计技术就绪度"高"）。

## 清单（按优先级）
1. **robots.txt**：显式 Allow 五大 AI 爬虫（GPTBot/ClaudeBot/GrokBot/GeminiBot/Bytespider/PerplexityBot/OAI-SearchBot）+ `User-agent: * Allow: /`
2. **WAF 核验**：robots 放行≠真放行！用各 UA 实测目标页状态码（403 多来自 Cloudflare WAF 自定义规则，与 robots 无关）
3. **llms.txt**：品牌实体、产品清单（名称/价格/规格）、消歧块（NOT affiliated with 同名品牌）、购买政策、博文入口。产品/价格/政策变更后 48h 内更新
4. **Product JSON-LD**：schema.org/Product 含 offers/aggregateRating/review/brand——AI 抽取实体的主通道
5. **/geo 页**：结构化"best X for Y"段落，对齐 AI 查询意图
6. **sitemap 动态化**：从商品数据源生成，杜绝新品漏收录（硬编码列表必腐化）
7. **SSR 检查**：禁用 JS 抓取首页，数字/文案必须是服务端终值（计数器动画初值=$0 是典型事故）

## 红线
- 无验证数字一律不写（评价数/倍数宣称/时效）——AI 引擎对数据不一致的惩罚比谷歌快
- llms.txt 与产品页事实必须一致（配送时效、运费阈值以结账代码为准）

## 验证脚本要点
UA 矩阵探针（7 个爬虫 UA + 浏览器对照）+ `curl 页面 | grep 关键事实` 一致性检查。每周跑一次，防 WAF 规则回归。
