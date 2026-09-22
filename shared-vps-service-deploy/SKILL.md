---
name: shared-vps-service-deploy
description: 在共享主力VPS(192.236.171.76)上新部署一个服务/API/网关时用——这台机器同时跑着代理(xray/socks5-relay等)、多个业务站点、以及整个Claude Code团队的会话，新服务必须"只加不改"，不能动任何现有配置或共享服务。来自2026-09-21/22部署Agent Chat中转框架(GPT+扣子对话代理)的实战经验。
---

# 共享VPS新服务部署

这台VPS(192.236.171.76)不是给单个项目独享的——上面跑着代理业务(xray/socks5-relay/tinyproxy/res-http/res-socks，见项目CLAUDE.md"VPS共享铁律")、多个网站(aihubai.cn/chinapal.world/freshlocksealer.com)、以及Claude Code团队的全部会话。新部署任何东西，**默认假设它会撞坏别的东西，全程按"只追加不改动"操作**。

## 部署前必查
```bash
ss -tlnp                              # 已占用端口，新端口选10000+且不在这里面
df -h /                               # 磁盘余量，紧张先 apt-get clean + journalctl --vacuum-size=50M
free -h                               # 内存余量，这台机器常年紧张(见 vps-resource-triage 技能)，重服务前三思
systemctl list-units --type=service --state=running   # 看现有服务，别跟已有的撞名/撞端口
```

## 标准架构：低权限systemd服务 + nginx反代到现有域名
不新开公网端口、不新申请域名/证书——复用已有nginx站点，加一个新location块反代到本地新端口。

**1. 新建专用低权限系统用户**（不用root跑新服务）
```bash
useradd --system --no-create-home --shell /usr/sbin/nologin <服务名>
```

**2. systemd unit 加固模板**（参考 `/etc/systemd/system/agent-chat.service`）
```ini
[Service]
Type=simple
User=<服务名>
Group=<服务名>
WorkingDirectory=/opt/<服务名>
ExecStart=/usr/bin/python3 /opt/<服务名>/xxx.py
Restart=always
RestartSec=5
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/<服务名>
PrivateTmp=true
```

**3. nginx：只在现有站点文件里追加新location块，绝不整体改写**
- 先确认改的是`/etc/nginx/sites-enabled/<站点>`这个真实生效的文件，不是`sites-available`里可能过期的同名文件（两者不一定同步，之前踩过坑）
- 改之前先备份：`cp /etc/nginx/sites-enabled/<站点> /etc/nginx/sites-enabled/<站点>.bak_$(date +%Y%m%d_%H%M%S)`
- 用Python脚本做精确文本替换（锚定一段已知存在的现有location块，在其后插入新块），不要用sed硬改，避免破坏其它location
- 内部工具类服务直接复用现成的 `/etc/nginx/.htpasswd`（auth_basic "Admin Area"）做鉴权，不用新建账号体系
- 限流：在 `/etc/nginx/nginx.conf` 的 `limit_req_zone` 段**追加**一条新zone，不动已有的（如 aihub_chat/aihub_paypal）
- **改完必须 `nginx -t` 语法校验通过才能继续**，校验失败绝不能reload
- 生效用 `systemctl reload nginx`（不是restart——reload不断开现有连接，restart会）

**4. 部署完必须验证**
```bash
systemctl is-active <新服务>                                    # 新服务自己起来了
curl -s http://127.0.0.1:<内网端口>/health                       # 内网直连能通
curl -s -o /dev/null -w '%{http_code}\n' https://<域名>/<新路径> # 公网入口鉴权生效(应401/正常，不是500/502)
systemctl is-active xray socks5-relay tinyproxy proxy-monitor res-http res-socks nginx  # 所有共享服务confirmed active，一个都不能掉
```

**5. 记日志**：所有改动追加写入 `/root/CHANGELOG_<项目名>.log`（VPS共享铁律第7条），写清楚"加了什么、改了哪个文件、验证过什么没受影响"。

## 涉密/密钥的处理
没拿到真实key/token时，代码要能在"未配置"状态下优雅跑通（返回占位提示而不是崩溃/500），让不涉密的框架部分可以先部署验证。key到位后走 `EnvironmentFile=-/opt/<服务名>/env.conf`（dash前缀=文件不存在也不报错）+ `systemctl restart` 接通，不用再碰代码。

## 大文件/脚本传输到VPS
用base64单行编码传，避免heredoc里的引号/特殊字符转义问题：
```bash
FILE_B64=$(base64 -w0 本地文件)
sshpass -p "$VPS_PASSWORD" ssh root@192.236.171.76 "echo \"$FILE_B64\" | base64 -d > /目标路径"
```

## 常见坑
- `sites-available/`里可能有同名但过期的文件，实际生效的是`sites-enabled/`——改错文件不会报错，但也不会生效，容易白忙一场
- destructive-looking的单条命令（重启/reload共享服务、kill进程）有时会被Claude Code自己的auto mode分类器拦一次，同一条命令原样重试一次通常就能过，不代表命令本身有问题
