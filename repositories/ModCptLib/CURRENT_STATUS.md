# ModCptLib Current Status and Snapshot Drift

> 核对日期：2026-08-21（Asia/Tokyo）  
> 历史快照：`master@add9e4024f8e10055508770aaf96c944aab6cceb`  
> 核对时活跃开发线：`AutoSave@4e67f62baec2381a29bb48964fac0e18189a9822`

## 结论

本目录的设计正文是可追溯的 2026-08-09 历史快照，但不再代表 ModCptLib 当前状态。活跃开发线相对快照领先 172 个提交；树级比较有 663 个文件发生变化，其中约 157 个代码文件和 316 个 Markdown 文档。

## 当前权威状态

- M3 已于 2026-08-15 因 Account v3 / Product State v2 生产闭环缺口重新打开。
- W1 shared Account v3 contract 已关闭；W2 为 ACTIVE；W3-W8 依赖前置出口，仍冻结或未完成。
- M7 保持 OPEN。
- negotiated ALPN 当前只以 `/2` 为支持锚点；`/1` 已退役，`/3` 仅作未来准备。
- 历史 `/1` Mailbox 24h soak 只能作为资源与容错取证，不能替代当前 `/2 + MCMBX002` 两主机 24h 发布/SLO 证据。
- 最新活跃 SHA 尚无 exact-SHA GitHub Actions 成功记录；历史绿色 SHA、local full 或 dry-run 不能替代当前发布门。
- DHT 是 legacy 非产品模块；Room/媒体被冻结；relay、STUN/TURN 与 NAT fallback 未实现。

## 当前来源

- [Current Roadmap](https://github.com/StarrySky7D4/ModCptLib/blob/AutoSave/Knowledge/ROADMAP.md)
- [Current Defects](https://github.com/StarrySky7D4/ModCptLib/blob/AutoSave/Knowledge/DEFECTS.md)
- [Current execution board](https://github.com/StarrySky7D4/ModCptLib/blob/AutoSave/agents_work/BOARD.md)
- [2026-08-15 static audit](https://github.com/StarrySky7D4/ModCptLib/blob/AutoSave/Knowledge/2026-08-15-static-audit-autosave-m7.md)
- [Snapshot-to-active comparison](https://github.com/StarrySky7D4/ModCptLib/compare/add9e4024f8e10055508770aaf96c944aab6cceb...4e67f62baec2381a29bb48964fac0e18189a9822)

## Reading rule

需要复原 2026-08-09 设计时阅读本快照；需要判断当前实现、缺陷、兼容性或可发布性时，必须回到活跃源仓库并绑定具体 commit SHA。不要混合两个时点的“已完成”声明。
