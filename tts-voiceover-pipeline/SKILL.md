# 视频配音流水线 TTS + 混流（已实战 2026-10-22）

> 给无口播视频加英语解说。VPS 自闭环，不需要任何外部账号。

## 流水线
1. **写稿**：每条视频的 json title+description → 30-40 词解说（Shorts 15-20s 口播）
2. **TTS**：`gTTS(文本, lang="en", tld="com").save(v.mp3)` ✅ 可用
   - ⚠️ edge-tts（微软）在这类机房 IP 被 403 —— 别试，直接 gTTS
3. **混流（命门）**：
   ```
   ffmpeg -y -i 视频.mp4 -i 解说.mp3 -map 0:v -map 1:a -c:v copy -c:a aac -shortest 输出.mp4
   ```
   - **必须显式 -map 0:v -map 1:a**！ffmpeg 默认挑"最优音轨"会选中视频自带的静音轨（-55dB 假音频）
4. **音量门禁**：`ffmpeg -i 输出 -af volumedetect -f null -` → mean > -25dB 才算合格（wet_food 事故：有音轨但 -56dB 全静音）
5. **上传**：Contents API PUT 需先 GET 拿 sha（同名替换）；>2MB 用 VPS python urllib 直发（shell argv 会超限）
6. **发布**：youtube_rpa_publish.py / tk_daily.py

## 批量建议
26 条队列逐个：写稿→gTTS→混流→音量门禁→上传→发布。每日一条配音版混入日更轮换。
