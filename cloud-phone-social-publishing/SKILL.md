# 云手机社媒发帖流（GeeLark OpenAPI + ADB + App 会话）

> 适用：服务器 IP 被平台封锁（Reddit 对机房 IP 全域名 403）时，通过手机 App 已登录会话完成发帖/互动。案例：FreshLock Reddit 首帖（2026-10-20）。

## 为什么走这条路
1. 不需要平台账号密码（走 App 会话，与"不改密"令兼容）
2. 住宅侧出口，反爬最安全
3. 完全自助：Bearer 注入一次，长期自己开

## 流程
1. 凭证：`GEELARK_BEARER` + 设备 ID 由 root 注入 vpsapi 可读的 600 文件（密钥不落明文进仓库/聊天）
2. OpenAPI：`POST /open/v1/env/start`（开机）→ `POST /open/v1/adb/getData`（拿 ADB 地址+端口）
3. ADB：`adb connect <addr>` → 设备内运行 `glogin <adb_pwd>`（adb_pwd 同 getData 返回）
4. 操作 App：am start 拉起目标 App → input tap/swipe 走 UI（参考 freshlock-web/automation/*_adb_flow.py）
5. 发帖：App 内完成，截图取证，关环境可选保留

## 故障速查
- 服务器 IP 被 Reddit 封：全域名 403（连首页 HTML 都拦）= IP/ASN 信誉封锁，换请求形态无解，只能换出口（云手机/住宅代理）
- Pinterest 注册 API 400：要 reCAPTCHA → 需浏览器级自动化（puppeteer），纯脚本过不了
- X 网页登录反自动化：绕不过就不绕，走官方 App 会话

## 参考实现
`/root/projects/freshlock-web/automation/`（orchestrator.sh、social_adb_common.py、reddit_pick.py）
技能源文档：ai-skills/cloud-phone-adb-publishing/SKILL.md
