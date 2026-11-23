---
name: clash-network-fix
description: Diagnose and fix Trae ChatGPT GitHub network issues on this machine. Use when the user says ChatGPT GitHub or apps cannot connect, reconnection loops, or proxy speeds are slow. Do not use for general programming or non-network tasks.
---

# Clash Verge 网络修复

本机 Clash Verge 代理环境下的网络问题排查与修复流程。覆盖 ChatGPT、GitHub 等应用无法连接、反复重连、TLS 握手失败、超时等场景。

## 前置信息

- Clash Verge 服务进程名: `verge-mihomo.exe`，PID 不固定
- 控制管道: 从进程命令行提取 `-ext-ctl-pipe \\.\pipe\<pipe-name>`
- 配置文件:
  - Clash Verge UI 配置: `C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\clash-verge.yaml`
  - 运行时配置: `C:\ProgramData\clash-verge-service\users\<hash>\runtime\config.yaml`
  - 节点选择持久化: `C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\profiles.yaml`
- 已知必须规则: **mode 必须是 rule**（不能 global），**ipv6 必须是 false**
- 订阅节点会成批失效，常见死节点标记: `美国 08 Gemini 测试`、`日本 A`
- 分组约定（重要，别改回去）:
  - `OpenAI` → 固定选非港区节点（如 `[Vless] 日本 [直连]`），**不要用"故障转移"组**
  - `NCloud`（GitHub/Google）→ 选 `自动选择`（url-test），**不要固定某个节点名**

## 第一步: 诊断

跑下面的 PowerShell 脚本，一次性拿到所有关键信息:

```powershell
# === 从进程命令行提取控制管道 ===
$cmd = (Get-CimInstance Win32_Process -Filter "Name='verge-mihomo.exe'").CommandLine
if ($cmd -notmatch '-ext-ctl-pipe\s+\\\\\.\\pipe\\(\S+)') { throw '未找到 Clash 控制管道' }
$script:pipeName = $Matches[1]

# === 通用 Clash REST 调用函数 ===
function Invoke-Clash([string]$Method, [string]$Path, [string]$Body = '') {
    $pipe = [System.IO.Pipes.NamedPipeClientStream]::new('.', $script:pipeName, [System.IO.Pipes.PipeDirection]::InOut)
    try {
        $pipe.Connect(3000)
        $bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($Body)
        $headers = "$Method $Path HTTP/1.0`r`nHost: localhost`r`nAuthorization: Bearer set-your-secret`r`nContent-Type: application/json`r`nContent-Length: $($bodyBytes.Length)`r`n`r`n"
        $headBytes = [System.Text.Encoding]::ASCII.GetBytes($headers)
        $pipe.Write($headBytes, 0, $headBytes.Length)
        if ($bodyBytes.Length) { $pipe.Write($bodyBytes, 0, $bodyBytes.Length) }
        $reader = New-Object System.IO.StreamReader($pipe)
        $response = $reader.ReadToEnd()
        $reader.Dispose()
        if ($response.IndexOf("`r`n`r`n") -lt 0) { return '' }
        return $response.Substring($response.IndexOf("`r`n`r`n") + 4)
    } catch {
        try { $pipe.Dispose() } catch {}
        return ''
    }
}

# === 核心诊断 ===
$config = Invoke-Clash 'GET' '/configs' | ConvertFrom-Json
"MODE=$($config.mode) IPV6=$($config.ipv6) PORT=$($config.'mixed-port')"

# 分组当前选中的节点（关键：先看它是不是已知死节点）
foreach ($g in @('OpenAI','NCloud','自动选择','故障转移')) {
    $p = Invoke-Clash 'GET' "/proxies/$([uri]::EscapeDataString($g))" | ConvertFrom-Json
    if ($p) { "$g : type=$($p.type) now=[$($p.now)]" }
}

# 分域名测试：不同域名走不同分组，必须分开测才能定位是哪个分组挂了
foreach ($u in @('https://chatgpt.com','https://github.com','https://www.google.com')) {
    try { $r = Invoke-WebRequest $u -Method Head -UseBasicParsing -TimeoutSec 12
        "$u => $($r.StatusCode)" } catch { "$u => FAIL" }
}

# 批量测速（注意：返回结果只含存活节点，选中节点不在列表里 = 已死；只有 DIRECT 活着 = 订阅可能过期）
$url = [uri]::EscapeDataString('https://github.com')
$delay = Invoke-Clash 'GET' "/group/NCloud/delay?url=$url&timeout=5000" | ConvertFrom-Json
$delay.PSObject.Properties | Where-Object { $_.Value -is [int] -and $_.Value -gt 0 } |
    ForEach-Object { [pscustomobject]@{ Node = $_.Name; Delay = $_.Value } } |
    Sort-Object Delay | Select-Object -First 10

# 订阅到期时间（从 profiles.yaml 里读，是 Unix 时间戳）
$profPath = "C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\profiles.yaml"
$expireLine = Select-String -Path $profPath -Pattern '^\s+expire:\s*(\d+)' | Select-Object -First 1
if ($expireLine) {
    $expire = [int64]$expireLine.Matches[0].Groups[1].Value
    $expireDt = [DateTimeOffset]::FromUnixTimeSeconds($expire).ToLocalTime()
    $daysLeft = [math]::Round(($expireDt - [DateTimeOffset]::Now).TotalDays, 1)
    "SUB_EXPIRE=$($expireDt.ToString('yyyy-MM-dd HH:mm'))  剩余${daysLeft}天"
}

# 本机基础网络检查（直连百度，不走代理）
$wc = New-Object System.Net.WebClient; $wc.Proxy = $null; $wc.Headers.Add('User-Agent','Mozilla/5.0')
try { $s = $wc.DownloadString('https://www.baidu.com'); "DIRECT_BAIDU=OK" } catch { "DIRECT_BAIDU=FAIL" }

# ChatGPT 进程数（正常 5-8 个；>10 说明僵尸进程堆积，是之前强杀残留的）
$gptProcs = Get-Process ChatGPT -ErrorAction SilentlyContinue
"CHATGPT_PROCS=$(@($gptProcs).Count)"
if (@($gptProcs).Count -gt 8) {
    "  ⚠️ 僵尸进程堆积！最早启动=$((@($gptProcs | Sort-Object StartTime)[0].StartTime.ToString('HH:mm:ss')))"
}

# Clash 活动连接概览
$conns = (Invoke-Clash 'GET' '/connections' | ConvertFrom-Json).connections
"TOTAL_CONNS=$($conns.Count)"
$conns | Where-Object { -not $_.closed } | Group-Object { $_.metadata.host -replace '\d+','N' } | Sort-Object Count -Descending |
    Select-Object -First 8 | ForEach-Object { "  $($_.Name) x$($_.Count)" }
```

## 第二步: 判断根因

| 诊断结果 | 根因 | 处置 |
|---------|------|------|
| `MODE=global` | Clash 重启后回到全局模式，规则分组失效 | 切回 rule 模式 + 固化配置 |
| 目标域名超时但 delay 显示有节点存活 | 当前选中的节点挂了 | 按下方场景 B 切分组 |
| **ChatGPT 200 但 GitHub/Google 同时超时** | `NCloud` 分组选中的固定节点已死 | NCloud 切 `自动选择`，别固定节点名 |
| **OpenAI 分组 now 是香港节点** | 故障转移组按 gstatic 检测，香港能通但 ChatGPT 不认港区 | OpenAI 手动固定日本/新加坡/美国节点，禁用故障转移 |
| 网络栈测试 200 但应用反复重连 | 应用进程里的旧连接卡住 | 清旧连接 + 重启应用 |
| **HTTP HEAD 连测全 200 但 git ls-remote 超时 / ChatGPT 只转圈不回答** | **僵尸进程堆积 + 旧代理连接积压**（HEAD 被 fake-ip 骗过，实际 TLS 连接被挤） | 杀僵尸进程 + 清连接 + 重启应用 |
| **CHATGPT_PROCS > 8** | 之前强杀 ChatGPT 留下的僵尸进程 | 场景 C 处理（杀进程 + 清连接 + 重启） |
| **所有域名 FAIL + 测速只剩 DIRECT 或全超时 + 订阅已过期** | **订阅到期，节点鉴权全失效**（TCP 通但代理握手失败） | 续费订阅 → Clash Verge 更新订阅 → 节点恢复后再按场景切分组 |
| 所有域名 FAIL + 测速剩少数节点 + 订阅未过期 | 整批节点失效（机场质量波动） | 测速挑存活节点 + 切自动选择 |

> 识别死节点：测速返回结果**只包含存活节点**，所以 `ALIVE=0` 或只剩 `DIRECT` 时首先查订阅到期；如果订阅还有效，再判断是节点批量失效。

> 订阅过期的典型表现：节点服务器 TCP 可连（如 `hkgtv.hkdsync.com:443`），但走代理的 HTTPS 请求全部报"连接被关闭"或 TLS 握手失败，本机直连百度正常。

## 第三步: 修复

### 场景 A — mode 被改回 global

```powershell
# 运行时切回 rule
Invoke-Clash 'PATCH' '/configs' '{"mode":"rule"}'
# 固化到两个配置文件
# clash-verge.yaml: mode: global -> mode: rule
# runtime/config.yaml: mode: global -> mode: rule
```

### 场景 B — 当前节点挂了，切换分组

- **OpenAI 分组**: 固定选非港区节点，优先 `[Vless] 日本 [直连]`，其次 `[Vless] 新加坡 1 [直连]`、`美国 09`。
  - 与该节点名称必须与 `/proxies/OpenAI` 的 `all` 列表完全一致（含方括号和空格）
- **NCloud 分组（GitHub/Google）**: 一律切 `自动选择`（url-test，自动避开死节点），**不要固定具体节点名**

```powershell
# OpenAI 固定日本直连
Invoke-Clash 'PUT' '/proxies/OpenAI' '{"name":"[Vless] 日本 [直连]"}'
# NCloud 切自动选择
Invoke-Clash 'PUT' '/proxies/NCloud' '{"name":"自动选择"}'

# 切换后确认生效（PUT 返回 204 不代表真的是目标值，必须回读）
(Invoke-Clash 'GET' '/proxies/OpenAI' | ConvertFrom-Json).now
(Invoke-Clash 'GET' '/proxies/NCloud' | ConvertFrom-Json).now
```

**固化**: 更新 `profiles.yaml` 中 `type: remote` 那段 `selected:` 块里的 `now:` 值。
注意该文件会被订阅更新/Clash UI 操作重写，`selected` 里的 OpenAI 条目可能整个消失，发现没了就补回。

### 场景 C — 僵尸进程 + 旧连接堆积（最常见的"代理好但应用坏"）

这是最容易被忽略的场景：**HTTP HEAD 连测全 200，但 git ls-remote 超时、ChatGPT 只转圈不回答**。HEAD 请求被 fake-ip 应答骗过了，但实际 TLS 长连接被积压的僵尸进程和旧代理连接挤死了。

**判定线索**：
- `CHATGPT_PROCS > 8`（正常应 5-8 个）
- 活动连接里 `closed=false` 的 OpenAI/GitHub 连接数明显偏多
- HTTP HEAD 全 200，但 `.NET Invoke-WebRequest -Proxy` 走真实代理端口时超时或很慢

**修复（严格按顺序，每步必须做）**：

```powershell
# 1. 先清旧连接（所有 ChatGPT + GitHub 相关，带 closed 也清，防止残留）
$conns = (Invoke-Clash 'GET' '/connections' | ConvertFrom-Json).connections |
    Where-Object { $_.metadata.host -match 'chatgpt|openai|oaistatic|oaiusercontent|github|githubusercontent' }
foreach ($c in $conns) { Invoke-Clash 'DELETE' "/connections/$($c.id)" }
"清连接=$($conns.Count)"

# 2. 杀 ChatGPT 进程（全部，不管僵尸还是正常，彻底清干净）
$procs = Get-Process ChatGPT -ErrorAction SilentlyContinue
"杀进程=$(@($procs).Count)"
$procs | Stop-Process -Force -ErrorAction SilentlyContinue

# 3. 等待完全退出再重启（至少 4 秒，让 Clash 释放所有被占的端口）
Start-Sleep -Seconds 4

# 4. 重启 ChatGPT（UWP 应用用 shell:AppsFolder）
Start-Process 'shell:AppsFolder\OpenAI.Codex_2p2nqsd0c76g0!App'
Start-Sleep -Seconds 10   # 等进程起来再看

# 5. 验证进程数回到正常范围
$newProcs = @(Get-Process ChatGPT -ErrorAction SilentlyContinue)
"重启后进程=$($newProcs.Count)  (正常 5-8)"
```

**关键教训**：
- 不要只清连接就重启——僵尸进程还在的话，又会快速产生新的坏连接
- 不要只重启不清连接——旧连接不会自动释放，新进程还是会走到同一批死连接上
- 旧连接和僵尸进程必须**一起清**，顺序是：清连接 → 杀进程 → 等 → 重启 → 验证

### 场景 D — 订阅过期（所有节点全挂）

**关键判定**：`SUB_EXPIRE` 日期在今天之前，且测速只有 DIRECT 活着或全超时，直连百度正常。

**注意**：这种情况**不是代理配置问题**，不要反复切节点、改 rule 或重启应用——没用的。节点服务器 TCP 虽然能通，但机场已经不再给你鉴权了。

```powershell
# 1. 先确认订阅确实过期
$profPath = "C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\profiles.yaml"
$expireLine = Select-String -Path $profPath -Pattern '^\s+expire:\s*(\d+)' | Select-Object -First 1
$expire = [int64]$expireLine.Matches[0].Groups[1].Value
"SUB_EXPIRE: $( [DateTimeOffset]::FromUnixTimeSeconds($expire).ToLocalTime() )   NOW: $(Get-Date)"

# 2. 关闭 Clash Verge，等用户续费完后再开（或者 Clash Verge 界面点"更新订阅"）
#    如果 TUN 开着先关掉，否则国内网站也被 TUN 接管
```

**续费后**：
1. Clash Verge → 订阅 → 手动更新订阅（或等自动更新，1440 分钟一次）
2. 更新完节点列表刷新，重新跑第一步诊断
3. 按实际情况再切 OpenAI / NCloud 分组

## 第四步: 验证

```powershell
# HTTP HEAD 连测 3 轮，目标域名全部 200 才算好
# ⚠️ 但 HTTP HEAD 全 200 不够——它会被 fake-ip 应答骗过，
#    必须再走一次真实 TLS 长连接（git ls-remote / 应用实际调用）
foreach ($i in 1..3) {
  $res = @()
  foreach ($u in @('https://chatgpt.com','https://github.com')) {
    try { $x = Invoke-WebRequest $u -Method Head -UseBasicParsing -TimeoutSec 12; $res += "$u=$($x.StatusCode)" }
    catch { $res += "$u=FAIL" }
  }
  "ROUND_$i : $($res -join '  ')"
}

# Git 实际连通验证（比 HTTP HEAD 更能证明 GitHub 真恢复了）
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
git ls-remote https://github.com/balabalahong99/clash-network-fix.git 2>&1 | Select-Object -First 3

# ChatGPT 进程数恢复正常范围
$finalProcs = @(Get-Process ChatGPT -ErrorAction SilentlyContinue)
"CHATGPT_PROCS=$($finalProcs.Count)  (正常 5-8)"
if ($finalProcs.Count -gt 8) { "  ⚠️ 僵尸进程还没清干净，再来一轮场景 C" }
```

- 连续 2-3 次返回 200 + `git ls-remote` 成功 + ChatGPT 进程数回到 5-8 = 完全修好
- **只测 HTTP HEAD 不够**：fake-ip 会把 HEAD 请求本地应答掉，但实际走代理的 TLS 连接还没好

## 关键文件路径速查

| 用途 | 路径 |
|------|------|
| clash-verge.yaml（mode 固化） | `C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\clash-verge.yaml` |
| runtime/config.yaml | `C:\ProgramData\clash-verge-service\users\<hash>\runtime\config.yaml` |
| profiles.yaml（节点选择固化） | `C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\profiles.yaml` |
| ChatGPT 启动命令 | `shell:AppsFolder\OpenAI.Codex_2p2nqsd0c76g0!App` |

## 注意事项

- 不要把 OpenAI 分组切到香港节点（ChatGPT 不支持港区出口）
- **故障转移组不能给 OpenAI 用**：它的检测 URL 是 gstatic.com，香港节点能通就会被选中，但 ChatGPT 拒绝港区出口，表现为能测通却反复断网
- **NCloud 不要固定节点名**：订阅节点会成批失效（`日本 A`、`美国 08 Gemini 测试` 都死过），固定节点 = 必然周期性断网，用 `自动选择` 才能自愈
- Clash Verge 每次重启可能会丢失 UI 里的 mode 设置，**必须固化到 yaml 文件**
- `profiles.yaml` 会被订阅更新重写，固化后的 `now:` 需要复查
- 切换分组的 PUT 返回 204 只代表请求被接受，**必须回读 `/proxies/<组名>` 的 `now` 确认真的生效**
- 这个订阅的节点质量周期性波动，成批失效是常态
- **所有节点测速只剩 DIRECT 或全超时**时，第一反应是查订阅到期时间（`profiles.yaml` 里 `expire` 字段），不要浪费时间反复切节点
- 订阅到期的典型表现：节点服务器 TCP 可连（`Test-NetConnection` 通），但走代理的 HTTPS 全部 TLS 握手失败或"连接被关闭"，本机直连百度正常
- **僵尸进程是 ChatGPT 桌面版的固有问题**：多次强杀 / 重启会残留子进程，正常状态应 5-8 个，超过 10 个必须清；**只清连接不清进程** → 僵尸进程又产生新坏连接，**只重启不清连接** → 新进程继续走同一批死连接，两者必须一起清
- **HTTP HEAD 全 200 不等于代理真的通**：fake-ip 会把 HEAD 请求在本地应答掉，走真实 TLS 长连接的 git / curl / 应用 API 才是金标准；修完必须用 `git ls-remote` 或应用实际调用验证
- Clash Verge 的 fake-ip 模式下，**HEAD 请求和实际请求走的路径可能不同**——HEAD 被本地 fake-ip 应答绕过了代理，实际请求才真正走节点，所以 HEAD 全绿不代表代理没问题
