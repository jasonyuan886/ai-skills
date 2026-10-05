# SEO外链建设计划

> 创建时间：2026-08-12
> 背景：品牌词"FreshLock vacuum sealer"在Google搜索完全不可见，10+竞品占用同名品牌。根因：0外链+新站+品牌名碰撞。GSC数据显示有20点击/1527曝光，Google有索引但排名极低。

## ✅ 已完成（8/12 16:30）

| 措施 | 说明 | 影响 |
|------|------|------|
| IndexNow提交 | EN 101 + JP 45 + TH 65 = 211 URLs提交到IndexNow API（Bing/Yandex） | Bing/Yandex加速爬取收录 |
| Bing直接提交 | EN站20个关键URL直接提交到Bing IndexNow端点 | HTTP 200确认接收 |
| JP站Pinterest域名验证 | 添加`p:domain_verify` meta标签到JP站layout.tsx并部署 | Pinterest可验证JP站所有权 |
| GitHub仓库外链 | 3个仓库更新description+homepage链接（freshlock-store/jp/th） | GitHub DA 96，3条外链 |
| GitHub README外链 | EN站README.md添加live site链接 | GitHub页面渲染的dofollow链接 |
| 技术SEO审计 | EN/JP站meta/canonical/hreflang/structured data全部确认正确 | 无技术障碍 |

## 🔴 需浏览器操作（按优先级排序）

### P0 — 最高影响（建议立即执行）

1. **Google Merchant Center免费商品列表**
   - 影响：产品出现在Google Shopping搜索结果（免费）
   - 需要：Google账号登录 → 创建Merchant Center → 验证网站 → 上传产品feed
   - 账号：freshlocksealer@gmail.com
   - 网站验证：已有GSC验证
   - 产品feed：需创建XML/CSV格式产品数据

2. **Bing Webmaster Tools注册**
   - 影响：直接提交URL到Bing，监控Bing索引状态
   - 需要：Google账号登录 → 添加网站 → 验证 → 提交sitemap
   - 验证：可复用Google site verification meta tag

### P1 — 高影响

3. **Facebook Business Page**
   - DA 96，29亿月活用户
   - 页面链接dofollow，提升品牌搜索可见度
   - 需要：Facebook账号登录 → 创建商业页面 → 填写信息+链接

4. **LinkedIn Company Page**
   - DA 99，公司页面在Google排名高
   - 帖子内容已准备好（linkedin_post_20260810.md）
   - 需要：LinkedIn账号登录 → 创建公司页面

5. **Trustpilot商家列表**
   - DA 93，信任信号
   - 免费商家列表
   - 需要：注册 → 添加商家信息

### P2 — 中等影响

6. **Hotfrog目录提交**
   - DA 75，dofollow链接
   - 免费提交，1-3天审核

7. **Product Hunt发布**
   - DA 90+，dofollow链接
   - 适合新品发布，社区曝光

8. **Medium文章**
   - DA 96，可在文章中插入链接
   - 写一篇食物保存指南文章，链接到产品页

9. **Quora问答**
   - DA 93，回答真空保存相关问题
   - 在回答中自然插入产品链接

### P3 — 长期策略

10. **客座博客（Guest Post）**
    - 联系食物/厨房类博客
    - 提供高质量文章换取dofollow外链
    
11. **Reddit评论**
    - 服务器IP被封，需袁总本机操作
    - 在相关subreddit中自然推荐产品

12. **Curlie目录（DMOZ继承者）**
    - DA 75+，免费但审核3-6个月
    - 志愿者编辑审核，质量高

## 📊 现有外链来源

| 来源 | 类型 | 链接到 | 状态 |
|------|------|--------|------|
| Pinterest每日Pin | nofollow | 产品页 | ✅ 每日2 Pin |
| Reddit评论×2 | nofollow | 网站首页 | ✅ 已发布 |
| GitHub仓库×3 | dofollow | 网站首页 | ✅ 已更新 |
| GitHub README | dofollow | 网站首页 | ✅ 已更新 |

## 📈 IndexNow Key信息

- Key: `eb589451cbd949fe908fd5d47d6ad5e7`
- Key文件位置: `https://www.freshlocksealer.com/{key}.txt`（三站均已部署）
- API端点: `https://api.indexnow.org/indexnow`
- 支持搜索引擎: Bing, Yandex, Naver, Seznam, Yep
- 注意: Google不支持IndexNow

## 🔄 后续维护

- 每次发布新博客文章后，通过IndexNow提交新URL
- 每周检查Bing Webmaster Tools索引状态
- 每月评估外链建设进度
- A佬8/25审计时检查外链数量变化
