# Duku fork of YSwift

This is a fork of [y-crdt/yswift](https://github.com/y-crdt/yswift) used by the Duku iOS app.
Upstream has had no commits since July 2024 and pins `yrs` 0.18, whose V1 decoder truncates
client ids to 32 bits. Peers on `yrs` >= 0.26 (including Duku's collaboration service, `yrs` 0.28)
generate 53-bit ids, so a stock YSwift client silently forks the document. Our branch is
[upstream PR #55](https://github.com/y-crdt/yswift/pull/55) (`yrs` 0.27.4, 53-bit ids,
`diff(from: [])` fix, subscription leak fix, undo no longer deadlocks), unmodified apart from the
manifest change below.

## Policy

- **Keep this fork generic.** Duku-specific code (wire envelope, presence, editor binding) lives in
  `duku-ios`, never here. Only `yrs` bumps and bug fixes belong in this fork.
- Consumers pin an exact tag (`exact: "0.3.0-duku.1"`), never a branch.
- Stay within one `yrs` minor of the collaboration service (`duku-collaboration-svc/Cargo.toml`).
  The wire format is stable, but verify with the interop test whenever either side moves.

## The one deliberate manifest change

`Package.swift` falls back to a **released** XCFramework when `lib/yniffiFFI.xcframework` is absent
(the case for every remote consumer). That URL must be this fork's release, never upstream's.

## Cutting a release

1. `./scripts/build-xcframework.sh` (needs rustup; `rust-toolchain.toml` pins the compiler and
   the five Apple targets). It writes `lib/yniffiFFI.xcframework.zip` and prints its SHA-256.
   Revert any formatting-only churn it causes in `lib/swift/scaffold/yniffi.swift`.
2. `swift package compute-checksum lib/yniffiFFI.xcframework.zip` must equal that SHA-256.
3. Put the new tag in the URL and the checksum in `Package.swift`; commit.
4. Push, then `gh release create <tag> lib/yniffiFFI.xcframework.zip --target <sha>`.
   The framework and zip are gitignored on purpose; the release asset is the artifact.
5. `swift test` in this repo (57 tests), then bump the tag in `duku-ios`.

## Releases

| Tag | Source | yrs | Notes |
| --- | --- | --- | --- |
| `0.3.0-duku.1` | upstream PR #55 @ `2e84937`, rustc 1.98.0 | 0.27.4 | First Duku release. SHA-256 `e17ce5b537f6e5968f2bc12280ae2779df5e20ad52cd475281d7b1dbc88716fb` |

## Verified behavior (2026-11-10)

- 1,200/1,200 random concurrent-edit runs (ASCII + non-ASCII) converge against `yrs` 0.28, including
  the server's 53-bit client ids (stock YSwift 0.2.1: 300/300 diverged).
- Cross-process round trip: server-seeded snapshot (53-bit id) -> Swift edits -> server merge with a
  concurrent edit -> Swift converges.
- Text indexes in the Swift API are **UTF-16** (matches `NSString`/UIKit), despite the wording in
  some upstream doc comments.
