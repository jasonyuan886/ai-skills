# FreshLock UTM 命名规范文档

> 用途：统一全站流量追踪参数，确保GA4数据准确可分析
> 执行人：袁小助（部署到所有链接）+ A佬（内容中统一使用）

---

## 一、UTM参数标准格式

所有外链URL必须按以下格式添加UTM参数：

```
https://www.freshlocksealer.com/?utm_source={source}&utm_medium={medium}&utm_campaign={campaign}&utm_content={content}&utm_term={term}
```

### 参数定义

| 参数 | 含义 | 必填 | 示例 |
|------|------|------|------|
| utm_source | 流量来源（平台/渠道） | ✅ | tiktok, pinterest, reddit, email, google |
| utm_medium | 流量媒介类型 | ✅ | social, email, organic, paid, referral |
| utm_campaign | 活动名称 | ✅ | seo_batch1, abandoned_cart, welcome_series |
| utm_content | 具体内容/变体 | 可选 | email1, video3, pin_001, post_title |
| utm_term | 关键词（付费搜索用） | 可选 | vacuum_sealer, food_preservation |

---

## 二、命名规则

### 规则
1. **全部小写**，禁止大写字母
2. **用下划线**连接单词，禁止空格或连字符
3. **简短明确**，每个参数不超过30个字符
4. **英文命名**，统一用英文
5. **版本号**：如需区分版本，用v1/v2/v3后缀

---

## 三、各渠道UTM规范

### TikTok
```
utm_source=tiktok
utm_medium=social
utm_campaign=tiktok_organic
utm_content=video_{序号}
```
示例：`?utm_source=tiktok&utm_medium=social&utm_campaign=tiktok_organic&utm_content=video_001`

### Pinterest
```
utm_source=pinterest
utm_medium=social
utm_campaign=pinterest_pins
utm_content=pin_{序号}
```
示例：`?utm_source=pinterest&utm_medium=social&utm_campaign=pinterest_pins&utm_content=pin_001`

### Reddit
```
utm_source=reddit
utm_medium=social
utm_campaign=reddit_posts
utm_content={subreddit}_{post_type}
```
示例：`?utm_source=reddit&utm_medium=social&utm_campaign=reddit_posts&utm_content=mealprepsunday_sunday`

### LinkedIn
```
utm_source=linkedin
utm_medium=social
utm_campaign=linkedin_company
utm_content={post_topic}
```

### Quora
```
utm_source=quora
utm_medium=social
utm_campaign=quora_answers
utm_content=answer_{序号}
```

### Email
```
utm_source=email
utm_medium=email
utm_campaign={序列名}
utm_content={邮件序号}
```
示例：`?utm_source=email&utm_medium=email&utm_campaign=abandoned_cart&utm_content=email1`

### SEO博客文章
```
utm_source=blog
utm_medium=organic
utm_campaign=seo_batch_{批次号}
utm_content={文章slug}
```
示例：`?utm_source=blog&utm_medium=organic&utm_campaign=seo_batch_1&utm_content=how_to_vacuum_seal`

### 程序化SEO页面
```
utm_source=pseo
utm_medium=organic
utm_campaign=pseo_landing
utm_content={页面slug}
```

### Influencer合作
```
utm_source=influencer
utm_medium=referral
utm_campaign=influencer_{influencer名}
utm_content={合作内容类型}
```
示例：`?utm_source=influencer&utm_medium=referral&utm_campaign=influencer_chefjohn&utm_content=youtube_review`

### Google Shopping（如开通）
```
utm_source=google
utm_medium=paid
utm_campaign=shopping_{产品名}
utm_term={关键词}
```

### 冷邮件
```
utm_source=email
utm_medium=email
utm_campaign=cold_outreach
utm_content=influencer_{序号}
```

---

## 四、Referral链接

Referral系统生成的链接格式：
```
https://www.freshlocksealer.com/?ref={user_code}&utm_source=referral&utm_medium=referral&utm_campaign=referral_program&utm_content={referrer_id}
```

---

## 五、验证清单

部署后检查：
1. [ ] 所有TikTok视频bio链接带UTM
2. [ ] 所有Pinterest pin带UTM
3. [ ] 所有Reddit/Quora帖子带UTM
4. [ ] 所有邮件序列中的链接带UTM
5. [ ] 博客文章内链CTA带UTM
6. [ ] Influencer合作专属链接带UTM
7. [ ] GA4中能正确识别各渠道流量

---

## 六、GA4自定义维度建议

在GA4中配置以下自定义维度，以便更好地分析：
- `utm_content` → 作为自定义维度，追踪具体内容表现
- `utm_campaign` → 追踪活动级别ROI

---

## 附：UTM构建器模板

快速生成UTM链接的模板（袁小助可做成工具）：

```
基础URL = https://www.freshlocksealer.com/products
UTM = ?utm_source={平台}&utm_medium={类型}&utm_campaign={活动}&utm_content={内容}

示例输出：
https://www.freshlocksealer.com/products?utm_source=tiktok&utm_medium=social&utm_campaign=tiktok_organic&utm_content=video_005
```
