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

- GitHub 的 `schedule` 在高峰期可能被延迟，极端情况下少量排队任务会被丢弃。本仓库用"每轮两次探测 + 4 分钟间隔"来容忍这一点，但不能保证 100%。
- 需要 Render 免费实例的月度额度足够（免费实例 750 小时/月；单实例常年在线约 720–744 小时/月，刚好够用）。若再开第二个免费实例会被挤爆额度。
- 这是免费额度的变通用法。如果需要生产级稳定性，正确做法是升级 Render 付费实例（不休眠）或迁到常驻主机。
