---
name: cloud-phone-adb-publishing
description: |
  用 GeeLark 云手机 + ADB 把视频发布到 Instagram / TikTok / Facebook / YouTube 四个平台的全链路成功做法。
  适用场景：
  - 需要让 Agent 无人值守地把短视频发到社媒 App（没有官方 API 或不想用 API）
  - 需要在云手机上"看一眼"App 当前状态（截图核验、确认登录态、确认弹窗）
  - ADB 连上了但命令报 device offline / 截图是白屏 / 发布脚本跑完什么都没发出去
  核心是三条经过实测的时序规则：ADB 必须轮询到 device 再认证、等界面必须轮询不能固定 sleep、
  云手机按分钟计费用完立刻关机。来自 2026-09-21/22 FreshLock DTC 四平台发布链路的实战排查。
---

# 云手机 ADB 视频发布全链路

**动手写 ADB 点击流之前，先确认该平台有没有官方 RPA 模板——有就别用 ADB。**

GeeLark Base 套餐的 OpenAPI 里，YouTube 有官方发布端点，实测一次成功：
```bash
POST https://openapi.geelark.com/open/v1/rpa/task/youtubePubShort
必填: id(设备ID) / scheduleAt(unix秒) / title / video(公网URL) / originalVoice / sameStyleVoice
POST /task/query  查进度  status: 1排队 2执行中 3完成 4失败 7取消
```
2026-09-22 实测：`taskId=638474551243630178`，耗时 275s，YouTube 公开 feed 确认上线。
`video` 字段直接收公网 URL，不用先传到 GeeLark。素材托管建议放已有的公开站点（CDN），
**不要为此在运维域名上新开无鉴权静态入口**。

IG / TikTok / Facebook 在 Base 套餐**没有对应 RPA 模板**，才需要下面的 ADB 方案。
能力边界以 GeeLark 实际返回为准，动手前先查项目内的云手机手册（本项目是
`/root/.claude/CLOUD_PHONE.md`）——**这一步跳过的代价是几天的无效排查**。

ADB 方案的本质：`adb push` 推视频进相册 → `am start` 打开 App → `uiautomator dump`
抓界面找元素 → `input tap` 逐个点过去。

参考实现（FreshLock）：`automation/{instagram,tiktok,facebook}_adb_flow.py`，
公共原语在 `automation/social_adb_common.py`；YouTube 走 `youtube_rpa_publish.py`。

## 一、ADB 就绪：唯一正确的时序

**这是最容易错、也最致命的一步。** `adb connect` 返回 success 之后，设备通常先处于 `offline`，要 5–20 秒才转成 `device`。在 offline 状态下发任何 shell 命令都会 `error: device offline`。

❌ **错误做法**（会间歇性失败，慢的时候必挂）：
```python
adb connect IP:PORT
time.sleep(2)                    # 固定等，不够
adb -s IP:PORT shell glogin PWD  # 此时还 offline → 认证失败
adb devices                      # 失败之后才去查状态
```

✅ **正确做法**：
```python
adb disconnect IP:PORT          # 先清残留：旧的 offline 条目会一直脏在设备表里
adb connect IP:PORT
轮询 adb devices 直到该地址状态 == "device"   # 关键，不能用固定 sleep
adb -s IP:PORT shell glogin PWD               # 到 device 之后才认证
再查一次 adb devices 复核没掉回 offline
```

要点：
- **重试次数由总时限约束，不要写死 2 次**。实测见过第一轮 `adb connect` 直接超时（20s）、状态 `absent`，靠第二轮 disconnect+重连救回来
- 解析 `adb devices` 用 `line.split()` 取第二列，**不要用 `endswith("\tdevice")`**，不同 adb 版本分隔符不一定是 tab
- `unauthorized` 状态立即放弃重试，等下去不会自愈
- GeeLark 的 `glogin <pwd>` 是认证步骤，密码从 `adb/getData` 拿

**实测基线**：冷启动（关机状态）→ 开机 → ADB 就绪 → 截图成功，全程 **1 分 43 秒 ~ 2 分 07 秒**。设备已开机时约 30 秒。超过 200 秒还没到 device 就该判失败了。

## 二、等界面渲染：同样必须轮询

App 冷启动后会先白屏。固定 `sleep(8)` 之后截图，拿到的是**纯白图片 + UI 文字 0 条**（2026-09-21 实测两次都这样）。

✅ 正确做法：循环 `uiautomator dump` 抓界面层级，直到解析出的 `text`/`content-desc` 条数达到阈值（比如 ≥3 条）再截图，上限 45 秒。

**第三种表现：App 启动后的闪屏。** 2026-09-22 实测 TikTok 冷启动 15 秒后，前台仍停在
`com.ss.android.ugc.aweme.splash.SplashActivity`，界面上一个可点元素都没有。原来的
`launch_app` 固定 `sleep(4)` 就返回，后续 dump/tap 全部落空，脚本却一路静默跑到结尾。

✅ `launch_app` 正确写法：轮询前台 Activity，**必须已属目标包、且不再停留在 Splash/Launch 页**才返回：
```python
adb(dev,"shell","monkey","-p",pkg,"-c","android.intent.category.LAUNCHER","1")
while time.time() < end:
    time.sleep(3)
    cur = 解析 dumpsys window 的 mCurrentFocus
    if pkg in cur and not re.search(r"(Splash|Launch)", cur, re.I):
        time.sleep(2); return
```

同一个教训在这个项目里**一天之内出现了三次**（ADB connect 后的 offline 期、App 冷启动白屏、
启动闪屏）。所以这不是个案：**凡是"等外部系统就绪"，一律轮询实际状态；代码里出现固定
sleep 就当作缺陷对待。**

## 三、进 App 指定页面：深度链接优先，兜底才点击

想看自家主页（核验发布结果）或跳到指定页面时：

**第一选择：VIEW 深度链接直达，零点击**
```bash
adb -s $DEV shell am start -a android.intent.action.VIEW -d "<url>" -p <包名>
```
`-p` 限定包名，防止被浏览器接管。

各平台实测结果（2026-09-21）：

| 平台 | 包名 | 深度链接 | 结果 |
|---|---|---|---|
| TikTok | `com.zhiliaoapp.musically` | `https://www.tiktok.com/@<handle>` | ✅ 直接成功 |
| YouTube | `com.google.android.youtube` | `https://www.youtube.com/@<handle>` | ✅ 可用 |
| Instagram | `com.instagram.android` | `instagram://user?username=<handle>` 和 https 两种写法 | ❌ 都卡在 `UrlHandlerActivity` 白屏 |
| Facebook | `com.facebook.katana` | `https://www.facebook.com/<page>` | ❌ 被 Google Play 更新弹窗挡住 |

**兜底：正常启动 App + 按标签白名单定位导航入口点一次**
```
am force-stop <包名>
monkey -p <包名> -c android.intent.category.LAUNCHER 1
轮询等渲染
从 uiautomator dump 里按 content-desc/text 精确匹配白名单标签，取 bounds 中心坐标，tap 一次
```
**关键安全边界：只允许点白名单里的导航标签，绝不接受任意坐标。** 各平台"我的主页"标签：
- Instagram: `Profile`
- TikTok: `Profile` / `Me`
- Facebook: `Your profile` / `Profile` / `Menu`
- YouTube: `You` / `Account`

这样即使定位失败也只是找不到元素，不会误触到 Share/Post 把东西发出去。

## 三之二、发布入口：元素名会变，而坐标兜底比失败更糟

2026-09-22 真机逐屏排查 Instagram，查出连续多日失败的根因，值得当反面教材：

脚本找 `desc="Create New"`——**该元素在当前版本根本不存在**。实测首屏底部导航只有
`Home / Reels / Message / Search and explore / Profile` 五项，没有"+"创建键；唯一创建入口是
左上角 `desc="Create a reel"` @[0,53][77,130]。

更糟的是找不到时的兜底逻辑"点屏幕底部居中 `(w/2, h*0.895)`"——该坐标恰好落在信息流帖子的
`See more` @[26,1213][635,1252] 上，等于**点开了别人帖子的展开全文**。流程从这里断掉，之后
所有 tap 全打在空处，脚本一路静默跑到结尾。

✅ 正确做法：
- 找不到入口就**抛错终止**，要求重新真机排查。盲点坐标既不可能成功，还会在别人的信息流里
  乱点，有误互动风险
- 多版本兼容用**多个具名元素备选**（新名 or 旧名），不要用坐标兜底

**各平台创建入口实测（2026-09-22）**：

| 平台 | 创建入口 | 点进去之后 |
|---|---|---|
| Instagram | `desc="Create a reel"`（左上角） | **直接进拍摄相机** `ModalActivity`，要再点左下角 `Gallery` 才到相册 |
| Facebook | 启动即被 Google Play 更新弹窗盖住 | 先点 `desc="Dismiss update dialog"` 才能用 |

注意"点创建 → 直接进相机而非选项菜单"这个模式在 IG 和 YouTube 上都出现过，
**别假设点了"+"会弹出菜单让你选"发帖/快拍/Reel"**。

另外：有些拦路弹窗**只有 content-desc 没有 text**（如 Google Play 更新弹窗的关闭键），
只按 text 匹配的通用关弹窗函数会完全抓不到，需要补一轮 desc 匹配。但 desc 白名单里
**只能收语义明确的纯关闭项**，绝不能混入 Share/Post 这类提交按钮。

## 四、成本控制：用完立刻关机

云手机按分钟计费。**每次操作完必须 `phone/stop`**，不要因为"待会儿可能还要用"就开着。

- 编排脚本用 `trap` 在退出时强制关机（正常退出、报错、被 kill 都要覆盖）
- 另有看门狗兜底：开机超 60 分钟强制关机（见 [[vps-watchdog-pattern]]）
- 单次开机约 2 分钟就绪，多做几件事再关比反复开关划算——但别超过必要时长

## 五、必读的前置检查

动手发布前先确认，否则白跑一趟：

```bash
# 四个 App 装没装（只读查询，不改任何状态）
adb -s $DEV shell pm list packages -3

# App 是不是登录着的 / 有没有弹窗挡路（只读）
adb -s $DEV shell dumpsys window | grep mCurrentFocus
adb -s $DEV shell uiautomator dump /sdcard/t.xml && adb -s $DEV shell cat /sdcard/t.xml
```

实测踩过的坑：
- 主屏看不到社媒图标 ≠ 没装。**App 可能在应用抽屉里**，用 `pm list packages -3` 才准
- Facebook 被 Google Play「Update available」弹窗整个挡住主界面，**这个弹窗同样会挡住发布脚本**，是 FB 发布失败的具体原因。App 本身是登录着的
- YouTube 报 `you should run glogin to login first` = ADB 认证没过，不是账号没登录

## 六、published_hint 不可信（重要）

脚本自报的 `published_hint`（点完 Share 后抓 UI XML 判断有没有 "sharing"/"published" 字样）存在大量假阴性，**不能拿它当"没发出去"去重发**，会造成同平台重复发布。完整原理、实证案例和正确核验方法见 [[publish-verification]]，队列侧的防重发机制见 [[content-queue-scheduling]]。

## 七、只读操作清单（安全，可随时执行）

这些命令不改变手机任何状态，排查时放心用：
```bash
adb -s $DEV exec-out screencap -p > shot.png   # 截图（注意要二进制读，不能 text=True）
adb -s $DEV shell uiautomator dump /sdcard/x.xml   # 抓界面层级
adb -s $DEV shell pm list packages -3              # 列第三方 App
adb -s $DEV shell dumpsys window | grep mCurrentFocus  # 当前前台 Activity
adb devices                                        # 连接状态
```

Python 里取截图要用二进制：`subprocess.run([...], capture_output=True)` 不加 `text=True`，并校验开头是 PNG magic `\x89PNG\r\n\x1a\n`，否则拿到的可能是错误信息而不是图。
