# Rust Guidelines

A short, general-purpose Rust coding guideline for AI coding agents and the people who review their work. It distils six public sources into 138 checkable rules with stable IDs, a pre-finish checklist, and ready-to-use lint and formatter configuration.

## Who it is for

- **AI coding agents** that write or change Rust code. The rules are imperative and checkable, and each one names its enforcement (a lint, a rustfmt option, a tool, or review).
- **Reviewers** of agent-written code. Cite a rule ID instead of re-explaining a convention.
- **Project maintainers** who link this guideline from their `AGENTS.md` (see [AGENTS-SNIPPET.md](AGENTS-SNIPPET.md)).

It is not a Rust tutorial, and it does not repeat what rustfmt does automatically.

## Files

| File | Use |
|---|---|
| [GUIDELINES.md](GUIDELINES.md) | The rules, in 15 chapters, plus the priority order, the list of source disagreements, and an appendix on build-machine setup. |
| [CHECKLIST.md](CHECKLIST.md) | One page to run before you finish a Rust change. Each line points to rule IDs. |
| [lints/Cargo.lints.toml](lints/Cargo.lints.toml) | `[workspace.lints.rust]` and `[workspace.lints.clippy]` tables: an active default tier and a commented strict tier. Each lint names its rule ID. |
| [lints/clippy.toml](lints/clippy.toml) | Clippy configuration (tests may unwrap, example deny-lists). |
| [lints/rustfmt.toml](lints/rustfmt.toml) | rustfmt configuration: stable options active, unstable ones commented and labelled. |
| [AGENTS-SNIPPET.md](AGENTS-SNIPPET.md) | A block to paste into a project's `AGENTS.md`. |
| [VERIFICATION.md](VERIFICATION.md) | What was checked and how, what is still unverified, and the licence choice. |

## How an agent should use it

Read in this order:

1. **Before the first change in a session**: the "Priority when rules conflict" section of [GUIDELINES.md](GUIDELINES.md).
2. **While writing code**: the chapters that match the change (for example ERR and CON for an async handler, CMP when you add a crate or a dependency).
3. **Before you finish**: run every line of [CHECKLIST.md](CHECKLIST.md) that applies, and fix what fails.

Project rules come first. If a project's own `AGENTS.md`, lint configuration or style document contradicts a rule here, follow the project and mention the conflict in your summary.

Do not change lint levels or formatter settings to make a change pass (FMT-05). Suppress a single finding with `#[expect(lint, reason = "...")]`.

## How to cite rules

Use the rule ID, the location and one line of explanation. Examples:

```text
ERR-03 src/config.rs:42: `unwrap()` on user-supplied input; return the parse error instead.
CON-01 src/cache.rs:88: `MutexGuard` held across `.await`; drop it in an inner block.
CMP-12 tests/: five integration-test files compile to five binaries; merge into tests/it/.
```

In a summary of your own work, list the rules you deliberately did not follow, with the reason, for example: `NUM-01: kept the lossy cast in hash.rs; truncation is intended (comment added)`.

Rule IDs are stable. A retired rule keeps its ID unused; a new rule gets the next free number in its chapter.

## Adopting the lint configuration

1. Copy the two tables from [lints/Cargo.lints.toml](lints/Cargo.lints.toml) into the workspace root `Cargo.toml`.
2. Add `[lints] workspace = true` to every member crate.
3. Copy [lints/clippy.toml](lints/clippy.toml) and [lints/rustfmt.toml](lints/rustfmt.toml) to the workspace root, and set `style_edition` to your edition.
4. Run `cargo clippy --workspace --all-targets` and fix or `#[expect]` the findings. Enable strict-tier lints one at a time.

The configuration was checked with Rust 1.98 (Clippy 0.1.98, rustfmt 1.9.0). `[lints]` needs Cargo 1.74 or later.

## Sources and licences

Every rule is written in our own words. Only short rule identifiers (such as `C-CONV`, `M-PANIC-ON-BUG`, `G.ERR.01`), lint names and option names are quoted. Each rule links to the exact section it was distilled from.

| Source | Publisher | Licence (as stated by the source) | Link |
|---|---|---|---|
| Google Rust Style Guide | Google | CC BY 3.0 for the guide text; Apache 2.0 for its code samples | https://google.github.io/styleguide/rustguide.html |
| Rust Coding Guidelines (V1.0 beta), English and Chinese editions | Rust Coding Guidelines project (unofficial) | MIT, "Copyright (c) 2021 Rust Coding Guidelines (Unofficial)" | https://rust-coding-guidelines.github.io/rust-coding-guidelines/ |
| Rust API Guidelines | Rust library team | MIT or Apache 2.0, at your option | https://rust-lang.github.io/api-guidelines/ |
| Rust Style Guide | The Rust Project (style team) | MIT or Apache 2.0 (part of the `rust-lang/rust` repository) | https://doc.rust-lang.org/style-guide/ |
| Pragmatic Rust Guidelines | Microsoft | MIT, "Copyright (c) Microsoft Corporation" | https://microsoft.github.io/rust-guidelines/ |
| Clippy lint list and documentation | The Rust Project | MIT or Apache 2.0 | https://rust-lang.github.io/rust-clippy/master/index.html |

Attribution for the CC BY 3.0 source: this guideline draws on the *Google Rust Style Guide* by Google, https://google.github.io/styleguide/rustguide.html, licensed under CC BY 3.0 (https://creativecommons.org/licenses/by/3.0/). Its rules were summarised, merged with other sources and rewritten; no text was copied, and Google does not endorse this document.

The MIT and Apache 2.0 sources are credited above. No substantial portion of their text is reproduced.

The compile-time chapter and its appendix also cite (by link only, facts and numbers attributed in place): the Cargo book (MIT or Apache 2.0), the Rust blog and the 2025 Rust compiler performance survey, the nextest documentation, the rust-analyzer FAQ, the sccache documentation, and blog posts by matklad, corrode, Nicholas Nethercote, Kobzol and Feldera. No text from them is reproduced.

## Exclusion

The Safety-Critical Rust Coding Guidelines (Safety-Critical Rust Consortium) were deliberately left out. They target certified safety-critical software and were still a draft; this guideline is for general-purpose code.

## Licence

This guideline is released under the MIT licence; see [LICENSE](LICENSE). The reasons for that choice are in [VERIFICATION.md](VERIFICATION.md#licence-choice).
