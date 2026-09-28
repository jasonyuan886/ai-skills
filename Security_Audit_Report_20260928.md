# 安全审计报告（VPS + GitHub 代码库）

> 审计日期：2026-09-28
> 范围：VPS（192.236.171.76）+ 16 个 GitHub 仓库（工作区 + git 历史）
> 说明：本报告**不含任何真实密码 / 密钥 / Token**，凭证一律用引用占位。可公开。

---

## 🔴 高危

### 1. GitHub OAuth Client Secret 泄露在公开仓库历史
- 对应 OAuth App Client ID：`Ov23liOhWk7DLu1KMZwe`（密钥值见私有凭证，不在此写出）
- 位置：**公开仓库** `freshlock-store`、`freshlock-th` 的 `app/api/auth/route.ts` 历史提交
- 现状：2026-09-07 已改用环境变量，但 **git 历史仍可被任何人翻出**，删当前文件无效
- **处置：** 立即到 GitHub 后台重置该 OAuth App 的 Client Secret；如须彻底清历史需重写 git（成本高，一般 rotate 即可）

### 2. 私有备份仓库明文堆放大量活凭证
- 仓库：`vps-projects-backup`（私有）
- 在用的 GitHub PAT 明文写在 `freshlock-web/CLAUDE.md`；同处及 `infra/CLAUDE.md` 还散落：Gmail 密码、DeepSeek key、MiniMax key、New-API 口令、2captcha、BetaList 等
- 已交叉验证：这些凭证**未泄露到任何公开仓库**；但单一私有仓库被攻破即等于全套服务沦陷
- **处置：** 将凭证迁出 CLAUDE.md（抽到不入库 / 仅本地的独立文件），并轮换已多处出现的 GitHub PAT

---

## 🟠 中危（VPS）

### 3. SSH 暴力破解
- 22 端口每天遭数百至数千次尝试；原配置 root 直接密码登录、无 fail2ban
- 2026-09-28 已改为仅允许密钥登录（修复中，方向正确）；建议补装 fail2ban 兜底

### 4. 邮件端口全对公网开放
- 25 / 143 / 587 / 993 监听 0.0.0.0
- **待确认：** Postfix 是否构成 open relay（核查 `mynetworks`、`smtpd_relay_restrictions`、`smtpd_recipient_restrictions`），避免被利用发垃圾邮件

### 5. 代理服务监听全网
- tinyproxy（8080）、xray（8081/8443/10810）监听 0.0.0.0
- **待确认：** tinyproxy 访问白名单是否生效，配置不当会变成开放代理，被白嫖并导致 IP 进黑名单

---

## 🟡 中低危（代码层）

### 6. OAuth 登录缺少 state 参数
- 存在登录 CSRF / 账号绑定固定风险
- 回调页 `window.opener.postMessage(token, "*")` 以通配 origin 回传 token，应限定为自家域名

### 7. 缺少安全响应头
- 全站未配置 CSP、HSTS、X-Frame-Options、X-Content-Type-Options
- 无 X-Frame-Options 存在点击劫持风险

### 8. 部分接口无限流
- `contact` / `newsletter` / `coupon` 无任何速率限制（仅 abandoned-cart 有）
- 可被刷爆 / 刷邮件，建议统一限流

### 9. 博客正文使用 dangerouslySetInnerHTML
- 正文来源为仓库本地文件（非用户提交），风险较低
- 但 `mdToHtml` 不转义原生 HTML、链接未限制 `javascript:` 协议，建议加 sanitize

---

## 🟢 低危 / 运维

### 10. 系统补丁滞后
- 44 个安全补丁待更（共 118 包）；未启用 unattended-upgrades

### 11. 资源紧张
- 磁盘剩余约 2.6G、可用内存约 373M；日志暴涨即可拖垮服务，需监控 + 清理

---

## ✅ 已做对的项
- PayPal 下单金额由服务端按 slug 查权威价，不信任客户端价格（防篡改到位）
- 在用 PAT / DeepSeek key 确认未进入任何公开仓库
- 小型公开旅行仓库干净，无硬编码密钥

---

## 优先级建议
1. **今天：** 重置公开泄露的 GitHub OAuth Secret（#1）
2. **今天：** 私有备份仓库凭证迁出 + 轮换 PAT（#2）
3. **本周：** 完成 #3 fail2ban、核查 #4 open relay、#5 开放代理
4. **排期：** #6–#9 代码加固、#10–#11 补丁与资源
