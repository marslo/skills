---
name: corp-ssh-github-rst-instability
description: "On the corporate/office network, GitHub SSH (port 22) is intermittently killed by a forged-RST injection near the network edge; use HTTPS. Machine-wide, project-agnostic."
metadata:
  node_type: memory
  type: reference
  originSessionId: c5d533da-3e55-46c1-8a01-e745585d0e11
  modified: 2026-08-08T02:39:34.498Z
---

# GitHub SSH (:22) intermittently reset on the corporate network — use HTTPS

Machine/environment fact (project-agnostic). Chinese version: [[corp-ssh-github-rst-instability-zh]]. Related glob reference: [[git-wildmatch-cheatsheet]].

## Ruled-out causes (all NO — verified)

| Suspected cause | Verdict | Why |
|---|---|---|
| 16-way parallelism (`g:plug_threads`) | ❌ no | serial `threads=1` real `:PlugUpdate` still failed en masse; standalone serial SSH burst (`for i in $(seq 30); git ls-remote git@github.com:…`) also failed ~7/16 → not concurrency |
| authentication | ❌ no | fails at SSH *identification/KEX* phase, **before** auth; a valid on-disk key authenticates fine when the session isn't reset; host key matches GitHub |
| ssh-agent / gpg-agent | ❌ no | auth uses an on-disk IdentityFile (no passphrase, agent-independent); works with `SSH_AUTH_SOCK` unset |
| GitHub's own SSH rate-limit | ❌ no | identical config on a **non-corporate network (home)** works; `ssh.github.com:443` and `gitlab.com:22` are fine at the same connection rate; GitHub handles many SSH conns fine |
| DNS redirect / proxy impersonating GitHub | ❌ no | `20.29.134.23` is a **real GitHub IP** (in GitHub `api.github.com/meta` `git`/`web`/`actions` ranges; host key fp = GitHub's published `SHA256_ED25519 +DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`). `dig` vs `ssh` hitting different IPs = GitHub's normal multi-IP round-robin (140.82.x and Azure 20.x) |

## Actual cause (results only)

The **corporate network / VPN path** intermittently **injects a forged TCP RST** into outbound **SSH (port 22)** sessions to GitHub, tearing them down during the SSH handshake. Confirmed corporate (not ISP/GitHub) by a controlled 3-environment test — see "Controlled experiment" below. There are **two enforcement points**: a corp-LAN inline device on corp Wi-Fi (RST TTL 62, `ack 23`, after the SSH banner) and the GlobalProtect VPN gateway when tunneling from home (RST TTL 64, `ack 1`, at SYN — worse). **HTTPS (443) to GitHub is never affected.**

### Commands run + log findings

```bash
# capture on the physical egress interface while reproducing
sudo tcpdump -n -i en0 -vv -tttt 'tcp port 22'
for i in $(seq 30); do ssh -o BatchMode=yes -o ConnectTimeout=6 -T git@github.com >/dev/null 2>&1 || echo "fail $i"; done
```

Findings across 3 captures (RST source = the GitHub IP, but):

| packet (from GitHub IP) | TTL | note |
|---|---|---|
| SYN-ACK / data `[P.]` / FIN `[F.]` (real GitHub) | **49–55** | ~9–15 hops → real GitHub (Azure/140.82.x) |
| **RST `[R.]`, `ack 23`** | **62** | ~2 hops → injected at the **corporate edge**, spoofing GitHub's IP |

- Same source IP cannot emit both TTL 49–55 and TTL 62 ⇒ the RST is **forged/injected by a nearer device**, not sent by GitHub.
- All RSTs land at **`ack 23`** = right after the client's `SSH-2.0-OpenSSH_…` banner (22 bytes) → the injector waits to identify SSH, then kills it.
- RST count per ~30 attempts: **11 / 10 / 1** (intermittent); the rest completed auth and closed with a normal `FIN`.
- `ssh -vv` failure line: `kex_exchange_identification: read: Connection reset by peer`.
- `traceroute github.com (20.29.134.23)`: ~13 hops (corp `10.87.x`/`10.95.x` → Level3 `4.x` → Azure `51.10.x`/`104.44.x`, ~28 ms). With client initial TTL 64: RST TTL 62 ⇒ ~2 hops (corp edge); GitHub TTL ~50 ⇒ ~14 hops (Azure). Confirms the RST originates at the corporate edge.
- GitHub's current SSH banner is `SSH-2.0-7f27de7` (short hash, no longer `babeld-…`) — legit, verified by IP ranges + host key.

### Inferred cause

Intermittent **forged-RST injection by a corporate inline/tap security device** on outbound SSH:22 to GitHub. Intermittency = best-effort / rate-limited injection (a RST-injection race), so a single manual `clone`/`fetch` usually survives, while a burst (`:PlugUpdate` ≈ 74 connections) is hit hard (~57/74 in the worst window).

### Timeline (per failing connection)

```
client → SYN → github:22
client ← SYN-ACK ← github            (TCP established; ttl ~50)
client → ACK
client → "SSH-2.0-OpenSSH_…" banner (22 bytes)
client ← ACK (ack 23) ← github
client ← RST (ack 23, ttl 62) ← [corp edge device spoofing github's IP]   ← killed here
⇒ ssh: kex_exchange_identification: read: Connection reset by peer
```

## Controlled experiment (3 environments) — decisive: corporate, not ISP/GitHub

Same probe (30× `ssh -T git@github.com` + `tcpdump`) run in three environments:

| environment | machine (agents) | path | egress | handshake OK | RST-killed | RST TTL | RST `ack` |
|---|---|---|:--:|:--:|:--:|:--:|:--:|
| corp Mac + corp Wi-Fi | corp (Falcon/Cisco/GP) | corporate LAN | `en0` | ~20/30 | ~10 (33%) | **62** (~2 hops) | **23** (after SSH banner) |
| corp Mac + home Wi-Fi | corp (Falcon/Cisco/GP) | GlobalProtect VPN → corp | **`utun6`** | **4/30** | **26 (87%)** | **64** (0-hop / tunnel) | **1** (at SYN, before banner) |
| personal Mac + home Wi-Fi | none | home ISP, direct | `en0` | **30/30** | **0** | — | — |

- **personal Mac on the same home ISP = 30/30, zero RST** (30 complete handshakes, GitHub banner `SSH-2.0-7f27de7`, normal FIN) ⇒ home ISP / region / GitHub do **NOT** interfere → rules out any ISP/regional/GFW theory.
- The injection **follows the corporate path**, with **two distinct signatures/enforcement points**:
  - corp Wi-Fi → corp-LAN inline device: TTL **62** (~2 hops), reset **after** the SSH banner (`ack 23`), ~33% hit.
  - home + GlobalProtect VPN (egress `utun6`, all traffic tunneled back to corp) → VPN gateway / tunnel ingress: TTL **64** (0-hop), reset **at SYN** (`ack 1`, before the banner), ~87% hit (worse & earlier).
- ⇒ **Root cause = the corporate network/VPN path injecting forged RSTs into GitHub SSH:22.** Not ISP, not GitHub, not concurrency, not auth.
- Not fully isolated: whether an on-host agent also contributes (VPN was up at home); but the differing network-position TTLs (62 vs 64) point to network chokepoints, and the personal-Mac control confirms the corporate path.

## Local security / VPN / audit stack on this machine (observed inventory)

Discovery commands:

```bash
systemextensionsctl list
pgrep -fl -i 'falcon|crowdstrike|cisco|beyondtrust|globalprotect|csc'
scutil --proxy ; env | grep -i proxy ; git config --get-regexp proxy
```

Active macOS system **network extensions** (`systemextensionsctl list` → `[activated enabled]`):

| bundle id | product | role |
|---|---|---|
| `com.crowdstrike.falcon.Agent` (7.37) | CrowdStrike Falcon Sensor | EDR |
| `com.paloaltonetworks.GlobalProtect.client.extension` (6.2.8) | Palo Alto GlobalProtect | VPN + content filter / transparent proxy |
| `com.beyondtrust.endpointsecurity` (24.8.0.1) | BeyondTrust Endpoint Security | endpoint privilege/security |

Running processes: CrowdStrike agent; GlobalProtect / `PanGPS`; Cisco Secure Client `/opt/cisco/secureclient/bin/csc_iseposture`; macOS `socketfilterfw`.

System Settings → Network → **VPN & Filters** (overall status = **Active**):

| name | type | status |
|---|---|---|
| Falcon | Content Filter | Enabled |
| Cisco Secure… | Content Filter | Enabled |
| GlobalProtectEn | Content Filter | Disabled |
| GlobalProtectDo | Transparent Proxy | Disabled |
| GlobalProtectAp | VPN | Disconnected |

Notes:

- `scutil --proxy` empty, no `*_proxy` env, no `git http.proxy` ⇒ egress control is via **on-device `NEFilterDataProvider` / transparent-proxy network extensions**, not a classic HTTP proxy (HTTPS to GitHub went direct in traces).
- `~/.ssh/config.d/*` contains **commented** `ProxyCommand` lines — the sanctioned "SSH over HTTP-proxy CONNECT" tunnels: `corkscrew ipamunix.domain.com 8080 %h %p`, `nc -X connect -x ipamunix.domain.com:8080 %h %p`, `127.0.0.1:1087`. (Proxy `ipamunix.domain.com:8080` is lab-only per user; enabling one of these is the approved way to run GitHub SSH through an auditable proxy.)
- **Cisco Secure Client** (`/opt/cisco/secureclient/bin/csc_iseposture --fullposture`; "Cisco Secure…" = Content Filter **Enabled**): does endpoint **posture / trusted-network detection** — it identifies whether the host is on the **corporate network (office Wi-Fi)** vs **off-corp (home Wi-Fi / WFH)** and applies **different policies per environment**. This matches the location-dependence observed here (GitHub SSH:22 killed on corp Wi-Fi, fine at home): the stricter egress policy is active only on the corporate network. Its role is location/posture gating; the RST itself is injected at the network edge (~2 hops).
- **Relationship to the RST injection (kept rigorous):** the above are the endpoint security/VPN/audit tools present on the host and are *candidate* enforcers. However the packet evidence (RST TTL **62** ≈ 2 hops vs GitHub's real **49–55** ≈ ~14 hops) points to a device at the **corporate network edge (~2 hops)** — i.e. more likely a **network appliance** than a 0-hop on-host filter. The exact enforcer is not pinned from the host; confirming it needs the network/security team.

## Why `git https-set` works but `git ssh-set` doesn't

Both toggle which include (`credential.ssh` vs `credential.https`) is active. Content:

`ssh-set` on → this rewrite forces **every** github URL onto SSH:22 → hits the RST injection → unstable:

```gitconfig
[url "git@github.com:"]
  insteadOf = https://github.com/
  insteadOf = git@ssh.github.com:
  insteadOf = ssh://git@github.com/
```

`https-set` on → the reverse rewrite forces everything onto HTTPS:443 → not injected → stable:

```gitconfig
[url "https://github.com/"]
  insteadOf = git@github.com:
  insteadOf = git@ssh.github.com:
  insteadOf = git@github-marslo.com:
```

## Fix / recommendation

- **Public repos (e.g. vim-plug plugins)**: use HTTPS. Store remotes as `https://git::@github.com/OWNER/REPO.git` — the `git::@` userinfo makes the URL **not** match `insteadOf https://github.com/`, so it stays HTTPS even with `ssh-set` active (verified: reaches github over 443 anonymously). `vim-plug` default `g:plug_url_format` already produces this form.
- **Own / internal repos** (`ghe-marslo`, `vgitcentral.domain.com` gerrit, `sj1git1.cavium.com`, etc.): keep SSH — that traffic stays **inside** the corporate network and is not subject to the GitHub-egress RST injection.
- Net rule: **transport follows reliability** — external GitHub over HTTPS:443, internal hosts over SSH.
