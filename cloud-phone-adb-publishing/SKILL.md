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

**发布完全靠云手机上的真实 App 完成，不走任何平台 API**：`adb push` 把视频推进相册 → `am start` 打开 App → `uiautomator dump` 抓界面找按钮坐标 → `input tap` 逐个点过去。

参考实现（FreshLock）：`automation/{instagram,tiktok,facebook,youtube}_adb_flow.py`，公共原语在 `automation/social_adb_common.py`。

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

这条和第一节是同一个教训的两种表现：**凡是"等外部系统就绪"，一律轮询实际状态，不要固定 sleep。**

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

各平台脚本判断"发布成功"的方式是点完 Share 后抓一次 UI XML，看有没有 "sharing"/"published" 字样：
```python
published_hint = "sharing" in xml.lower() or "published" in xml.lower()
```
**界面稍变或抓取时机差几秒就检测不到，存在大量假阴性**——明明发出去了却记成"未确认"。实证：`oddly_satisfying_seal` 的 YouTube 明明 2026-09-21 06:27 已经上线，系统里却一条记录都没有。

所以**绝不能拿 `published_hint=false` 当作"没发出去"去重发**，那会造成同平台重复发布。正确的核验方法见 [[publish-verification]]，队列侧的防重发机制见 [[content-queue-scheduling]]。

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
