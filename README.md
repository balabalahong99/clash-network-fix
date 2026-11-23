# clash-network-fix

> Trae CN Skill — 一键诊断并修复 Clash Verge 代理环境下的 ChatGPT / GitHub 网络问题

## 最新更新（2026-11-23，commit e867db4）

这次补了订阅过期的完整处理流程，因为 11/21 那次整站断网的根因就是订阅到期 14 分钟：

- **新增场景 D — 订阅过期**：节点测速只剩 DIRECT 或全超时 + 订阅已过期时，不要反复切节点/改 rule/重启应用——没用的，机场已经不再鉴权
- **诊断脚本加 SUB_EXPIRE 输出**：自动从 `profiles.yaml` 里读 `expire` Unix 时间戳转日期，一眼看到期剩余天数
- **诊断脚本加 DIRECT_BAIDU 直连检查**：区分"代理层挂了"还是"本机网络挂了"
- **判断根因表拆分**：原"所有域名 FAIL + delay 全超时"拆成"订阅过期"和"节点批量失效"两条独立场景
- **注意事项补充**：节点服务器 TCP 通但代理 HTTPS 全 TLS 握手失败 / "连接被关闭"，这是订阅过期的典型表现

## 适用场景

- ChatGPT 桌面版反复重连（"正在重新连接 3/5"）
- GitHub 打不开 / git push 超时
- TLS 握手失败（`curl: (35) schannel: failed to receive handshake`）
- 下载 release-assets.githubusercontent.com 很慢或卡死
- 代理切换后应用依然断网
- 所有节点测速只剩 DIRECT 或全超时（先查订阅到期！）

## 诊断流程

1. 从 Clash 内核进程自动提取控制管道（`ext-ctl-pipe`）
2. 读取当前运行配置：`mode`、`ipv6`、各分组选中节点
3. 以目标域名为测试目标批量测速，定位死节点
4. 扫描活动连接，判断是节点挂了还是应用旧连接卡住
5. 读 `profiles.yaml` 的 `expire` 字段看订阅是否过期
6. 直连百度验证本机基础网络是否正常

## 修复策略

| 根因 | 修复 |
|------|------|
| mode 被 Clash 重启自动改回 `global` | 切回 `rule`，固化到 `clash-verge.yaml` 和 `runtime/config.yaml` 两个文件 |
| 当前节点挂了 | OpenAI → 固定非港区节点（新加坡直连）；GitHub → `自动选择` url-test 组 |
| 应用旧连接卡住 | 用 `DELETE /connections/{id}` 清除失效代理连接，重启 ChatGPT |
| **订阅过期** | **续费 → Clash Verge 更新订阅 → 重新诊断**，不要反复切节点 |
| 节点批量失效（机场质量波动） | 测速筛选存活节点 + 切自动选择 |

## 关键约束

- **mode 必须是 `rule`**（不能 `global`），否则规则分组失效
- **ipv6 必须是 `false`**，禁用不稳定的 IPv6 出口
- OpenAI 分组走美国/日本/新加坡节点，**不要走香港**（ChatGPT 不支持港区）
- OpenAI 分组**不要用故障转移**——检测 URL 是 gstatic.com，香港节点能通但 ChatGPT 不认港区，表现为能测通却反复断网
- NCloud 分组**不要固定节点名**——订阅节点会成批失效（日本 A、美国 08 都死过），固定节点必然周期性断网，`自动选择` 才能自愈

## 本机环境信息

- Clash Verge Rev 2.5.10 + verge-mihomo 内核
- 订阅服务商：subsyncsoft.com `/sd/` 路径
- 当前过期时间：2026-12-23（2026-11-21 那次过期 14 分钟导致整站断网）
- Trae CN 全局 skill 目录：`~/.trae-cn/skills/clash-network-fix/`

## 文件

```
SKILL.md    — Trae skill 主体（含 PowerShell 诊断脚本和完整修复流程）
```
