# Issue 37760 V1 patch verification

## Status

This patch is **unverified at runtime**. Nothing in the regression-test or production commits compiled or ran on the development machine. Do not treat the branch as proof that the issue is fixed.

The branch contains a regression-test commit followed by a production commit:

1. `51c514d917` adds request-level V1 spawn, idle-follow-up, and active-steer tests.
2. `62bdc250e1` adds the filtered metadata snapshot and propagation.

Changed-file formatting passed with the cached `rustfmt` from nightly `2026-10-01`, not the repository's pinned Rust 1.95 toolchain. `git diff --check` passed. An independent source review found no blocking defect after the final adjustments. These checks do not establish compilation or runtime behavior.

## Test blocker

Cargo cannot resolve the repository's pinned Git dependencies in offline mode. Each focused test exits 101 before compilation. The logs identify missing `crossterm` or `tungstenite-rs` checkouts. Public Git dependency downloads were not used.

Run these commands from `codex-rs` in an environment with the exact locked dependencies:

```sh
cargo test -p codex-core --test all spawned_child_receives_forked_parent_context
cargo test -p codex-core --test all steering_active_v1_child_replaces_inherited_client_metadata
cargo test -p codex-core --lib inherited_client_metadata_scopes_review_overrides_to_their_turn
```

Run the first two against `51c514d917` to observe the missing child metadata, then run all three against the production commit. Clippy and warning-as-error checks also remain necessary.

The active-steer test uses a five-second response delay. A slow environment can miss the steering window and fail the turn-ID assertion even when the behavior is correct. Steered-input follow-up behavior, late child-event delivery, and the non-Guardian `parent_response_id` assertion remain unverified.

## Scope and unresolved decisions

This branch covers only V1 spawn, idle follow-up, and active steer. It does not fix V2 mailboxes, delegate, review, or Guardian child-entry paths. It does not close issue #37760.

No maintainer has ruled that dynamic client metadata should cross subagent boundaries. The implementation follows the issue's requested behavior. Excluding parent `misalignment_override` and `parent_response_id` is an inference from their turn-scoped and child-specific roles, not a maintainer ruling.

The project rejects external code contributions and pull requests. This branch exists only in the user's fork. No upstream PR was created.

## Behavior changes and tradeoffs

- Parent-originated V1 sends replace the child's dynamic client metadata. An empty parent snapshot clears stale values on active steer.
- Full replacement clears a child's legacy timestamp `misalignment_override` and any metadata set directly on that child. A child's targeted `review_target` override still survives. Clearing legacy authorization fails closed, but it can require another review acknowledgment.
- Dynamic `source` and `workspace_kind` values now follow the triggering parent turn, including their analytics consumers.
- The snapshot uses the raw dynamic map. A key shadowed by the parent's configured metadata can surface when the child's own configuration does not shadow it.
- The new crate-private `SpawnAgentOptions` field prevents external Rust struct-literal construction of that type. In-tree callers are updated, but this is a Rust API change. The wire protocol is unchanged.
