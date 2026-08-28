# Static Audit — Architecture Repository Remediation

- Repository: `StarrySky7D4/decentralized-chat-architecture`
- Pull request: #2
- Source: `agent/repository-audit-remediation`
- Target: `main`
- Reviewed base: `b08405e9b8cb2e742a8849c37392c30c4d7eeb7a`
- Reviewed code/document head: `a3f559469ecef39714a5fc3a6078961849e1ed72`
- Review date: 2026-08-15
- Review mode: static tree, source, and document review only; GitHub Actions were not invoked or used as a merge condition.

## Scope

The change adds the modular implementation contracts and isolated anonymous-overlay research plan, closes core threat-model and protocol-semantics gaps, corrects MVP/delivery claims, adds documentation validation source, and records the repository remediation.

The reviewed diff contains 50 files and is overwhelmingly documentation. The only executable project source is the PowerShell documentation validator; the included workflow definition was inspected as text but not run.

## Findings

1. **The Standard E2EE boundary is explicit.** The threat model treats contacts and every network/service role as untrusted, separates content protection from anonymity claims, defines A0–A9 adversaries, and makes recovery, metadata, resource, and release requirements fail-closed.
2. **Event sequencing is implementation-ready.** `actor_sequence` is scoped to `(ConversationId, DeviceId)`, reserved durably per operation, permits permanent gaps, does not drive nonce/key derivation, and defines replay, late arrival, revocation, rollback, and equivocation semantics.
3. **Control-fork recovery no longer depends on network order.** Recovery derives authority from the last unambiguous base snapshot, requires intersecting approval quorums, rejects stale branches permanently, forces rekeying, and enters terminal `CompromisedControlState` when conflicting decisions both satisfy quorum.
4. **Delivery guarantees are corrected.** Network delivery is at least once, while `EventId` uniqueness and local transactions provide exactly-once local fact effects. Service receipts do not impersonate device receipt, validation, decryption, or reading.
5. **MVP and solo-developer constraints align.** Attachment scope is placed in Phase 4.5, and module boundaries are logical contracts rather than a mandate to create 18 empty crates. Work-in-progress limits favor one vertical slice plus one direct unblocker.
6. **Anonymous routing remains isolated.** The overlay is marked future, exploratory, removable, and non-binding; strong-anonymity modes may not silently fall back to ordinary routes.
7. **PR #2 fully supersedes PR #1.** Tree comparison found no file present in PR #1 and absent from PR #2. Of the 53 blobs in PR #1's tree, 14 remain byte-identical and 39 are revised or integrated alongside the remediation.
8. **Documentation validation source is bounded.** The PowerShell script confines relative paths to the repository, rejects rooted/escaping links, enforces strict UTF-8 and a single final newline, and checks fences and trailing whitespace. The workflow uses read-only contents permission and pins checkout to a full commit SHA.
9. **Independent static document scan passed.** Direct tree/blob inspection of the reviewed head found 57 Markdown files and zero missing relative-link targets, unclosed fences, trailing-whitespace findings, or EOF-shape findings. This scan did not invoke GitHub Actions.

## Residual risks and follow-up

- These documents are Draft architecture and release gates, not evidence that the product or protocol is implemented, secure, interoperable, or audited.
- The separate `decentralized-chat` implementation repository now contains a preliminary local prototype. Its JSON/hex prototype encoding and local-only crypto boundary must not be described as conformance with the deterministic-CBOR, remote-admission, ratchet, recovery, and delivery contracts here.
- The architecture status and implementation traceability should be reconciled after each vertical slice; otherwise correct design documents can still drift from executable behavior.
- The validator checks file-level Markdown hygiene, not semantic consistency, anchor existence, reference freshness, or security correctness. Those remain review responsibilities.
- The repository-recorded validation result was not relied on; the static review performed its own connector-based tree/blob checks.

## Decision

**Approved for squash merge into `main`.** PR #2 should become the sole architecture baseline, and PR #1 should be closed as superseded after the merge. Approval covers documentation coherence and repository hygiene only, not product implementation or release security.

