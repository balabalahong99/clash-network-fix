# clash-network-fix

> Trae CN Skill — 一键诊断并修复 Clash Verge 代理环境下的 ChatGPT / GitHub 网络问题

## 适用场景

- ChatGPT 桌面版反复重连（"正在重新连接 3/5"）
- GitHub 打不开 / git push 超时
- TLS 握手失败（`curl: (35) schannel: failed to receive handshake`）
- 下载 release-assets.githubusercontent.com 很慢或卡死
- 代理切换后应用依然断网

## 诊断流程

1. 从 Clash 内核进程自动提取控制管道（`ext-ctl-pipe`）
2. 读取当前运行配置：`mode`、`ipv6`、各分组选中节点
3. 以目标域名为测试目标批量测速，定位死节点
4. 扫描活动连接，判断是节点挂了还是应用旧连接卡住

## 修复策略

| 根因 | 修复 |
|------|------|
| mode 被 Clash 重启自动改回 `global` | 切回 `rule`，固化到 yaml 文件 |
| 当前节点挂了 | OpenAI → 故障转移 fallback 组；GitHub → 自动选择 url-test 组 |
| 应用旧连接卡住 | 用 `DELETE /connections/{id}` 清除失效代理连接，重启应用 |
| 整批节点失效 | 手动测速筛选可用节点 + 切换故障转移组 |

## 关键约束

- **mode 必须是 `rule`**（不能 `global`），否则规则分组失效
- **ipv6 必须是 `false`**，禁用不稳定的 IPv6 出口
- OpenAI 分组走美国/日本/新加坡节点，**不要走香港**（ChatGPT 不支持港区）
- 订阅节点质量周期性波动，故障转移组是最好的长期方案

## 本机环境信息

- Clash Verge Rev 2.5.10 + verge-mihomo 内核
- 订阅服务商：subsyncsoft.com `/sd/` 路径
- 过期时间：2026-11-21（到期前两天批量换节点是常态）
- Trae CN 全局 skill 目录：`~/.trae-cn/skills/clash-network-fix/`

## 文件

```
SKILL.md    — Trae skill 主体（含 PowerShell 诊断脚本和修复流程）
```
