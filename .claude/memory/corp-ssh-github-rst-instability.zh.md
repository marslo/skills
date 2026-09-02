---
name: corp-ssh-github-rst-instability-zh
description: "公司/office 网络下,到 GitHub 的 SSH(22 端口)被网络边缘伪造 RST 间歇性掐断;应改用 HTTPS。机器级、与项目无关。"
metadata:
  node_type: memory
  type: reference
  originSessionId: c5d533da-3e55-46c1-8a01-e745585d0e11
  modified: 2026-08-08T02:39:51.850Z
---

# 公司网络下 GitHub SSH(:22)被间歇 RST 掐断 —— 改用 HTTPS

机器/环境事实(与项目无关)。英文版:[[corp-ssh-github-rst-instability]]。相关 glob 参考:[[git-wildmatch-cheatsheet-zh]]。

## 已排除的原因(全部为"否",均经实测)

| 怀疑原因 | 结论 | 依据 |
|---|---|---|
| 16 并行(`g:plug_threads`) | ❌ 否 | 串行 `threads=1` 的真实 `:PlugUpdate` 依然大面积失败;独立串行 SSH burst(`git ls-remote git@github.com:…`)也 ~7/16 挂 → 与并发无关 |
| 认证 | ❌ 否 | 失败在 SSH *标识/KEX* 阶段,**早于**认证;不被 reset 时有效密钥能正常认证;host key 与 GitHub 匹配 |
| ssh-agent / gpg-agent | ❌ 否 | 认证用的是磁盘上的 IdentityFile(无 passphrase、不依赖 agent);`SSH_AUTH_SOCK` 清空也能成 |
| GitHub 自身 SSH 限流 | ❌ 否 | 同一份配置在**非公司网络(家里)**正常;同样频率下 `ssh.github.com:443` 与 `gitlab.com:22` 都没事;GitHub 扛得住大量 SSH |
| DNS 改道 / 代理冒充 GitHub | ❌ 否 | `20.29.134.23` 是**真 GitHub IP**(在 `api.github.com/meta` 的 `git`/`web`/`actions` 段;host key 指纹 = GitHub 官方 `SHA256_ED25519 +DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`)。`dig` 与 `ssh` 命中不同 IP = GitHub 正常的多 IP 轮询(140.82.x 与 Azure 20.x) |

## 真实原因(只写结果,不写动机)

**公司的网络 / VPN 路径**间歇性地往到 GitHub 的 SSH(22 端口)会话里注入伪造 TCP RST,在 SSH 握手阶段掐断。经三环境受控实验证实是**公司侧**(非 ISP/GitHub)—— 见下方"受控实验"。有**两个执法点**:公司 wifi 下的公司 LAN inline 设备(RST TTL 62、`ack 23`、banner 之后),以及在家经 GlobalProtect VPN 隧道时的 VPN 网关(RST TTL 64、`ack 1`、SYN 时 —— 更狠)。**到 GitHub 的 HTTPS(443)从不受影响。**

### 运行的命令 + 日志发现

```bash
# 在物理出口网卡抓包,同时复现失败
sudo tcpdump -n -i en0 -vv -tttt 'tcp port 22'
for i in $(seq 30); do ssh -o BatchMode=yes -o ConnectTimeout=6 -T git@github.com >/dev/null 2>&1 || echo "fail $i"; done
```

三次抓包发现(RST 源伪装成 GitHub IP,但):

| 包(源=GitHub IP) | TTL | 说明 |
|---|---|---|
| SYN-ACK / 数据 `[P.]` / FIN `[F.]`(真 GitHub) | **49~55** | ~9-15 跳 → 真 GitHub(Azure / 140.82.x) |
| **RST `[R.]`,`ack 23`** | **62** | ~2 跳 → 在**公司边缘**伪造 GitHub IP 注入 |

- 同一源 IP 不可能同时发 TTL 49~55 和 TTL 62 ⇒ RST 是**近端设备伪造注入**,不是 GitHub 发的。
- 所有 RST 都落在 **`ack 23`** = 客户端发完 `SSH-2.0-OpenSSH_…` banner(22 字节)之后 → 注入者等识别出 SSH 再掐。
- 每 ~30 次的 RST 数:**11 / 10 / 1**(间歇);其余完成认证、以正常 `FIN` 关闭。
- `ssh -vv` 失败行:`kex_exchange_identification: read: Connection reset by peer`。
- `traceroute github.com (20.29.134.23)`:~13 跳(公司 `10.87.x`/`10.95.x` → Level3 `4.x` → Azure `51.10.x`/`104.44.x`,~28 ms)。按客户端初始 TTL 64:RST TTL 62 ⇒ ~2 跳(公司边缘);GitHub TTL ~50 ⇒ ~14 跳(Azure)。证实 RST 来自公司边缘。
- GitHub 现在的 SSH banner 是 `SSH-2.0-7f27de7`(短 hash,不再是 `babeld-…`)—— 真 GitHub,已由 IP 段 + host key 双证。

### 推断的原因

公司内网一台 **inline/旁路安全设备间歇性伪造注入 RST**,专掐到 GitHub 的出站 SSH:22。间歇 = 尽力而为 / 限速注入(一场竞速),所以单条手动 `clone`/`fetch` 常常存活,而 burst(`:PlugUpdate` ≈ 74 条连接)被大量命中(最差窗口 ~57/74)。

### 时间线(单条失败连接)

```
client → SYN → github:22
client ← SYN-ACK ← github            (TCP 建立;ttl ~50)
client → ACK
client → "SSH-2.0-OpenSSH_…" banner(22 字节)
client ← ACK (ack 23) ← github
client ← RST (ack 23, ttl 62) ← [公司边缘设备伪造 github IP]   ← 在此被掐
⇒ ssh: kex_exchange_identification: read: Connection reset by peer
```

## 受控实验(三环境)—— 决定性:公司侧,非 ISP/GitHub

同一探针(30× `ssh -T git@github.com` + `tcpdump`)在三个环境各跑一次:

| 环境 | 机器(agent) | 路径 | 出口 | 握手 OK | RST 掐 | RST TTL | RST `ack` |
|---|---|---|:--:|:--:|:--:|:--:|:--:|
| 公司 Mac + 公司 Wi-Fi | 公司(Falcon/Cisco/GP) | 公司 LAN | `en0` | ~20/30 | ~10 (33%) | **62**(~2跳) | **23**(banner 后) |
| 公司 Mac + 家 Wi-Fi | 公司(Falcon/Cisco/GP) | GlobalProtect VPN → 公司 | **`utun6`** | **4/30** | **26 (87%)** | **64**(0跳/隧道) | **1**(SYN 时,banner 前) |
| 私人 Mac + 家 Wi-Fi | 无 | 家 ISP 直连 | `en0` | **30/30** | **0** | — | — |

- **私人 Mac 在同一家 ISP = 30/30、零 RST**(30 次完整握手、github banner `SSH-2.0-7f27de7`、正常 FIN)⇒ 家 ISP / 地区 / GitHub **不**干扰 → 排除任何 ISP/地区/GFW 说法。
- 注入**跟着公司路径走**,有**两个不同签名/执法点**:
  - 公司 Wi-Fi → 公司 LAN inline 设备:TTL **62**(~2跳),**banner 之后**(`ack 23`)掐,~33%。
  - 家里 + GlobalProtect VPN(出口 `utun6`,全部流量隧道回公司)→ VPN 网关/隧道入口:TTL **64**(0跳),**SYN 时**(`ack 1`,banner 前)掐,~87%(更狠更早)。
- ⇒ **根因 = 公司网络/VPN 路径向 GitHub SSH:22 注入伪造 RST。** 非 ISP、非 GitHub、非并发、非认证。
- 未完全隔离:是否还有端侧 agent 也在注入(家里 VPN 是连着的);但两处不同的网络位置 TTL(62 vs 64)指向网络层的执法点,加上私人 Mac 对照,确证是公司路径。

## 本机的 VPN / 安全 / 审计工具(实际观察到的清单)

排查命令:

```bash
systemextensionsctl list
pgrep -fl -i 'falcon|crowdstrike|cisco|beyondtrust|globalprotect|csc'
scutil --proxy ; env | grep -i proxy ; git config --get-regexp proxy
```

已激活的 macOS 系统**网络扩展**(`systemextensionsctl list` → `[activated enabled]`):

| bundle id | 产品 | 角色 |
|---|---|---|
| `com.crowdstrike.falcon.Agent` (7.37) | CrowdStrike Falcon Sensor | EDR |
| `com.paloaltonetworks.GlobalProtect.client.extension` (6.2.8) | Palo Alto GlobalProtect | VPN + 内容过滤 / 透明代理 |
| `com.beyondtrust.endpointsecurity` (24.8.0.1) | BeyondTrust Endpoint Security | 端点权限/安全 |

运行中的进程:CrowdStrike agent;GlobalProtect / `PanGPS`;Cisco Secure Client `/opt/cisco/secureclient/bin/csc_iseposture`;macOS `socketfilterfw`。

系统设置 → 网络 → **VPN & Filters**(总状态 = **Active**):

| 名称 | 类型 | 状态 |
|---|---|---|
| Falcon | Content Filter | Enabled |
| Cisco Secure… | Content Filter | Enabled |
| GlobalProtectEn | Content Filter | Disabled |
| GlobalProtectDo | Transparent Proxy | Disabled |
| GlobalProtectAp | VPN | Disconnected |

说明:

- `scutil --proxy` 为空、无 `*_proxy` 环境变量、无 `git http.proxy` ⇒ 出网管控走的是**端侧 `NEFilterDataProvider` / 透明代理网络扩展**,不是传统 HTTP 代理(抓包里 HTTPS 是直连 GitHub 的)。
- `~/.ssh/config.d/*` 里有**注释掉的** `ProxyCommand` —— 即"SSH 经 HTTP 代理 CONNECT 隧道"的合规写法:`corkscrew ipamunix.domain.com 8080 %h %p`、`nc -X connect -x ipamunix.domain.com:8080 %h %p`、`127.0.0.1:1087`。(代理 `ipamunix.domain.com:8080` 据用户说仅 lab 用;要在公司用 GitHub SSH,启用其中一条即可把 SSH 裹进可审计代理。)
- **Cisco Secure Client**(`/opt/cisco/secureclient/bin/csc_iseposture --fullposture`;面板里 "Cisco Secure…" = Content Filter **Enabled**):做端点 **posture / 可信网络检测** —— 判断当前在**公司网络(office Wi-Fi)**还是**非公司网络(家里 Wi-Fi / WFH)**,并**按环境套用不同策略**。这与本次观察到的"位置相关性"吻合:GitHub SSH:22 在公司 Wi-Fi 被掐、在家正常,即更严的出网策略只在公司网络生效。它的角色是位置/合规判定 + 策略门控;RST 本身是在网络边缘(~2 跳)注入的。
- **与 RST 注入的关系(严谨表述):** 以上是本机上存在的端点安全/VPN/审计工具,属**候选**执法者。但包证据(RST TTL **62** ≈ 2 跳,vs GitHub 真包 **49~55** ≈ ~14 跳)指向**公司网络边缘(~2 跳)**的设备 —— 更像**网络设备**,而非 0 跳的本机过滤器。确切执法者本机无法指认,需网络/安全团队确认。

## 为什么 `git https-set` 行、`git ssh-set` 不行

两者切换启用哪个 include(`credential.ssh` vs `credential.https`)。内容:

`ssh-set` 开 → 这条改写把**所有** github URL 强制走 SSH:22 → 撞上 RST 注入 → 不稳:

```gitconfig
[url "git@github.com:"]
  insteadOf = https://github.com/
  insteadOf = git@ssh.github.com:
  insteadOf = ssh://git@github.com/
```

`https-set` 开 → 反向改写把一切强制走 HTTPS:443 → 不被注入 → 稳:

```gitconfig
[url "https://github.com/"]
  insteadOf = git@github.com:
  insteadOf = git@ssh.github.com:
  insteadOf = git@github-marslo.com:
```

## 修复 / 建议

- **公开仓库(如 vim-plug 插件)**:走 HTTPS。remote 存成 `https://git::@github.com/OWNER/REPO.git` —— `git::@`(userinfo)使 URL **不**匹配 `insteadOf https://github.com/`,即使 `ssh-set` 开着也保持 HTTPS(实测:匿名走 443 到 github 成功)。`vim-plug` 默认 `g:plug_url_format` 就是这个形式。
- **自己的 / 内网仓库**(`ghe-marslo`、gerrit `vgitcentral.domain.com`、`sj1git1.cavium.com` 等):保留 SSH —— 那些流量在**公司内网内部**,不经过 GitHub 出网的 RST 注入。
- 总原则:**传输跟着可靠性走** —— 外部 GitHub 走 HTTPS:443,内网主机走 SSH。
