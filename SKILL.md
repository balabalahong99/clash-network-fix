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
- 订阅节点会成批失效，常见死节点标记: `美国 08 Gemini 测试`
- OpenAI 分组已设置为"故障转移"组 fallback，节点坏时自动切换

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
"MODE=$($config.mode) IPV6=$($config.ipv6)"
try {
    $r = Invoke-WebRequest 'https://chatgpt.com' -Method Head -UseBasicParsing -TimeoutSec 12
    "CHATGPT=$($r.StatusCode)"
} catch { "CHATGPT=FAIL: $($_.Exception.Message)" }
# 批量测速（以目标域名为测试目标）
Invoke-Clash 'GET' '/group/OpenAI/delay?url=https%3A%2F%2Fchatgpt.com&timeout=8000'
```

## 第二步: 判断根因

| 诊断结果 | 根因 | 处置 |
|---------|------|------|
| `MODE=global` | Clash 重启后回到全局模式，规则分组失效 | 切回 rule 模式 + 固化配置 |
| 目标域名超时但 delay 显示有节点存活 | 当前选中的节点挂了 | 切到故障转移组 fallback |
| 网络栈测试 200 但应用反复重连 | 应用进程里的旧连接卡住 | 清旧连接 + 重启应用 |
| `CHATGPT=FAIL` 且 delay 全部超时 | 整批节点失效 | 手动测速选可用节点 + 切故障转移 |

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

**OpenAI 分组**: 首选切到 `故障转移`（fallback 组，自动轮询）。如果 fallback 也不可用，从 delay 结果里挑一个延迟最低的节点。

**GitHub/NCloud 分组**: 切到 `自动选择`（url-test 组，最低延迟）。

```powershell
# 切 OpenAI 到故障转移
Invoke-Clash 'PUT' '/proxies/OpenAI' (@{name='故障转移'} | ConvertTo-Json -Compress)
# 切 NCloud 到自动选择
Invoke-Clash 'PUT' '/proxies/NCloud' (@{name='自动选择'} | ConvertTo-Json -Compress)
```

**固化**: 更新 `profiles.yaml` 对应 selected 块的 `now:` 值。

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
1..3 | ForEach-Object {
    $n = $_
    try { $r = Invoke-WebRequest 'https://chatgpt.com' -Method Head -UseBasicParsing -TimeoutSec 12
        "TEST_$n=$($r.StatusCode)" } catch { "TEST_$n=FAIL" }
}
```

连续 2-3 次返回 200 即算修好。

## 关键文件路径速查

| 用途 | 路径 |
|------|------|
| clash-verge.yaml（mode 固化） | `C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\clash-verge.yaml` |
| runtime/config.yaml | `C:\ProgramData\clash-verge-service\users\<hash>\runtime\config.yaml` |
| profiles.yaml（节点选择固化） | `C:\Users\Administrator\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\profiles.yaml` |
| ChatGPT 启动命令 | `shell:AppsFolder\OpenAI.Codex_2p2nqsd0c76g0!App` |

## 注意事项

- 不要把 OpenAI 分组切到香港节点（ChatGPT 不支持港区出口）
- Clash Verge 每次重启可能会丢失 UI 里的 mode 设置，**必须固化到 yaml 文件**
- 故障转移组每 300 秒自动检测，url-test 组也是，它们都会自动避开死节点
- 这个订阅的节点质量周期性波动，成批失效是常态，故障转移是最好的长期方案
