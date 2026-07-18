# Spec — `std.fmt` (formatting / display as a dual human + machine projection)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-fmt` (M-533, #173, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.fmt` · Ring `2` (RFC-0016 §4.2) · Tier `B` |
| **Tracks** | `M-533` (#173) — the Phase-5 task this spec delivers |
| **Scope** | Formatting and display: rendering a `Value` into a **human** view and a **machine** (JSON) view over one canonical form. Owns the display/debug/`to-json`/`from-json` surface and the marked-truncation discipline. |
| **Boundary** | Out of scope: the wire-level (de)serialization codec and the binary/base-N encodings — those are `serialize`/`encoding` (M-514); `fmt` is the *display* projection, not the canonical transport. UTF-8 string slicing / `parse` is `text` (M-524). A representation change is `swap` (M-516), never a format. Structured diagnostic rendering is `diag` (M-510); `fmt` provides the projection mechanism `diag` reuses, it does not own diagnostics. |
| **Depends on** | RFC-0016 §4.1 (the contract), §4.4 (`fmt` row), §4.5 (the matrix); RFC-0013 §4.3 (the dual human/JSON projection precedent, I3 round-trip); RFC-0001 (the value model — `Value`/`Repr`/`Meta`, content-addressing §4.6); ADR-003 (content-addressed identity; metadata is not identity). |
| **Grounds on** | `mycelium-core` (the `Value`/`Repr`/`Meta` types and the content-id, RFC-0001 §4.6); the M-380 projection framework RFC-0013 names. KC-3: `fmt` adds no trusted code — it is a projection **consumer**. |

---

## 1. Summary

`std.fmt` renders a `Value` into two views of **one canonical form**: a **human** projection (readable text) and a **machine** projection (JSON), exactly as RFC-0013 §4.3 renders one diagnostic as "two renderers of one truth" (G11). The user-facing surface is `display`/`debug` (human), `to_json`/`from_json` (machine), and a `truncate`-aware display for bounded output. Its **honesty crux** is twofold and structural: (1) a display is a **projection, not identity** — formatting a value never changes its content hash (ADR-003); and (2) a truncated or elided rendering **says so** — it is explicitly marked, never a silent drop (C1/G2). It is a Ring-2 module that adds **no trusted code** (KC-3): it consumes `mycelium-core`'s value model and content-id and renders over them.

## 2. Scope & module boundary

- **In scope:** the human display projection (`display`, `debug`); the machine projection (`to_json`, `from_json`); the marked-truncation/elision discipline (a bounded display that records what it dropped); and the round-trip property binding the machine projection to the canonical value.
- **Out of scope (and who owns it):**
  - canonical (de)serialization codec + binary/base-N encodings → `serialize`/`encoding` (**M-514**); `fmt`'s JSON view is a *display* projection that round-trips, not the canonical transport format.
  - UTF-8 string operations, slicing, and `parse` → `text`/`string` (**M-524**).
  - a representation change (binary↔ternary↔dense↔VSA) → `swap` (**M-516**) — a format is never a swap.
  - structured-diagnostic rendering and error-policy → `diag` (**M-510**); `fmt` supplies the projection mechanism, it does not own the diagnostic record.
- **Ring & layering:** Ring 2 (general library, RFC-0016 §4.2). `fmt` is written **to the §4.1 contract over Ring 0** (the `Value`/`Repr`/`Meta` re-exports). It *consumes* the content-id from `mycelium-core` and never enlarges the trusted base (KC-3); it builds new projection ops rather than re-exporting.

## 3. Exported-op surface (design sketch)

A signature sketch — value-semantic and immutable-by-default. The display ops take a `&Value` (a borrow, never a mutation) and yield a rendering; the round-tripping machine op pair is `to_json`/`from_json`. Truncation is a *parameter*, and its outcome is a typed value that records whether and what it elided. This is a DESIGN sketch to fix the surface and feed the matrix, not a committed grammar.

```
// illustrative signatures (not a committed surface)

// Human projection — total, pure, EXPLAIN-irrelevant (no selection/approx).
display(v: &Value) -> Text                 // readable, full
debug(v: &Value)   -> Text                 // structural, full

// Machine projection — JSON view over the one canonical form (G11).
to_json(v: &Value)   -> Json               // total: every Value has a JSON view
from_json(j: &Json)  -> Result<Value, FromJsonError>   // fallible: malformed/unknown -> Err

// Bounded display — truncation is a typed, MARKED outcome, never a silent drop.
display_bounded(v: &Value, limit: Budget) -> Rendering
//   Rendering { text: Text, truncation: Truncation }
//   Truncation = Complete | Elided { omitted: Count, marker: Text }
//                                  // an Elided rendering CARRIES the marker; it cannot be Complete-shaped

// Errors the surface can raise (the explicit set; never a sentinel).
enum FromJsonError { Malformed(span), UnknownTag(name), OutOfDomain(field) }
```

`Truncation::Elided` is the never-silent guard made type-level: a bounded display that drops data **cannot** be constructed without the `marker`/`omitted` fields, so a silent truncation is unrepresentable rather than merely discouraged (C1/G2).

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops. Encoded as a checked table (the RFC-0003 §4 template), asserted in tests once code lands — never prose only. The machine-projection **round-trip** (`from_json ∘ to_json = id` on the canonical value, sharing its content-id) is the one *checked property* of this module (G11; the RFC-0013 §4.3 / I3 precedent).

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|
| `display` | `Exact` | total | `none` | n/a (a faithful full render; no selection/approx) |
| `debug` | `Exact` | total | `none` | n/a |
| `to_json` | `Exact` | total | `none` | n/a (the machine view of one canonical form) |
| `from_json` (parse-back) | `Exact` | `Err(Malformed \| UnknownTag \| OutOfDomain)` | `none` | n/a (round-trip is the checked property, not a heuristic) |
| `display_bounded` (truncated display) | `Exact` | total — but yields `Truncation::Elided{omitted, marker}` when bounded; never a silent drop | `alloc(budget)` (the `limit`/`Budget`) | **yes** — the `Truncation` record is the reified, inspectable artifact of *what* was elided and *why* (C3) |

**Tag justification.** Every row is `Exact` and none is `Proven`/`Empirical`/`Declared`: `fmt` carries **no** accuracy/precision/probability semantics (C2 — an op with no accuracy semantics is simply `Exact`, like `len`). `fmt`'s `from_json` `Exact` rests on **decode determinism** — the same JSON text always decodes to the same `Value`, a structural property with no numeric bound — and on malformed input it is an explicit `Err` (C1), never a best-effort coercion. **Scope-distinct framing (DN-16, 2026-06-19):** the related **round-trip fidelity** claim (`from_json(to_json(v))` recovers the same content-id, ADR-003 / RFC-0001 §4.6) is owned by `std.io` (the canonical codec `fmt` delegates to, M-372), where it is tagged **`Empirical`** (proptest corpus, no theorem; VR-5). `fmt`-`Exact` (decode determinism) and `io`-`Empirical` (round-trip fidelity) name *different* properties of the same call and are deliberately both kept — neither over-claims `Proven` (see io.md §7-Q1 + the crate guarantee matrices). `display_bounded` stays `Exact` because its result is faithful *to what it claims to render*: an elided rendering does not assert completeness — it carries the `Elided{omitted, marker}` evidence (so it is not a downgrade to `Declared`, it is an honest partial view whose partiality is in the type). No op is `Proven`, so no theorem side-conditions are claimed; downgrading rather than overclaiming (VR-5) is moot because nothing here reaches above `Exact`.

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2):** `from_json` returns `Result` with an explicit error set (`Malformed`/`UnknownTag`/`OutOfDomain`), never a sentinel or partial value. A bounded display that drops data returns `Truncation::Elided{omitted, marker}` — the truncation **says so** and is unrepresentable as a `Complete` rendering, so silent data loss is a type error, not a runtime hope.
- **C2 — honest per-op tag (VR-5):** every op is `Exact` because `fmt` has no accuracy/precision semantics. The one honest property that *is* checked — the machine-projection round-trip to the same content-id — is recorded as `Exact` in §4 and asserted as a property test once code lands, not asserted in prose.
- **C3 — no black boxes / EXPLAIN (SC-3/G11):** the only op that *elides* a user-visible outcome (`display_bounded`) reifies *what* it elided in an inspectable `Truncation` record — the EXPLAIN-able artifact for that decision. The other ops neither select, convert, nor approximate, so they carry no hidden heuristic to explain (EXPLAIN is `n/a`, not absent-and-needed).
- **C4 — content-addressed, value-semantic (ADR-003):** **formatting is a projection, not identity.** A display/`to_json`/`debug`/bounded render is a pure function of a borrowed `&Value`; it **never** mutates the value and **never** changes its content hash (ADR-003 — metadata and rendering are not identity). The human and machine views are two projections of *one* content-addressed canonical form (G11); neither is "new truth" (RFC-0013 §4.3). This is the load-bearing §-level statement: *formatting a value leaves its content-id unchanged.*
- **C5 — above the kernel (KC-3):** `fmt` consumes `mycelium-core`'s value model and content-id and adds **no** trusted code; it is a projection consumer in the RFC-0013 §4.3 sense. No `wild`/FFI (ADR-014) — pure rendering needs no OS facility.
- **C6 — declared, bounded effects (RFC-0014):** the pure renderers (`display`/`debug`/`to_json`/`from_json`) declare `none`. `display_bounded` declares its bound on the signature as a `Budget`/`limit` (an `alloc(budget)` effect) — the output size is explicitly capped, and the cap's consequence (elision) is surfaced, not hidden (the EffectBudget discipline).

## 6. Grounding

- The **dual human + machine (JSON) projection over one canonical form** and its **round-trip** are RFC-0013 §4.3 / invariant I3 ("two renderers of one truth", "round-trip to the same content-addressed identity"), the G11 precedent RFC-0016 §4.4 names for `fmt`.
- **Display is a projection, not identity; formatting never changes the content hash** is ADR-003 (content-addressed identity; metadata is not identity) and RFC-0001 §4.6 (content-addressing), via §4.1 clause **C4**.
- **Never-silent / marked truncation** is G2 and §4.1 clause **C1** (no silent loss; an off-grid/over-budget case is explicit), the same posture RFC-0013 takes for elide-never-hide projection.
- **The per-op tags and the round-trip as a checked table** are RFC-0016 §4.5 (the guarantee matrix) over the RFC-0003 §4 matrix template; the honest-tag rule is **VR-5** / clause **C2**.
- **Ring 2 / above the kernel** is RFC-0016 §4.2 and clause **C5** (KC-3).
- The module's place in the taxonomy is RFC-0016 §4.4 (the `fmt` row) and the index [`README.md`](./README.md); the structural template is [`_TEMPLATE.md`](./_TEMPLATE.md).

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) — RESOLVED (2026-06-19, wired).** Is `fmt`'s JSON view the *same* projection mechanism as `serialize`'s, or a sibling? **Resolution:** `fmt.to_json`/`from_json` **delegate** to `mycelium-std-io::to_json`/`from_json` — the one canonical JSON projection (spec §7-Q1 / `README.md §5`). The delegation is wired in `crates/mycelium-std-fmt` (M-372): `io` owns the codec, `fmt` keeps its thin display facade (`Json`/`ToJsonError`/`FromJsonError`). The round-trip property is established once, in `std.io`, and re-checked here. **Tag-framing residual (honesty, VR-5 — for the maintainer):** `std.io` tags `from_json` `Empirical` (proptest corpus); `std.fmt` tags it `Exact` (deterministic decode, no accuracy semantics). Both are honest from their angle; unifying the framing is a finer reconciliation deferred to the maintainer; the delegation does not silently change either tag.
- **(Q2) Does the human projection round-trip, or only the machine one?** RFC-0013 §4.3 / I3 makes *both* human and JSON projections recover the same diagnostic. For arbitrary `Value`s a human display may be intentionally lossy (it is for reading, not transport). This spec commits the **machine** projection to round-trip (§4) and treats human round-trip as out of scope unless the maintainer wants the stronger RFC-0013 bar. Disposition: FLAGGED; proposed default is machine-only round-trip, with human display free to elide (marked).
- **(Q3) The truncation marker's form and stability.** Is the `Elided{marker}` a fixed sentinel string, a structured `Truncation` value the machine view also carries, or both? It must be unambiguous (un-confusable with real content) and, for the bounded *machine* view, itself round-trippable. Disposition: FLAGGED; the structured-value form is proposed so the marker is data, not a parseable-by-accident string. Ties to RFC-0016 §8-Q3.

## Meta — changelog

- **2026-06-19 — Q1 resolved/wired (M-372, delegation ratified).** The §7-Q1 open question ("is `fmt`'s JSON the same projection as `serialize`'s?") is **resolved — delegation wired**: `fmt.to_json`/`from_json` now delegate to `mycelium_std_io::{to_json, from_json}` (one canonical JSON, two entry points; M-372). The round-trip property is established once in `std.io` (tagged `Empirical`; proptest corpus) and re-checked in `fmt`. The public API and guarantee tags of `fmt` are **unchanged** (all-`Exact` matrix; `ToJsonError::NonFinite{index}` is still the typed refusal for non-finite payloads). **Tag-framing residual** noted: `std.io` tags `from_json` `Empirical`; `std.fmt` tags it `Exact` (deterministic decode, no accuracy claim) — both honest, framing reconciliation deferred to the maintainer (VR-5). No spec status change (spec stays "implemented, pending ratification"; the delegation wiring is the code change, not a ratification). Append-only.
- **2026-06-17 — Draft (needs-design).** Stands up `std.fmt` (Ring 2, Tier B; M-533, #173) as the **dual human + machine (JSON) projection over one canonical form** (G11; the RFC-0013 §4.3 / I3 precedent). Honesty crux: (1) display is a **projection, not identity** — formatting a borrowed `&Value` never mutates it and never changes its content hash (ADR-003 / clause C4); (2) a truncated/elided display **says so** via a typed `Truncation::Elided{omitted, marker}` record, never a silent drop (C1/G2), with the elision reified as the EXPLAIN-able artifact (C3). Guarantee matrix: five rows (`display`/`debug`/`to_json`/`from_json`/`display_bounded`), all `Exact` (no accuracy semantics), with the **machine-projection round-trip to the same content-id** as the one checked property (RFC-0016 §4.5; the RFC-0003 §4 matrix template); `from_json` carries the explicit `Malformed`/`UnknownTag`/`OutOfDomain` error set. §4.1 conformance C1–C6 stated concretely; boundary fixed against `serialize` (M-514), `text` (M-524), `swap` (M-516), `diag` (M-510). Three §8-tied open questions FLAGGED (fmt/serialize JSON overlap → §8-Q1; human vs machine round-trip scope; truncation-marker form → §8-Q3). No code; no kernel change (KC-3). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
