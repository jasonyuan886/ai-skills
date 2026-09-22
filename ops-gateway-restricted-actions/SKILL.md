---
name: ops-gateway-restricted-actions
description: |
  给运维网关增加"受限动作"（白名单 API，禁止任意 Shell）的设计模式与安全边界。
  适用场景：
  - 要让人或其他 Agent 通过网页/API 触发服务器上的运维操作，但不能开放任意命令执行
  - 需要接收上传文件并导入系统，且必须防路径穿越/zip bomb/覆盖既有数据
  - 网关加了动作却调不通、或上传报 413、或表单提交 500
  核心是：动作白名单 + 三套鉴权不动 + realpath 围栏 + 归档不删除 + 幂等 + 上传链路端到端对齐。
  来自 2026-09-21/22 FreshLock Ops Gateway 的实战加固。
---

# 受限动作网关设计模式

参考实现：`automation/task_gateway.py`（Flask + HTTPS，nginx 反代）。
设计原则是**白名单动作，禁止任意 Shell**——网关只暴露一组预定义动作，每个动作的参数和副作用都被严格约束。

## 一、三套接入方式，鉴权各自独立

| 方式 | 用途 | 鉴权 |
|---|---|---|
| Bearer Token API | 其他 Agent/脚本调用 | `Authorization: Bearer <token>` |
| Session + 外部 JS | 网页控制台（增强） | Cookie Session + CSRF |
| 原生 POST 表单 | 无 JS 兜底 | Session + CSRF，服务端渲染结果 |

**加新动作时这三套鉴权一律不动**。常见错误是只在 Bearer 路径上测通了就以为完事。

## 二、加一个新动作的完整检查表

漏掉任何一步，动作要么调不通、要么有安全缺口：

```
□ 1. 加进 VALID_ACTIONS 白名单
□ 2. 写 handler，参数全部校验（见第三节）
□ 3. 注册进 Bearer API 的 handlers 字典
□ 4. 注册进 console_execute 的 handlers 字典        ← 最常漏
□ 5. 在 console_execute 里为该动作组装 params      ← 最常漏，漏了参数恒为空
□ 6. 加控制台表单（含 csrf_token 隐藏域）
□ 7. 两条路径都实测：Bearer API + 网页表单
```

**实战踩过的坑**：`console_execute` 里 `params` 只给 `publish` 组装，新动作拿到的永远是空 `{}`，导致参数校验必定失败、**该动作从网页控制台根本调不通**，但 Bearer API 测试是通过的，所以一直没被发现。

## 三、参数校验：白名单 + realpath 围栏

**名称类参数用正则白名单**，不要用黑名单过滤：
```python
if not re.match(r'^[a-zA-Z0-9_]+$', name):   # 只允许字母数字下划线
    return jsonify({"error": "Invalid name"}), 400
```
这一条同时挡住了 `../../etc/passwd`、`abc;rm -rf /`、带连字符的畸形名。

**落盘前必须做 realpath 围栏**，确认目标真的在允许目录内：
```python
root = os.path.realpath(ALLOWED_DIR)
dest = os.path.realpath(os.path.join(root, os.path.basename(name)))
if os.path.dirname(dest) != root:
    return jsonify({"error": "路径越界"}), 400
```

**对外输出文件的路由也要白名单文件名**：
```python
if not re.fullmatch(r"screen_\d{8}_\d{6}\.png", fname):
    return jsonify({"error": "Invalid name"}), 400
```
并且该路由要求与控制台同等鉴权，文件存在 web 根目录之外（含敏感画面时尤其重要，如已登录社媒账号的截图，目录 0700 / 文件 0600）。

## 四、接收上传：ZIP 导入的完整防御

只接受一个固定包名，并逐条校验成员：

```
□ 文件名必须等于约定的包名
□ 仅接受顶层文件：任何含 / 或 \ 的条目直接拒绝
□ 拒绝目录条目（info.is_dir()）
□ 拒绝软链接：(info.external_attr >> 16) & 0o170000 == 0o120000
□ 拒绝隐藏文件（以 . 开头）
□ 拒绝含 .. 的名字
□ 扩展名白名单（如只允许 .mp4/.json）
□ zip bomb 防护：条目数上限 + 解压后总体积上限（边遍历边累加，超了立刻拒）
□ 已存在的同名内容一律跳过，不覆盖
```

**"仅接受顶层"这条很容易漏**：`sub/a.json` 不含 `..`、不以 `/` 开头、扩展名合法，能通过常见检查被解压出来，而顶层 `os.listdir()` 又扫不到它 —— 结果是留下一堆无主文件。

## 五、上传体积：三段链路必须一起对齐

上传报 `413 Request Entity Too Large` 时，要同时看三处：

| 层 | 配置 | 实战值 |
|---|---|---|
| nginx | `client_max_body_size` | `200m` |
| Flask | `MAX_CONTENT_LENGTH` | `200 * 1024 * 1024` |
| 业务层 | 解压后总体积上限 | `300MB` |

nginx 那条**要加在具体 server 块里，不要加全局**，避免影响同机其他站点。改完必须：
```bash
nginx -t && systemctl reload nginx     # reload 不是 restart，不断其他站点连接
```
**双向验证**：传一个略小于上限的（应穿透到应用层，返回鉴权/校验错误而不是 413）、再传一个超上限的（应被 nginx 挡下返回 413）。只测一个方向可能把上限改成了事实上的无限制。

## 六、删除类操作：一律做成可恢复

**不提供真正的删除。** 用归档代替：
```
<目录>/<name>.ext  →  <目录>/archive/<时间戳>_<name>.ext
```
好处：选择器按目录扫描，文件一离开目录立刻失效；素材还在，移回去就恢复；误操作代价接近零。

配套要求：
- **幂等**：对象不存在时返回 404 并说明，不抛异常，重复执行不报错
- 归档不动关联的历史记录文件（键是文件名，恢复时历史还在）
- 日志用 `logger.warning` 记录谁在什么时候归档了什么

## 七、复用共享逻辑，不要在网关里重写

需要队列状态时调既有脚本取 JSON，不要在网关里重新实现一套：
```python
stdout, _, _ = safe_run_python("content_queue.py", ["status"])
queue_status = json.loads(stdout) if stdout else {}
```
两套实现迟早不一致。注意消费方读的字段名要和脚本输出对齐（这类契约错位会造成静默空转，详见 [[content-queue-scheduling]] 第四节）。

## 八、Python 里两个真实咬过人的低级错误

**变量自我覆盖**：
```python
with open(sf) as f: sd = json.load(f)        # f 变成文件对象
with open(f)  as f: pset = set(json.load(f)) # 把文件对象传进 open() → TypeError
```
只要第二个文件存在就必然 500。修法是用独立变量名（`f2`）。

**二进制输出用了 text 模式**：截图 `exec-out screencap -p` 要用 `capture_output=True` **不加** `text=True`，并校验 PNG magic `\x89PNG\r\n\x1a\n`，否则拿到的可能是被当成文本解码坏掉的数据或错误信息。

## 九、改完必做

```bash
python3 -m py_compile <file>.py          # 先编译
systemctl restart <service>              # 再重启
systemctl is-active <service>            # 确认起来了
curl -sk https://127.0.0.1:<port>/health # 健康检查

# 然后两条路径都要实测：
# - Bearer API：正常参数 + 各种非法参数（穿越/特殊字符/不存在对象）
# - 网页表单：确认 params 真的组装了、CSRF 通过、结果能渲染
```
非法输入的拒绝路径**必须实测**，不能只看代码觉得对。实战就是靠逐个试 `../../etc/passwd`、`abc;rm -rf /` 才确认围栏真的生效。
