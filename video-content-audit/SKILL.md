---
name: video-content-audit
description: 短视频发布前的内容审查清单。平台敏感词（TikTok 对战争/政治词零容忍且 OCR 连画面烧入文字一起审）、caption 外链降权、Shorts 字数限制等。发布任何视频前必过此清单，防止 0 播放/限流事故重演。
---

# 视频发布前内容审查（血泪版，2026-10-26 实战沉淀）

## 事故案例（FreshLock UK 视频，TikTok 0 播放）
UK "Be Ready" 热点视频：文案含 "ready for war in 1000 days"，且**第一帧画面烧入 "Defence Secretary: ready for war" 字样**。发布后 TikTok 0 推流（账号其他视频正常 33-259 播放）。
**教训**：TikTok 的 OCR 会读画面里的所有文字，"war/weapon/战斗/政治立场" 类词出现在**任何一帧**都触发限流，与文案无关也照样死。

## 必查清单（发布前逐项打勾）

### 1. 画面文字审查（最重要，最易漏）
- [ ] 逐帧检查烧入文字：无 war/weapon/kill/政府黑料/政治口号/竞争对手商标
- [ ] 钩子可以猛，词必须干净："stockpile food"✅ / "stockpile for war"❌
- [ ] 替代表达：战争题材 → "emergency prep / Be Ready campaign / preparedness"；用政府公告原文代替主观判断

### 2. 文案审查（四平台分检）
- [ ] **TikTok/IG/FB**：无外链（写 "link in bio"，站外链接=降权/拦截）
- [ ] **TikTok**：无政治/战争/疫情类关键词
- [ ] **YouTube Shorts**：标题 ≤100 字符（超了 Upload 按钮不激活）；受众必须选 "Not made for kids"
- [ ] caption 里 hashtags 控制在 4-6 个

### 3. 热点蹭流国家标签
- UK→#UKNews #BeReady；US→#USNews #EmergencyPreparedness；JP→防災 #備蓄
- 战争类热点只接"政府号召/民生角度"，不碰政治立场

### 4. 发布后验证（2 小时节点）
- 0-3 播放 = 限流嫌疑，立刻检查（换小号搜标题/看是否能被推荐页刷到）
- **不信任何自动化管线的状态字段**（verified 可能假阳性）——YouTube 用 RSS 对标题、其他平台看前台真实帖
- 限流处理：删帖 → 去敏感词重渲 → 重发（0 播放的帖子删了不亏）

## 各平台红线速查
| 平台 | 红线 | 后果 |
|---|---|---|
| TikTok | 战争/政治词（文案+画面帧） | 0 推流，无声限流 |
| FB/IG | caption 带外链 | 降权/拦截 |
| YouTube Shorts | 标题>100字符 | 无法发布 |
| 全部 | 抄袭/搬运画面 | 封号 |
