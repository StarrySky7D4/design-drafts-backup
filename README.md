# Design Drafts Backup

集中保存现有及未来项目的设计草案。所有内容按源仓库分目录归档，保留原始相对路径，并记录可追溯的源分支、源提交和备份日期。

## 仓库结构

```text
repositories/
├── PRIVATE_SOURCE_BACKUPS.md
└── <source-repository>/
    ├── README.md
    ├── BACKUP_SOURCE.md
    ├── LICENSE / NOTICE / LICENSE_STATUS.md
    └── <selected-source-paths>/
```

- 每个分库的 `README.md` 提供内容导航；选择性快照必须明确标注日期，不得冒充源仓库当前状态。
- `BACKUP_SOURCE.md` 记录来源、提交、授权与实际收录清单。
- 文档尽量保留源仓库中的相对路径，便于追踪和后续增量备份。

## 当前备份

| 分库 | 来源可见性 | 主要内容 | 许可 |
| --- | --- | --- | --- |
| [`decentralized-chat-architecture`](repositories/decentralized-chat-architecture/README.md) | 公开 | 系统架构、模块契约、ADR、威胁模型及未来匿名覆盖网络方案 | MIT |
| [`ModCptLib`](repositories/ModCptLib/README.md) | 私有，已授权公开备份 | 2026-08-09 历史设计快照；另附 2026-08-26 当前状态与漂移说明 | 参见分库 `LICENSE` |
| [`VSCode_mobile`](repositories/VSCode_mobile/README.md) | 私有，已授权公开备份 | 项目编码方案、架构、安全模型、协议草案、路线图及 ADR | 源提交未发现根许可证，参见 `LICENSE_STATUS.md` |
| [`NewAPI-private`](repositories/NewAPI-private/README.md) | 私有，已授权公开备份 | React → Flutter 迁移清单 | AGPL-3.0，参见分库 `LICENSE` 与 `NOTICE` |

三个私有来源的统一授权与索引见 [`repositories/PRIVATE_SOURCE_BACKUPS.md`](repositories/PRIVATE_SOURCE_BACKUPS.md)。

## 收录与安全原则

- 收录设计文档、架构决策、模块契约、路线草案、协议草案与迁移清单。
- 不收录实现代码、密钥、令牌、凭据、构建产物、依赖缓存或任务看板。
- 私有来源必须获得仓库所有者明确的公开备份授权。
- 活跃缺陷清单以及详细披露鉴权、安全边界或运维命令的高敏感文档，不因一般“设计文档”授权自动公开。
- 每次备份前执行敏感信息扫描，并在分库来源记录中固定源提交。

## 更新约定

1. 在 `repositories/<source-repository>/` 下保留源路径。
2. 更新该分库的 `BACKUP_SOURCE.md` 与导航页。
3. 核对源许可证和 NOTICE；没有明确许可证时不得推定采用本仓库 MIT。
4. 检查相对链接、敏感信息和公开授权范围。
5. 使用可追溯提交写入 `main`。

## License

本仓库自行编写的索引与辅助文件采用 [MIT](LICENSE)。备份文档继续适用各自源仓库的许可证或权利状态；分库目录中的 `LICENSE`、`NOTICE` 或 `LICENSE_STATUS.md` 优先。

