# keepalive-ping

云端定时唤醒探针：让一个部署在 Render 免费实例上的内部看板保持在线，消除"每次打开都要等 30–60 秒冷启动"的问题。

## 为什么单独建一个公开仓库

GitHub Actions 对**公开仓库**的标准 runner 免费不限量，对**私有仓库**只提供 2,000 分钟/月。
保活需要常年占用运行时间（24 小时连续在线 = 43,200 分钟/月），放进私有数据仓库会产生高额计费，
所以探针放在这个不含任何业务代码与数据的公开仓库里。

代价：公开仓库的 workflow 文件、运行日志、提交历史都是公开的。因此：

- 目标地址保存在 **`KEEPALIVE_URL`** 仓库 Secret 中，文件里只有 `${{ secrets.KEEPALIVE_URL }}`；
- 日志只打印 HTTP 状态码，不打印地址，避免把线上入口暴露给扫描者。

## 组成

| 文件 | 作用 |
| --- | --- |
| `.github/workflows/keep-alive.yml` | 每 5 分钟探测一次目标地址。每次运行探测两次（间隔 4 分钟），即使 GitHub 延迟了下一次调度，实例也不会闲置到休眠。运行失败不报警，避免邮件噪音。 |
| `.github/workflows/heartbeat.yml` | 每天提交一次 `heartbeat.txt`。公开仓库若 60 天无任何活动，GitHub 会自动停用定时工作流；这个提交用于防止被停用。 |

## 目标地址

地址是 `业务看板入口 + /health`（健康检查路径不需要登录），已写入仓库 Secret：

```bash
gh secret set KEEPALIVE_URL --repo <owner>/keepalive-ping --body 'https://<host>/health'
gh secret list --repo <owner>/keepalive-ping
```

修改指向时只需重设这个 Secret，不用改代码。

## 验证

```bash
# 手动触发一次并查看结果
gh workflow run keep-alive.yml --repo <owner>/keepalive-ping
gh run list --repo <owner>/keepalive-ping --limit 5
gh run view --repo <owner>/keepalive-ping --log

# 直接确认目标当前是醒着的
curl -s -o /dev/null -w '%{http_code} %{time_total}s\n' https://<host>/health
```

健康时状态码为 `200`，冷启动时首字节要等 30–60 秒；保活正常工作时响应时间应稳定在 1 秒级。

## 已知边界

**实测：GitHub 的 cron 在本仓库没有派发过（2026-09-21，尚未恢复）。**
建仓后 87 分钟、覆盖 17 个以上 5 分钟档期，`schedule` 事件触发次数为 **0**。已排除：

- 工作流状态是 `active`（未被停用）；
- 默认分支上的 YAML 确认是 `on: schedule:` 加两条 cron（`*/5` 与 `2-57/5`）；
- GitHub 状态页全绿，不是平台故障；
- `workflow_dispatch` 手动触发完全正常（`ping -> HTTP 200`），Actions 本身可用。

问题在 GitHub 定时调度器的派发环节，不在本仓库配置。社区有同类报告，但本仓库的
等待时间已超出"最长约 1 小时"的常见说法。**因此不要把本仓库当成保活的唯一保障**：
它只是免费冗余，调度器哪天开始派发就自动叠加一层保险；真正的保障是 Google Apps
Script（每 5 分钟触发，跑在 Google 服务器上）。

其他边界：

- 即使调度器恢复，`schedule` 在高峰期仍可能被延迟，极端情况下少量排队任务会被丢弃。本仓库用"每轮两次探测 + 4 分钟间隔"来容忍这一点，但不能保证 100%。
- 需要 Render 免费实例的月度额度足够（免费实例 750 小时/月；单实例常年在线约 720–744 小时/月，刚好够用）。若再开第二个免费实例会被挤爆额度。
- 这是免费额度的变通用法。如果需要生产级稳定性，正确做法是升级 Render 付费实例（不休眠）或迁到常驻主机。
