# intel-actor

**The pure `.cljc` actor boundary for the Intel actor (`did:web:intel.etzhayyim.com`) —
a deny-by-default planner that turns a cell invocation into a set of
`:mst/put-record` effects, and refuses to emit any effect until every required
attestation gate is present.**

This repository is **not** the Intel platform. It holds the boundary only:
thirteen tracked files, one namespace, no network, no runtime, no host.

## What is actually here

| | |
|---|---|
| Source | `src/intel/murakumo.cljk` — one namespace, `intel.murakumo` |
| Tests | `test/intel/murakumo_test.cljk` — 9 tests / 213 assertions |
| Declarations | `actor-manifest.jsonld`, `.well-known/did.json`, `storage-profile.edn` |
| Prose | `CLAUDE.md` — describes the **deployed platform**, which does not live here |

`intel.murakumo` is one data table and eight functions:

- **`cell-specs`** — 15 cells, each declaring `:legacy-cell`, `:phase`,
  `:murakumo-node`, `:collections` and `:required-gates`.
- **`gate-value` / `missing-gates`** — attestation lookup that accepts either a
  map or a set, keyed by keyword or by string.
- **`records-for` / `put-record-effect`** — one record per declared collection,
  stamped with `:actorDid`, `:legacyCell`, `:scaffold true`.
- **`cell-plan` / `all-cell-plans`** — the boundary itself.
- **`collection` / `safe-rkey`** — name derivation. `collection` is the subject
  of §"Three planes" below; `safe-rkey` sanitises a record key to
  `[A-Za-z0-9._~-]` and falls back to `"unknown"` rather than emitting a blank.

### The gate is the point

`cell-plan` is deny-by-default. With no attestations it returns `:blocked` and
**an empty effect vector** — it does not emit a single record:

```
no attestations    -> :blocked  effects 0  missing 7
all gates attested -> :ready    effects 1
```

All 15 cells require the same 7 baseline gates (`common-gates`): council
charter attestation, no platform-held key, no probing, Murakumo-only inference,
DID-primary, append-only, Kotoba-only substrate. An unknown cell key throws
rather than planning nothing quietly.

Reproduce both directions in §2 of the quickstart — it takes about a second and
needs no JVM.

## Three planes describe this actor, and they do not agree

Measured 2026-09-03 at `4d2851e`. **Nothing here has been reconciled**: which
plane is canonical is an owner decision, not something a docs change may settle.
See `docs/adr/0001-three-planes-disagree.edn` for the decision to record rather
than reconcile, and for the commands that re-measure each row.

| Question | `src/…/murakumo.cljc` | `actor-manifest.jsonld` | `.well-known/did.json` | `CLAUDE.md` |
|---|---|---|---|---|
| actor DID | `did:web:intel.etzhayyim.com` | same | **`did:web:etzhayyim.com:actor:intel`** | — |
| runtime | (none) | `k8s-langserver` | — | **`Worker WASM`** |
| collections | `com.etzhayyim.intel.*` (15) | `com.etzhayyim.apps.*` (10) | — | — |

**The collection-name sets intersect in zero places.** Every name the planner
emits is `com.etzhayyim.intel.<lowercased-cell-key>`; every name the manifest
declares is `com.etzhayyim.apps.<app>.<camelCase>`. The correct name is present
in the data — `:legacy-cell` preserves it — but `collection` re-derives a
different one from the cell key instead of reading it.

So a `:ready` plan from this repo currently targets collections that no declared
trigger subscribes to. That is a real finding, not a lint nit, and it is why
this README states it instead of describing the actor as wired.

## Running it

See **[`docs/operator-quickstart.md`](docs/operator-quickstart.md)**. Every step
there was executed on 2026-09-03 and carries its measured result and duration.

## Provenance

Apache-2.0 with the etzhayyim Charter Compliance Rider v3.1 (`NOTICE`).
Storage profile `:kotoba/local-agent-kagi-chunks-v1` (`storage-profile.edn`):
local query, append-only transactions, kagi-chunked EDN payloads, kotobase head.
