# Design Drafts Backup

用于集中备份现有及未来项目的设计草案。内容按源仓库分目录保存，避免不同项目的架构文档、ADR 与未来方案相互混杂。

## 目录约定

```text
repositories/
└── <source-repository>/
    ├── README.md
    ├── docs/
    ├── adr/
    ├── Knowledge/modular-design/
    └── future_plan/
```

每个源仓库目录都应包含 `BACKUP_SOURCE.md`，记录源仓库、源分支、源提交和备份日期。后续新增项目时，在 `repositories/` 下创建同名目录并保留源文件的相对路径。

## 当前备份

- [`decentralized-chat-architecture`](repositories/decentralized-chat-architecture/README.md)：去中心化聊天软件的架构、模块契约、ADR 与未来匿名覆盖网络方案。

## 收录原则

- 收录设计文档、架构决策、模块契约、路线草案与未来方案。
- 不收录密钥、凭据、构建产物、依赖缓存或与设计备份无关的实现代码。
- 私有仓库内容只有在明确确认可公开后才可加入。

## License

MIT，详见 [LICENSE](LICENSE)。各备份目录若另有许可说明，以其目录内说明为准。

