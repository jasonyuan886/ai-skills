# 经验：AI导购(GEO)优化与PayPal订单幂等（FreshLock 2026-08-29首单实战）

## 一、AI导购已成为真实出单渠道（Grok案例）
- 真人客户Michael经Grok推荐下单$94.99：Grok因"strongest suction"卖点推荐本品 → 卖点被AI正确引用即转化
- 但AI把品牌名记成"snap seal fresh lock mini"（实为FreshLock Pro），客户搜不到官网，漏斗折损
- 教训：AI时代SEO=GEO，核心是**品牌实体一致性**，让AI记对名字、给对官网

## 二、GEO优化清单（已在freshlock-store实施，commit 355abb6/dd555d7/25f4050f）
1. 给AI看的权威页**绝不能noindex**：/geo页原设robots noindex+不进sitemap=自相矛盾，已解除并加入sitemap
2. llms.txt是AI的"网站说明书"：参数/价格/保修必须与主站一致（原llms.txt是澳区旧数据AUD/2000mAh/380g，与美区USD/1200mAh/210g矛盾），已重写
3. 品牌防混淆：llms.txt和/geo页首段显式声明"品牌名FreshLock，与FoodSaver Mini等无关联，无snap seal产品"——AI读到澄清会自我纠偏
4. 卖点锚点：-60kPa strong suction在首页/geo/llms.txt/博文多处一致重复，AI引用稳定
5. robots.txt显式Allow GPTBot/ClaudeBot/Google-Extended/PerplexityBot/Bytespider

## 三、PayPal订单幂等铁律（18封重复邮件教训）
- 现象：客户付1笔款，网站连发18封Order Confirmed；根因=success页useEffect依赖clearCart(内联箭头每次渲染新引用)+cartItems(localStorage水合)连锁重跑→重复PUT capture；服务端无幂等，PayPal幂等窗口内重复capture返COMPLETED→每次发信；第19次才422 MAX_NUMBER_OF_PAYMENT_ATTEMPTS_EXCEEDED
- 双层修复：①服务端PUT开头GET /v2/checkout/orders/{id}，status=COMPLETED直接返回alreadyCaptured:true，不capture不发信（根治）；预检查失败放行正常capture不阻断真实支付 ②前端useRef capturedRef守卫防重入
- 实证：PayPal对已COMPLETED订单重复capture在幂等窗口内可重入返COMPLETED，**商户侧必须自己做幂等，不能依赖PayPal**

## 四、新订单实时通知（别等定时查单）
- capture成功后服务端fire-and-forget发商家通知邮件：to=[support@, 老板Gmail]，主题"NEW ORDER $金额 — 客户名 (订单号)"
- 内容含：订单号/PayPal capture交易号/金额/客户名/邮箱/电话/完整收货地址/商品清单/打包提醒
- 电话字段：PayPal v2 orders无电话位，用purchase_units[0].custom_id="phone:xxx"透传（结账表单phone→POST /api/paypal→custom_id→capture响应读出）
- 地址从captureData.purchase_units[0].shipping.address取
- 实测：Zoho SMTP(support@)发Gmail直接进收件箱不进垃圾箱，手机Gmail App秒推送
