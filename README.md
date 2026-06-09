# isoterm first-party capability catalog (Spec 082)

This directory is the human-authored, reviewable **source of truth** for the
isoterm first-party capability catalog (FR-002). It is NOT the signed manifest:
the dev-time `isoterm-catalog-tools` CLI aggregates these descriptors into the
byte-exact `manifest.json` and signs it (`manifest.json.sig`) that the shipping
app verifies and consumes (Spec 081 client).

```
catalog/
  items/        one TOML descriptor per catalog item (filename stem == id)
  artifacts/    real, committed artifact files referenced by descriptors
  README.md     this file
```

## Descriptor format

Each `items/<id>.toml` is one [`ItemDescriptor`] (mirrors
`specs/082-catalog-provisioning/contracts/descriptor.schema.json` 1:1). The
filename stem MUST equal the `id`. Required fields: `id`, `category`, `name`,
`description`, `install_kind`, and at least one `[[versions]]` block with a
`version`, a `core_compat` semver range, and per-`(os, arch)`
`[[versions.artifacts]]`. `ui_asset` items additionally require `ui_asset_kind`
(`theme`/`font`). Unknown keys are a hard parse error (`deny_unknown_fields`),
so a typo can never silently produce a subtly-wrong signed manifest.

## Building + signing the signed manifest

```bash
# 1. Validate every descriptor (CI gate; FR-012/013, exits non-zero on any failure).
cargo run -q -p isoterm-catalog-tools -- validate --descriptors catalog/items

# 2. Aggregate descriptors → byte-exact manifest.json.
#    For every artifact backed by a committed file under --artifacts (resolved by
#    URL basename or <sha256> name), the tool RECOMPUTES sha256 and REFUSES to
#    write any output on a mismatch (FR-005). Remote-only artifacts are trusted
#    as-authored (URL reachability is `validate --check-urls`'s job).
cargo run -q -p isoterm-catalog-tools -- build \
  --descriptors catalog/items --artifacts catalog/artifacts --out manifest.json

# 3. Sign the exact manifest bytes with the PRIVATE catalog key (out-of-band
#    custody — never committed; see docs/082-catalog-provisioning-operations.md).
cargo run -q -p isoterm-catalog-tools -- sign --manifest manifest.json --key /secure/catalog.key
```

Re-running `build`+`sign` over identical inputs yields byte-identical
`manifest.json` and `manifest.json.sig` (SC-001) — set `SOURCE_DATE_EPOCH` to
pin the `generated_at` timestamp for bit-reproducibility.

## What is authored here today (v1 scope)

**Authored, with real committed artifacts + real sha256:**

- `theme.nord` → `artifacts/nord.json`
- `theme.gruvbox-dark` → `artifacts/gruvbox-dark.json`

Both are MIT-licensed Gogh-format theme packs (plain JSON, no native code, so
`os`/`arch` are `any` — they install identically on every host, FR-015). Their
`sha256` is the real digest of the committed bytes and is re-verified by `build`.

## What is intentionally NOT authored yet (out-of-scope-v1, not a stub)

The remaining optional components named by FR-002 — the **39 non-essential
tree-sitter grammars** (fetched on first open) and the **downloadable GGUF
embedder** — are deliberately NOT given descriptors here yet. Their descriptors
require a *real* artifact (a built `.wasm` grammar payload, a hosted GGUF weights
file) with a *real* sha256, and that artifact build-and-host pipeline is itself
**out-of-scope for Spec 082 v1** (the spec scopes real public artifact hosting
out). Authoring placeholder descriptors with invented sha256 for artifacts that
do not exist would be a stub — exactly what this project forbids — and `build`
would (correctly) have nothing to verify them against.

When that pipeline lands, those descriptors are **generated** (not hand-typed):
the grammar ids come from the canonical isoterm-core grammar catalog
(`crates/isoterm-core/src/syntax/grammars/all.rs` — e.g. `grammar.go`,
`grammar.ruby`, …, one per `GrammarSpec.name`), and the embedder id from the
intelligence crate's embedder selector. Each generated descriptor's artifact
URL points at the real hosted build output and its sha256 is the real digest of
that build output, recomputed and gated by `build` exactly as the theme packs
are today. Until then, the grammars continue to ship via the existing
fetch-on-first-open path (Spec 081), independent of this catalog.

This split is intentional v1 scope: the catalog **pipeline** (author → validate
→ build → sign → verify) is fully real and proven end-to-end on the theme packs;
extending coverage to grammars/embedder is a data-only addition once their
artifacts are built and hosted, with no code change to the tooling.
