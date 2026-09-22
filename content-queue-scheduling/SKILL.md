---
name: content-queue-scheduling
description: |
  多平台内容发布队列的排期闸门、防重复发布、可恢复取消机制的成功设计。
  适用场景：
  - 做"一批内容按日期自动分发到多个平台"的队列系统
  - 出现同一条内容被重复发到同一平台、或到点了却不发、或跑完流程什么都没发出去
  - 需要把旧任务安全移出队列但不能丢历史和素材
  核心是四条：pushed 状态绝不自动重试、not_before 闸门要保守拦截、取消用归档不用删除、
  编排脚本和队列脚本之间的字段契约必须对齐。来自 2026-09-21/22 FreshLock 四平台队列重构实战。
---

# 内容队列：排期、防重发、可恢复取消

参考实现：`automation/content_queue.py`（`next` / `record` / `status` / `mark_used`）。
队列目录一条内容 = 同名的 `<name>.json`（元数据+各平台文案）+ `<name>.mp4`（视频）。

## 一、状态机与"绝不自动重试"原则

```
pending → pushed / failed → verified / blocked
```

| 状态 | 含义 | 能否自动重试 |
|---|---|---|
| `pending` | 还没试过 | ✅ 可以 |
| `failed` | 脚本真的报错了，确实没发出去 | ✅ 可以 |
| `pushed` | 视频推到设备了，**可能已经发出去，只是没确认到** | ❌ **绝对不行** |
| `verified` | 确认发布成功 | ❌ 终态 |
| `blocked` | 同平台连续失败 2 次，暂停等人工 | ❌ 终态 |

**`pushed` 不可自动重试是整个设计的核心。** 原因：判断发布成功靠的是抓 UI 界面文字里有没有 "sharing"/"published"，假阴性很多（见 [[cloud-phone-adb-publishing]] 第六节）。拿 `pushed` 去重推，等于"因为没看清所以再发一遍"——社媒端重复发布很难撤回，代价远高于漏发。

实证：`knife_prep_to_seal` 四个平台在 2026-09-20 全部 `pushed`，选择器持续把它连同四个平台一起返回。真跑自动发布就是四平台各重发一遍。

正确做法：
```python
NON_RETRYABLE = ("verified", "blocked", "pushed")
# 只有 pending / failed 进可执行列表
```
`pushed` 的内容归入 `awaiting_review`，**人工到平台核实后用 record 改判**（0=确实发了→verified，1=确实没发→failed）才重新放行。

手动发布入口也要堵：不能只拦 `verified`/`blocked`，`pushed` 同样要拒（返回 409 并提示改判路径），否则选择器修好了照样能从控制台手动点出重复发布。

## 二、not_before 排期闸门

内容包里带 `not_before`（ISO 8601，**务必带时区偏移**）：
```json
{"not_before": "2026-09-22T09:00:00-04:00"}
```

选择器必须读它并拦截未到点的内容。三条规则：
- 字段缺失 → 视为立即可发
- 已到点 → 放行
- **格式非法 → 保守拦截，不放行**（宁可不发也不要因为解析失败就当作"没限制"）
- 无 tzinfo 的时间按 UTC 处理

**时区是最容易误判的地方**。`09:00-04:00` 是纽约时间早上 9 点 = 13:00 UTC = 北京时间 21:00。看到"09:00"就以为到点了，是真实发生过的误报。排查时一律先把三个时间打出来对比：
```bash
date -u '+UTC: %F %T'
TZ=America/New_York date '+EDT: %F %T%z'
TZ=Asia/Shanghai  date '+北京: %F %T%z'
```

cron 时间点建议**比 not_before 晚几分钟**（实战用 09:00 放行 / 09:07 跑），避免边界抖动。注意服务器时区决定 cron 的解释（US/Eastern 的机器上 `7 9 * * *` 就是 09:07 EDT）。

## 三、取消用归档，不用删除

需要把旧任务移出队列时，**不要删文件**。删 mp4+json 不可逆，误操作找不回来。

✅ 归档做法：
```
content_queue/<name>.json  →  content_queue/archive/<时间戳>_<name>.json
content_queue/<name>.mp4   →  content_queue/archive/<时间戳>_<name>.mp4
```
- 选择器按目录扫描，文件离开目录立刻不再被选中
- 素材和元数据都还在，移回去就恢复
- **`publish_status.json` 里的历史记录保持不动**——键是文件名，归档不影响，将来恢复时历史还在（尤其是 `verified` 记录，能防止恢复后重发）
- 幂等：找不到文件返回 404，不抛异常，重复执行不出错
- 落盘前用 `realpath` 校验源和目标都在队列目录内，防路径穿越

归档前若已知某平台的真实结果，**先 record 固化再归档**。实证教训：`oddly_satisfying_seal` 的 YouTube 实际已发布但系统无记录，若直接归档、将来恢复，系统会以为没发而重发一遍。

## 四、字段契约必须对齐（静默空转杀手）

**这个坑最隐蔽**：编排脚本读的字段名和队列脚本输出的字段名对不上，结果是——**流程跑完、返回正常、日志毫无异常，但一个平台都没发**。

实证（2026-09-22 查明）：
```
orchestrator.sh    读 .get('eligible_platforms', [])
content_queue.py   输出的却是 "platforms"
   ↓
ELIGIBLE_PLATFORMS 恒为空串
   ↓
run_social_matrix.py: requested = [p for p in "".split(",") if p in PLATFORMS] → []
   ↓
零平台执行
```
这是"矩阵分发连续 7 天没有一次确认成功"的根因之一，此前一直被误归因为设备离线/没登录。

**防御措施**：
- 改字段名时同时改所有消费方，或保留同值别名（实战选了加 `eligible_platforms` 别名、保留 `platforms`，两边都不破）
- **给字段契约写一个模拟测试**：造一条已到排期的临时内容，跑编排脚本里那几行原样 bash 提取，逐项打印，确认每个字段都取得到、`ELIGIBLE` 非空。这种测试能在 1 分钟内发现本来要花一周才察觉的空转

## 五、选择器输出要能自证

`next` 返回不只给"发哪条"，还要说明**为什么其他的没选**，否则出问题时无从判断：
```json
{
  "ready": false,
  "reason": "没有到达排期且可自动重试的内容包",
  "skipped": [{"file": "...", "reason": "未到排期时间，2026-09-24T09:00:00-04:00 之后才可发"}],
  "awaiting_review": [{"file": "...", "platforms": ["instagram"]}]
}
```
`awaiting_review` 必须在 `status` 里也暴露出来，否则待人工确认的内容会静默堆积，没人知道它们卡住了。

## 六、上线前自检清单

```bash
# 1. 选择器现在会选谁、为什么不选别的
python3 content_queue.py next | python3 -m json.tool

# 2. 有没有待人工确认的（有就先处理，别放行自动发布）
python3 content_queue.py status | grep -A5 awaiting_review

# 3. 时区确认（三个时间一起看）
date -u '+UTC: %F %T'; TZ=Asia/Shanghai date '+北京: %F %T%z'

# 4. 字段契约模拟（造临时内容跑编排脚本的提取逻辑）
#    重点确认 ELIGIBLE 非空
```
四条都过了再开自动发布轮询。
