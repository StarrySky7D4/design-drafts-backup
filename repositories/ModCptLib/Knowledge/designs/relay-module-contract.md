# 未来 Relay 模块契约

未来 relay 位于独立 crate 或仓库，依赖方向只能是 `relay -> core`。core 不依赖 relay。M7 已选择的 Mailbox 是独立的 opaque Envelope store-and-forward 产品，使用 `mailbox -> core`；它不授权或实现 relay、NAT traversal、CID 转发或自动 fallback。

保留线编号 `0x50/0x51`、`0x90-0x93`、`0xA2`，以及 `CertPolicy` 的三/七字节历史布局；这些保留位不是授权。未来模块必须拥有独立运行时 ACL 和迁移策略。

数据面只能转发密文。旧 HMAC token、CID 前缀、debug tag 和 Rust relay API 均未冻结，不得以兼容目标复刻。

至少重新设计身份绑定 ACL、防重放与 TTL 上限、带宽/并发限制、取消与优雅停机、候选地址与失败语义、端到端加密，以及真实跨 NAT 测试。
