# Verification record

What was checked for this guideline, how, and what is still unverified. Checked on 2026-11-10 with Rust 1.98.1 (Clippy 0.1.98, rustfmt 1.9.0, Cargo of the same release) on the stable channel.

## Summary

| Item | Result |
|---|---|
| Rules in GUIDELINES.md | 138, in 15 chapters, IDs unique and contiguous per chapter |
| Lint names cited in GUIDELINES.md | 137 verified: 117 Clippy, 18 rustc, 2 rustdoc; 0 unverified |
| Lint entries in `lints/Cargo.lints.toml` | 90 (5 default-on Clippy groups, 1 optional group, 84 lints); all accepted by Clippy 0.1.98 |
| Lint names from the research notes dropped | 7 (see "Dropped or corrected lint names") |
| Keys in `lints/clippy.toml` | 9 keys (4 active, 5 commented examples); all accepted by Clippy 0.1.98 |
| rustfmt options in `lints/rustfmt.toml` | 11 named; 4 stable and accepted, 7 unstable and labelled (one of them listed as stable in the docs, see below) |
| Links in GUIDELINES.md | 164 distinct pages, all returned HTTP 200 |

## How lint names were checked

1. **Clippy lint list.** Downloaded the lint list pages for `master` and `stable` (https://rust-lang.github.io/rust-clippy/master/index.html, which redirects to clippy.rust-lang.org, and the `stable` equivalent). Parsed every lint entry with its group and default level: 859 lints on `master`, 847 on `stable`. A script expanded each lint reference in the source of GUIDELINES.md and failed on any name that was missing from either list or whose group differed between them. No reference failed. The group printed after each lint in GUIDELINES.md comes from this data.
2. **rustc and rustdoc lints.** Checked against `rustc -W help` and `rustdoc -W help` from Rust 1.98.1, which also give the default levels printed in GUIDELINES.md. `unsafe_op_in_unsafe_fn` is listed as allow-by-default by `rustc -W help`; the Edition Guide states it warns by default in edition 2024 (https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html), and GUIDELINES.md says both.
3. **Real run.** Built a scratch workspace that used `lints/Cargo.lints.toml` with every strict-tier line uncommented, and `lints/clippy.toml` with every commented key uncommented. `cargo clippy --all-targets` (Clippy 0.1.98) reported no `unknown_lints` warning and no configuration error. As a control, adding a made-up lint produced `unknown lint` and a made-up `clippy.toml` key produced `unknown field`, so the run does detect bad names.

Clippy groups change between releases. Re-run step 3 when you upgrade the toolchain.

## Dropped or corrected lint names

From the research notes this guideline was built from:

| Name in the notes | Status | Action |
|---|---|---|
| `clippy::transmute_int_to_char`, `transmute_float_to_int`, `transmute_int_to_float` (implied) | Not in the Clippy list | Replaced by the rustc lint `unnecessary_transmutes` (warn by default) in UNS-07 |
| `clippy::drop_copy` | Not in the Clippy list | Not used (the notes already said it moved to rustc `dropping_copy_types`) |
| `clippy::empty_enum` | Not in the Clippy list; `empty_enums` (nursery) exists | Not used |
| `clippy::large_stack_frames` | In the `stable` list (nursery) but not in `master` | Not used |
| `clippy::integer_arithmetic` | Not in the Clippy list | `arithmetic_side_effects` (restriction) used instead |
| `clippy::cast_ref_to_mut` | Not in the Clippy list | Not used (rustc `invalid_reference_casting` covers it, deny by default) |
| `clippy::unchecked_duration_subtraction` | Not in the Clippy list; `unchecked_time_subtraction` (pedantic) exists | Not used |

Group corrections against the notes: `await_holding_lock` and `await_holding_refcell_ref` are `suspicious` (warn by default), not `pedantic`; `module_name_repetitions` is `restriction`, not `pedantic`; `too_long_first_doc_paragraph` is `nursery`; `format_push_string` is `pedantic`; `redundant_clone` is `nursery`.

Microsoft's lint list also contains taste lints that no rule here adopts (`semicolon_outside_block`, `if_then_some_else_none`, `deref_by_slicing`, `empty_drop`, `empty_enum_variants_with_brackets`, `empty_structs_with_brackets`, `redundant_type_annotations`, `unneeded_field_pattern`, `renamed_function_params`, `disallowed_script_idents`, `fn_to_numeric_cast_any`). All exist; they were left out to keep the default tier low-noise.

## How rustfmt and Clippy configuration options were checked

- **rustfmt.** Every option was checked against https://github.com/rust-lang/rustfmt/blob/main/Configurations.md for existence, default and the "Stable" field. Then all options were enabled in the scratch workspace and `cargo fmt --all -- --check` was run with rustfmt 1.9.0. Stable rustfmt accepted `style_edition`, `edition`, `use_field_init_shorthand` and `use_try_shorthand`, and printed "unstable features are only available in nightly channel" for the other seven.
- **Discrepancy found:** `hex_literal_case` is marked "Stable: Yes" in the main-branch Configurations.md, but rustfmt 1.9.0 (Rust 1.98) still rejects it on stable. `lints/rustfmt.toml` lists it as unstable. It may become usable in a later release.
- **Clippy configuration keys.** Every key in `lints/clippy.toml` is listed in https://doc.rust-lang.org/clippy/lint_configuration.html, with the lints it affects, and was accepted by Clippy 0.1.98 (see step 3 above). Clippy reads `msrv` from `rust-version` in `Cargo.toml` by default (same page).
- **Cargo.** The `[lints]` table and its `priority` field are documented in the Cargo manifest reference, which states MSRV 1.74 for `[lints]`. `cargo build --timings` is listed by `cargo build --help` on stable 1.98. The `cargo::` build-script syntax needs Cargo 1.77 (Cargo build-scripts reference).

## Source pages read or re-read in this pass

The research notes recorded where a page was only partly read. These pages were fetched again and read as raw text (no summarising tool):

- **Rust API Guidelines**: all 11 chapter pages (naming, interoperability, macros, documentation, predictability, flexibility, type-safety, dependability, debuggability, future-proofing, necessities). The notes had read only the checklist. Rule anchors were taken from the page IDs.
- **Rust Style Guide**: the non-formatting conventions page, which the notes reported as HTTP 404 (its URL is `advice.html`), plus `principles.html`, `items.html`, `expressions.html`, `statements.html`, `types.html` and `cargo.html`.
- **Microsoft Pragmatic Rust Guidelines**: the single-page version (`print.html`) with all guidelines, including the exact `M-STATIC-VERIFICATION` lint lists, `M-YIELD-POINTS` (10 to 100 µs), `M-PANIC-CONTINUATION`, `M-SMALLER-CRATES` and the AI chapter; also the "Agents & LLMs" page and the FAQ page (the notes had not read these). Rule anchors were taken from the checklist links.
- **Google Rust Style Guide**: the full page, including the licensing footer and the section bodies used for TYP-12, CON-07, ERR-10, MOD-03, MOD-04 and TYP-14. Section anchors were taken from the page IDs.
- **Rust Coding Guidelines**: the English `print.html`, used to confirm that every cited rule ID exists and to read its title, and to build the per-rule URLs. The appendix template pages (rustfmt.toml, clippy.toml, deny.toml) still contain only a heading.
- **Cargo**: the Build Performance guide, the timings, build scripts and manifest reference pages.
- **Clippy**: the lint groups page and the lint configuration page.
- **Others**: the nextest "Running tests" page (doctests are not run by nextest), and the Edition Guide page on `unsafe_op_in_unsafe_fn`.

## Still unverified

- **Rust Coding Guidelines rule bodies.** The research notes read about 30 rule pages in full (mostly in the Chinese edition). For the other cited RCG rules, only the title was checked in this pass. Rules that cite RCG may therefore rest on the rule title alone.
- **Compile-time sources not re-read in this pass.** The claims taken from the matklad posts, corrode, Feldera, Nicholas Nethercote's posts, Kobzol's post, the 2025 compiler performance survey, the Rust lld blog post, the rust-analyzer FAQ, the sccache documentation and the cargo-hakari documentation come from the research notes, which read and cited those pages. Their URLs load, but the statements were not re-checked word by word.
- **Inference in the appendix.** "Each git worktree should have its own target directory" generalises the rust-analyzer FAQ's description of lock contention and cache thrashing; no source tested worktrees directly.
- **CMP-15** (commit generated code instead of regenerating it in `build.rs`) is this document's position. No source measured it.
- **`cargo deny`** is named in DEP-07 as an option; none of the six sources names it (RCG and Microsoft name `cargo audit`).
- **Measured gains.** None of the compile-time numbers in the sources was reproduced here.

## Taken from memory

- That `#[expect]` is available on stable Rust from 1.81 was not checked and is not stated in the files; the files only require Cargo 1.74 for `[lints]`.
- The Rust idioms used in examples and rule text (`Arc::clone`, `cast_mut`/`cast_const`, `char::from_u32`, `let ... else`, `cargo tree` flags, `cargo hack --feature-powerset`, `cargo +nightly miri test`) were not looked up in this pass. The lint-run and doc checks above do not cover them.

## Wording and copying

Every rule is written in new words. The only text quoted from sources is short identifiers: rule IDs, lint names, option names and type names. Code examples were written for this guideline.

## Licence choice

**Recommendation: MIT** for the whole repository (the guideline text and the configuration files). `LICENSE` contains the MIT text with the holder "The Rust Guidelines contributors", plus a short third-party attribution note.

Reasons:

1. **The sources allow it.** Five of the six sources are MIT or MIT/Apache-2.0. Their conditions apply to copies or substantial portions of their text; this guideline reproduces none, and credits each source anyway.
2. **The CC BY 3.0 source.** The Google Rust Style Guide is CC BY 3.0 (its code samples are Apache 2.0). CC BY 3.0 requires, for copies and adaptations: the author's name, the title, the URL, a link to the licence, a statement of changes for adaptations, and no implied endorsement. This guideline restates ideas in new words, which is arguably not an adaptation, but README.md and LICENSE still give all of these items, so the obligation is met either way. CC BY does not require that a derived work use CC BY (it is not share-alike), so MIT remains possible.
3. **The configuration files are code.** Projects will paste `Cargo.lints.toml`, `clippy.toml` and `rustfmt.toml` into their repositories. MIT is the common licence in the Rust ecosystem and adds no attribution step for small configuration snippets in practice.

Alternative, for a documentation licence: CC BY 4.0 for the Markdown files and MIT for the files under `lints/`. This is a judgement call, not legal advice.

## Privacy check

The files contain no local paths and no contact details. The only personal and company names are the authors and publishers of the cited public sources. The only repository URL is `https://github.com/zichuanxu/rust-guidelines` in AGENTS-SNIPPET.md.
