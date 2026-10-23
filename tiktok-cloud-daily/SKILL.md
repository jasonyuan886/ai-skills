# TikTok 云手机全自动日更（已实战验证 2026-10-23）

> 零人工干预：cron 触发 → GeeLark 开机 → 推视频 → 发布 → 截图存证。本技能是 9 个坑换的，照做不踩雷。

## 运行方式
```
source /opt/vps-api/.geelark_env
setsid nohup env GEELARK_API=.. GEELARK_BEARER=.. GEELARK_US02_ID=.. python3 tk_daily.py > log 2>&1 < /dev/null &
```
tk_daily.py 要素：/tmp/tk_used.txt 轮换记账（防重发）；从 content_queue/*.json 读 captions.tiktok。

## 必须固化的坑（每个都是血泪）
1. **会话生命周期**：GeeLark ADB 会话用一次就过期。每个脚本必须自带：getData 轮询（ phone/start 后等 45-60s 再取）→ adb kill-server/start-server → connect → glogin → u2 连接前 `adb shell echo ok` 健康环（15 次×4s）
2. **5 话题标签门禁**：tiktok_adb_flow 的 caption 校验要求恰好 5 个 #标签；队列 json 常只有 2 个 → 发布前用 json.hashtags 自动补齐到 5
3. **root 属主文件改不动**：content_queue/*.json 是 root 的，vpsapi 写不了 → 复制到 /tmp/kimi_q/ 改副本，流程指向副本
4. **话题面板盖 Post 键**：打完标签弹出建议面板 → **用 set_input_ime(False) 收，绝不用返回键**（返回键丢草稿）
5. **Post 按钮**：优先 d(text="Post")， miss 则坐标 input tap 640 1340
6. **选片坐标**：相册 Videos tab 首片 input tap 180 400；Next 键 507 1353（×2）
7. **媒体不索引**：推送到 /sdcard/Download/ + am broadcast MEDIA_SCANNER_SCAN_FILE
8. **#!/bin/sh 的坑**：脚本里别用 "$(cat file)" 给 input text 传参（本地 shell 先展开）——文本用 u2 fastinput send_keys 或 argv 数组
9. **证据标准**：d.screenshot 存证 + 日志打 TK_DAILY_DONE；发布页文案要 dump 回读校验（"cap_in" 后门禁）

## 参考实现
VPS /tmp/tk_daily.py（轮换版）；/tmp/tk_mega.py（单发版）；automation/tiktok_adb_flow.py（Codex 战地流程，含更多门禁）
