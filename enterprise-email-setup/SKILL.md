---
name: enterprise-email-setup
description: 企业邮箱系统搭建技能，覆盖从域名注册到Zoho企业邮箱开通、DNS全配置（MX/SPF/DKIM/DMARC）、自动化SMTP发送配置、冷邮件预热全流程。基于多次真实配置成功的经验，包括Namecheap+Zoho、freshlocksealer.com、enoetech.com等实际案例。适用于新公司主体B2B外贸最小成本邮箱搭建、冷邮件域名预热、企业邮箱认证全套配置。
version: 1.0.0
---

# 企业邮箱系统搭建

## 技能概述

本技能基于多次真实配置成功经验（enoetech.com、freshlocksealer.com等），提供从域名购买到企业邮箱全链路配置，包含DNS认证、SMTP自动化发送、冷邮件预热等完整流程。适用于新公司主体B2B外贸最小成本邮箱搭建场景。

---

## 执行步骤

### 第一步：域名注册

1. **推荐注册商：Namecheap**
   - 价格透明，DNS管理方便，支持API操作
   - 新域名建议选 `.com`，B2B专业感强
   
2. **域名选择原则**：
   - 公司名称缩写 + 行业关键词
   - 短、好记、好拼写
   - 避免连字符和数字
   
3. **重要提醒**：
   - 冷邮件域名 ≠ 主站域名，建议用独立域名或 variant domain（如 `yourcompany.co`）
   - 主域名被burn后影响全站，冷邮件域名被burn不影响主站

---

### 第二步：Zoho企业邮箱开通

#### 注册入口

```
https://www.zoho.com/mail/signup.html
```

#### 注册信息填写

| 字段 | 填写内容 |
|------|---------|
| 域名 | 你的域名，如 enoetech.com |
| 邮箱地址 | 管理员邮箱，如 enoe@enoetech.com |
| 密码 | 强密码（字母+数字+符号） |
| 手机号 | 接收验证码 |
| 公司名 | 目标公司名称 |
| 数据中心 | 选新加坡或美国（禁选中国） |
| 账户类型 | Business（企业邮箱） |

#### 版本选择（关键决策点）

| 项目 | Free Forever（免费版） | Mail Lite（付费版） |
|------|---------------------|-------------------|
| 价格 | $0 | $1/用户/月（$12/年/用户） |
| 用户数 | 最多5个 | 按购买座位 |
| 存储 | 5GB/人 | 5GB/人 |
| IMAP/POP/SMTP | ❌ **不支持** | ✅ 支持 |
| 自动化发送 | ❌ **无法接入脚本** | ✅ 可接入VPS脚本 |
| 适用场景 | 仅手动网页收发 | 冷邮件自动化/B2B批量 |

**核心结论**：
- **要自动发信（SMTP）→ 必须买 Mail Lite**，$1/座/年（约$12-15/年）
- 免费版无SMTP，冷邮件自动化全链路（梯度预热、自动轮换、跟进序列）直接瘫痪
- 新公司主体做B2B的最小正解：新域名 + Zoho Lite 1座位 ≈ $12-15/年

---

### 第三步：域名验证

#### 3.1 添加TXT验证记录

Zoho注册后会给出域名验证TXT记录，格式如：
```
zoho-verification=zb74432944.zmverify.zoho.com
```

在Namecheap（或你的DNS管理面板）添加：
- **Type**: TXT Record
- **Host**: @
- **Value**: `zoho-verification=zb74432944.zmverify.zoho.com`
- **TTL**: Automatic

#### 3.2 在Zoho后台验证

回Zoho Admin Console → Domains → 点验证按钮 → 绿勾即通过

**注意**：DNS传播需等待5-30分钟，验证不通过就等一会再试。

---

### 第四步：DNS配置（核心步骤）

DNS共需配置7条记录，全部配齐后邮件认证才算完整。

#### 4.1 MX记录（收信）

| Host | Type | Priority | Value | TTL |
|------|------|----------|-------|-----|
| @ | MX | 10 | mx.zoho.com | Automatic |
| @ | MX | 20 | mx2.zoho.com | Automatic |
| @ | MX | 50 | mx3.zoho.com | Automatic |

#### 4.2 SPF记录（发信授权，防垃圾邮件）

| Host | Type | Value | TTL |
|------|------|-------|-----|
| @ | TXT | `v=spf1 include:zoho.com ~all` | Automatic |

**说明**：SPF告诉收件服务器"只有Zoho的IP有权以我的域名发信"，没这条记录发的信大概率进垃圾箱。

#### 4.3 DKIM记录（数字签名，防篡改）

DKIM需从Zoho后台获取：

1. 进入 Zoho Admin Console → Domains → 选择域名 → **Email Configuration**
2. 找到 **DKIM** 区域，点击生成DKIM记录
3. Selector默认为 `zoho`，会生成一条TXT记录

添加DNS记录：
| Host | Type | Value | TTL |
|------|------|-------|-----|
| zoho._domainkey | TXT | （Zoho给出的长字符串） | Automatic |

**注意**：DKIM建议选 **2048位**（非1024位），安全性更高，Google/Yahoo等大厂2024年后已要求2048位。

#### 4.4 DMARC记录（防钓鱼，提升信誉）

| Host | Type | Value | TTL |
|------|------|-------|-----|
| _dmarc | TXT | `v=DMARC1; p=none; rua=mailto:管理员邮箱@你的域名.com` | Automatic |

**策略说明**：
- `p=none`：监控模式，只收报告不拦信（安全起步）
- `p=quarantine`：可疑邮件进垃圾箱（稳定后升级）
- `p=reject`：直接拒绝（最终目标，需先确认所有发信源都配齐SPF/DKIM）

#### 4.5 验证全部DNS记录

等5-10分钟DNS生效后，回Zoho Admin Console → Domains → Email Configuration：
- 点 DKIM 的 **Verify** → 绿勾 ✅
- 点 SPF 的 **Verify** → 绿勾 ✅
- 点 MX 的 **Verify** → 绿勾 ✅

**三个全绿勾 = 邮件认证全套完工**

---

### 第五步：邮箱账号创建

#### 常见报错及解决方案

**问题**：Zoho向导页（Dashboard）显示创建邮箱报错/失败

**原因**：向导页有bug，是Zoho已知的"假报错"

**解决方案**：
1. 不进向导页，直接进 **Admin Console**（管理后台）
2. 左侧菜单找 **Users**
3. 点 **Create mail account**（创建邮箱账号）
4. 填写用户名、密码、备用邮箱
5. 保存即可

**这是Zoho界面设计的坑，向导页和后台入口不一致导致的假报错。**

#### 建议创建的邮箱

| 邮箱 | 用途 |
|------|------|
| enoe@/info@/sales@ | 超管/主力对外邮箱 |
| support@ | 客户支持 |
| payments@ | 收款/PayPal绑定 |

---

### 第六步：SMTP自动化发送配置

#### SMTP参数

| 参数 | 值 |
|------|-----|
| SMTP服务器 | smtp.zoho.com |
| 端口 | 587（TLS）或 465（SSL） |
| 用户名 | 完整邮箱地址（如 enoe@enoetech.com） |
| 密码 | 邮箱密码（或应用专用密码） |
| 加密 | TLS/STARTTLS |

#### VPS脚本配置示例

```bash
# 环境变量配置
SMTP_HOST=smtp.zoho.com
SMTP_PORT=587
SMTP_USER=enoe@enoetech.com
SMTP_PASS=your_password
SMTP_FROM=enoe@enoetech.com
```

#### 测试发送

```bash
# 用curl测试SMTP连通性
curl --url "smtp://smtp.zoho.com:587" \
  --mail-from "$SMTP_FROM" \
  --mail-rcpt "test@example.com" \
  --user "$SMTP_USER:$SMTP_PASS" \
  --upload-file - <<EOF
From: $SMTP_FROM
To: test@example.com
Subject: SMTP Test
This is a test email.
EOF
```

---

### 第七步：收发测试与验证

1. 用QQ邮箱/Gmail给新域名邮箱发一封测试信 → 确认收件正常
2. 用新域名邮箱回复一封 → 确认发件正常
3. 查看收到的测试信头部：
   - `SPF=pass` ✅
   - `DKIM=pass` ✅
   - `DMARC=pass` ✅
4. 用VPS脚本发一封测试信 → 确认SMTP链路通畅

---

## 冷邮件预热规范

新域名邮箱开通后，**严禁直接批量发送**，必须按梯度预热（4周完整节奏表见 b2b-cold-email 技能 Phase 3）。核心指标：退信率 < 3%，打开率 > 15%，垃圾邮件投诉率 < 0.1%。

---

## 成本对比

### 方案一：Zoho Free（仅手动收发）
- 域名：$8-12/年（Namecheap .com）
- 邮箱：$0/年
- **合计：$8-12/年**
- 限制：无SMTP，无法自动化

### 方案二：Zoho Mail Lite（推荐，B2B最小正解）
- 域名：$8-12/年
- 邮箱：$12-15/年（1座位）
- **合计：$20-27/年**
- 完整SMTP，支持冷邮件自动化

### 方案三：Google Workspace
- 域名：$8-12/年
- 邮箱：$72/年（$6/月/用户）
- **合计：$80-84/年**
- 优势：Google IP池信誉最高，Postmaster数据可查
- 劣势：价格是Zoho的3-4倍

### 方案四：自建邮件服务器
- VPS：$60-120/年
- 域名：$8-12/年
- 维护成本：高（需持续监控IP信誉、黑名单等）
- **不推荐**：性价比极低，IP信誉难养

---

## 常见报错及解决方案

### 1. 域名验证失败
- **原因**：DNS未生效
- **解决**：等30分钟重试，用 `dig TXT 域名` 检查传播状态

### 2. MX记录验证失败
- **原因**：MX记录值拼错或优先级写错位置
- **解决**：确认Value填 `mx.zoho.com`（不是URL），Priority填数字

### 3. DKIM验证失败
- **原因**：Selector不匹配或TXT值被截断
- **解决**：确认Host是 `zoho._domainkey`，Value完整粘贴（很长，别截断）

### 4. 向导页创建邮箱报错
- **原因**：Zoho向导页已知bug
- **解决**：进Admin Console → Users → Create mail account

### 5. SMTP连接失败
- **原因**：端口被封锁或密码错误
- **解决**：确认端口587（TLS），检查密码，部分VPS需开放587出站端口

### 6. 发送的邮件进垃圾箱
- **原因**：SPF/DKIM/DMARC未全配齐，或域名信誉未建立
- **解决**：确认三条全绿勾，执行预热流程，检查邮件内容是否含垃圾邮件关键词

---

## 检查清单

搭建完成后，逐项确认：

- [ ] 域名TXT验证通过
- [ ] MX ×3 记录添加，Zoho验证绿勾
- [ ] SPF记录添加，Zoho验证绿勾
- [ ] DKIM 2048位记录添加，Zoho验证绿勾
- [ ] DMARC记录添加（p=none监控模式）
- [ ] 邮箱账号通过Admin Console → Users创建（非向导页）
- [ ] 收件测试通过（QQ邮箱→新域名邮箱）
- [ ] 发件测试通过（新域名邮箱→QQ邮箱）
- [ ] 邮件头部检查 SPF=pass / DKIM=pass / DMARC=pass
- [ ] SMTP脚本发送测试通过
- [ ] 预热计划启动
