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

# 批量测速（注意：返回结果只含存活节点，选中节点不在列表里 = 已死）
$url = [uri]::EscapeDataString('https://github.com')
$delay = Invoke-Clash 'GET' "/group/NCloud/delay?url=$url&timeout=5000" | ConvertFrom-Json
$delay.PSObject.Properties | Where-Object { $_.Value -is [int] -and $_.Value -gt 0 } |
    ForEach-Object { [pscustomobject]@{ Node = $_.Name; Delay = $_.Value } } |
    Sort-Object Delay | Select-Object -First 10
```

## 第二步: 判断根因

| 诊断结果 | 根因 | 处置 |
|---------|------|------|
| `MODE=global` | Clash 重启后回到全局模式，规则分组失效 | 切回 rule 模式 + 固化配置 |
| 目标域名超时但 delay 显示有节点存活 | 当前选中的节点挂了 | 按下方场景 B 切分组 |
| **ChatGPT 200 但 GitHub/Google 同时超时** | `NCloud` 分组选中的固定节点已死 | NCloud 切 `自动选择`，别固定节点名 |
| **OpenAI 分组 now 是香港节点** | 故障转移组按 gstatic 检测，香港能通但 ChatGPT 不认港区 | OpenAI 手动固定日本/新加坡/美国节点，禁用故障转移 |
| 网络栈测试 200 但应用反复重连 | 应用进程里的旧连接卡住 | 清旧连接 + 重启应用 |
| 所有域名全 FAIL 且 delay 全部超时 | 整批节点失效 | 测速挑存活节点 + 切自动选择 |

> 识别死节点：测速返回结果**只包含存活节点**，所以 `DEAD=0` 不代表全活；如果当前选中的节点名**不在返回列表里**，就是已死。

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

### 场景 C — 应用旧连接卡住

应用（ChatGPT、Git）的进程会缓存失效的代理连接，即使节点修好也继续重连。**必须清理**:

```powershell
# ChatGPT 相关域名
$conns = (Invoke-Clash 'GET' '/connections' | ConvertFrom-Json).connections |
    Where-Object { $_.metadata.host -match 'chatgpt|openai|oaistatic|oaiusercontent' }

# GitHub 相关域名
$conns = (Invoke-Clash 'GET' '/connections' | ConvertFrom-Json).connections |
    Where-Object { $_.metadata.host -match 'github|githubusercontent' }

foreach ($c in $conns) {
    Invoke-Clash 'DELETE' "/connections/$($c.id)"
}

# 重启 ChatGPT（UWP 应用用 shell:AppsFolder）
Stop-Process -Name ChatGPT -Force -ErrorAction SilentlyContinue
Start-Process 'shell:AppsFolder\OpenAI.Codex_2p2nqsd0c76g0!App'
```

## 第四步: 验证

```powershell
# HTTP 连测 3 轮，目标域名全部 200 才算好
foreach ($i in 1..3) {
  $res = @()
  foreach ($u in @('https://chatgpt.com','https://github.com')) {
    try { $x = Invoke-WebRequest $u -Method Head -UseBasicParsing -TimeoutSec 12; $res += "$u=$($x.StatusCode)" }
    catch { $res += "$u=FAIL" }
  }
  "ROUND_$i : $($res -join '  ')"
}

# Git 实际连通验证（比 HTTP 更能证明 GitHub 恢复）
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
git ls-remote https://github.com/balabalahong99/clash-network-fix.git
```

- 连续 2-3 次返回 200 + `git ls-remote` 成功即算修好
- 只测 chatgpt.com 不够：不同域名走不同分组，必须按用户报障的应用分别验证

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
