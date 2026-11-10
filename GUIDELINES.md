# Rust Guidelines

General-purpose rules for writing and reviewing Rust code. Read [README.md](README.md) first for scope and usage.

## How to read a rule

Each rule has a stable ID (`ERR-03`), one imperative rule line, and three fields:

- **Why**: the reason, in one or two lines.
- **Check**: how to enforce it. A lint name is followed by its group or default level, verified against Clippy and rustc (see [VERIFICATION.md](VERIFICATION.md)). "Review" means no tool checks it.
- **From**: the sources the rule was distilled from. The wording here is our own.

Scope tags:

- **[lib]**: applies to library crates that other crates depend on (published crates and shared workspace crates). Binaries and crate-private code may relax these rules.
- No tag: applies to all Rust code.

Rule IDs are stable. A removed rule keeps its ID retired; new rules get new numbers.

## Priority when rules conflict

Apply the first item that decides the case:

1. **Soundness and correctness.** No undefined behaviour, no wrong results, no silently dropped errors.
2. **Clarity for the reader and reviewer.** The next person (or agent) must be able to verify the code.
3. **Measured runtime performance.** Only on a hot path with a benchmark that shows the gain.
4. **Compile time.** Prefer the cheaper-to-compile design when the runtime cost is unmeasured or negligible.
5. **Brevity and author convenience.**

Why this order: Microsoft admits a guideline only if it improves safety, cost or maintainability, and states that all code must be sound ([Microsoft M-UNSOUND](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-UNSOUND), [Microsoft: meta design principles](https://microsoft.github.io/rust-guidelines/)); Google's guide ranks readability above ease of writing ([Google: Goals](https://google.github.io/styleguide/rustguide.html#goals-of-the-style-guide)); the official style guide ranks readability first ([Style Guide: principles](https://doc.rust-lang.org/style-guide/principles.html)). Runtime speed outranks compile time only when measured, because Microsoft asks for benchmarks before optimising ([Microsoft M-HOTPATH](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-HOTPATH)) and Google asks for benchmarks before performance-motivated `unsafe` ([Google: Unsafe code](https://google.github.io/styleguide/rustguide.html#unsafe-code)). Compile time outranks brevity because the 2025 compiler performance survey reports slow rebuilds as the top complaint ([Rust survey 2025](https://blog.rust-lang.org/2025/09/10/rust-compiler-performance-survey-2025-results)).

## Where the sources disagree

| Topic | Positions | This document | Reason |
|---|---|---|---|
| Line break around binary operators | RCG `P.FMT.06`: operator at line start. Style Guide: break before operators, after `=`. | Run rustfmt with defaults (FMT-01). | rustfmt implements the Style Guide; a second convention cannot be enforced. |
| Import granularity | Google: one module per `use`. Style Guide: tools must not merge or split imports by default. | No rule beyond rustfmt (FMT-02). | The rustfmt options for this (`imports_granularity`, `group_imports`) are unstable. |
| Glob re-exports | Google allows `pub use submodule::*` in non-leaf modules. Microsoft forbids glob re-exports. | Forbid (MOD-02). | One explicit path per item is easier to review and to keep semver-stable. |
| `std::sync` vs `parking_lot`/`crossbeam` | RCG `G.MTH.LCK.03`/`04` prefer the third-party crates. Google and Microsoft use std. | std or the async runtime's primitives (CON-09). | Fewer dependencies; add a crate only with a measured need. |
| `unwrap` vs `expect` | Google and RCG `P.ERR.02` prefer `expect`; RCG `G.ERR.02` warns against overusing `expect`. | No `unwrap`; `expect` only for invariants (ERR-03, ERR-04). | Both sides agree that failure must be considered; the message documents the invariant. |
| Panics | Google: some codebases avoid panics fully. Microsoft: a panic means stop. | Panic only on bugs; no recovery (ERR-02, ERR-10). | Works for both `panic = "unwind"` and `"abort"`. |
| Generics vs `dyn` | Google: generics in libraries, `dyn` where code size matters. Microsoft: concrete, then generics, then `dyn`. RCG `P.GEN.03`: fewer generics for compile time. | Concrete first; thin generics; `dyn` when generics nest or the code is not hot (TYP-12, CMP-01..03). | Follows the priority order: measured speed beats compile time, compile time beats convenience. |
| `#[non_exhaustive]` | Google and RCG: on public enums and structs. API Guidelines: private fields. | Only on published types that may grow (TYP-14). | Inside one workspace it forces wildcard arms that hide new variants from the compiler. |
| Error type shape | Microsoft: one struct per situation with a backtrace. API Guidelines: meaningful, implements `Error`. | Situation-specific types that implement `Error + Send + Sync`; enum or struct (ERR-08). | Both shapes meet the API Guidelines contract; the backtrace is a library choice. |
| Lint groups | Microsoft enables all of `pedantic` at warn. Clippy's docs advise cherry-picking. | Both tiers cherry-pick; the strict tier adds more `pedantic` lints and shows the whole group as an option (FMT-04). | Low noise by default keeps warnings meaningful. |
| `as` casts | RCG `G.TYP.01` restricts `as` broadly. Microsoft lints only trivial casts. | `From`/`TryFrom` first; lossy `as` needs a comment (NUM-01). | Lossy casts are a correctness risk; the full `as` ban is too noisy for the default tier. |
| Global static names | RCG `P.NAM.09`: `G_` prefix. | Rejected (NAM-01). | Conflicts with the standard casing that rustc checks. |
| Copyright header in every file | RCG `P.CMT.04`. | Not adopted. | A licensing decision, not a coding rule. Follow the project's licence policy. |

Codebase-specific rules from Google (`[chrome]`, `[Fuchsia]` tags) are not used.

---

## 1. Naming (NAM)

### NAM-01
**Use the standard casing: `UpperCamelCase` for types, traits and variants; `snake_case` for functions, methods, modules, locals, fields and macros; `SCREAMING_SNAKE_CASE` for constants and statics. Treat acronyms as one word (`HttpClient`, `Uuid`). Use ASCII identifiers.**
- Why: readers and tools rely on case to tell types from values.
- Check: `non_camel_case_types` (rustc, warn by default), `non_snake_case` (rustc, warn by default), `non_upper_case_globals` (rustc, warn by default), `clippy::upper_case_acronyms` (style), `non_ascii_idents` (rustc, allow by default).
- From: [API C-CASE](https://rust-lang.github.io/api-guidelines/naming.html#c-case), [Style Guide: advice](https://doc.rust-lang.org/style-guide/advice.html), [RCG G.NAM.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/G.NAM.01.html), [RCG G.VAR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/variables/G.VAR.02.html).

### NAM-02
**Name conversion methods by cost and ownership: `as_` is free and borrowed to borrowed; `to_` is expensive or borrowed to owned; `into_` consumes `self`.**
- Why: the prefix tells the caller what the call costs without reading it.
- Check: `clippy::wrong_self_convention` (style).
- From: [API C-CONV](https://rust-lang.github.io/api-guidelines/naming.html#c-conv), [RCG G.NAM.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/G.NAM.02.html), [Google: Naming methods](https://google.github.io/styleguide/rustguide.html#naming-methods).

### NAM-03
**Name getters after the field (`name()`, `name_mut()`), not `get_name()`. Use `set_`, `take_`, `replace_`, `with_` for the matching operations. Use bare `get` only when there is one obvious thing to get.**
- Why: this is the convention in std; `get_` adds nothing.
- Check: review.
- From: [API C-GETTER](https://rust-lang.github.io/api-guidelines/naming.html#c-getter), [RCG P.NAM.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/P.NAM.05.html), [Google: Getters and setters](https://google.github.io/styleguide/rustguide.html#getters-and-setters).

### NAM-04
**Write constructors as static inherent methods: `new`, `with_<detail>`, `try_new` when fallible, `from_<source>` for conversions. Return `Self` (or `Result<Self, _>`).**
- Why: callers find constructors in one place under predictable names.
- Check: `clippy::new_ret_no_self` (style), `clippy::self_named_constructors` (style).
- From: [API C-CTOR](https://rust-lang.github.io/api-guidelines/predictability.html#c-ctor), [Google: Naming initialization functions](https://google.github.io/styleguide/rustguide.html#naming-initialization-functions).

### NAM-05
**Name collection iterator methods `iter`, `iter_mut`, `into_iter`, and name the returned types after the method (`Iter`, `IterMut`, `IntoIter`).** [lib]
- Why: matches std and lets `for x in &collection` work.
- Check: strict tier `clippy::iter_without_into_iter` (pedantic), `clippy::into_iter_without_iter` (pedantic).
- From: [API C-ITER](https://rust-lang.github.io/api-guidelines/naming.html#c-iter), [API C-ITER-TY](https://rust-lang.github.io/api-guidelines/naming.html#c-iter-ty), [RCG P.NAM.06](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/P.NAM.06.html).

### NAM-06
**Choose names that carry meaning: no filler words (`Manager`, `Service`, `Helper`, `Util`, `Data`, `Info`), no abbreviations an outsider would not know, no type names inside variable names, no repetition of the module name (`http::Client`, not `http::HttpClient`), and one word order for related names (errors: verb, object, `Error`, as in `ParseAddrError`).**
- Why: the module path already gives context; filler words hide what the item does.
- Check: review; optional `clippy::module_name_repetitions` (restriction).
- From: [Microsoft M-WEASEL-WORDS](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-WEASEL-WORDS), [Microsoft M-SHORT-NAMES](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-SHORT-NAMES), [Google: Avoid abbreviations](https://google.github.io/styleguide/rustguide.html#avoid-abbreviations), [RCG P.NAM.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/P.NAM.04.html), [RCG P.NAM.08](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/P.NAM.08.html), [API C-WORD-ORDER](https://rust-lang.github.io/api-guidelines/naming.html#c-word-order), [RCG P.NAM.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/P.NAM.01.html).

### NAM-07
**Name Cargo features after what they enable (`std`, `serde`), without placeholder words (`use-std`, `with-serde`) and without negation (`no-std`).**
- Why: features are additive (DEP-04); a negative name inverts that meaning.
- Check: `clippy::negative_feature_names` (cargo), `clippy::redundant_feature_names` (cargo).
- From: [API C-FEATURE](https://rust-lang.github.io/api-guidelines/naming.html#c-feature), [RCG P.NAM.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/P.NAM.02.html), [RCG G.CAR.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/G.CAR.03.html).

### NAM-08
**When a name is a keyword, use a raw identifier (`r#type`) or a trailing underscore (`type_`); do not misspell it (`typ`, `krate`).**
- Why: misspellings are hard to search for and look like typos.
- Check: review.
- From: [Style Guide: advice](https://doc.rust-lang.org/style-guide/advice.html), [RCG P.NAM.07](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/naming/P.NAM.07.html).

### NAM-09
**Name lifetimes after what they borrow from (`'src`, `'conn`) when an item has more than one; a single `'a` is fine. Name extension traits `<Type>Ext`.**
- Why: descriptive lifetimes explain which borrow outlives which.
- Check: review.
- From: [Google: Naming lifetimes](https://google.github.io/styleguide/rustguide.html#naming-lifetimes), [Google: Naming extension traits](https://google.github.io/styleguide/rustguide.html#naming-extension-traits), [RCG P.MEM.LFT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/memory/lifetime/P.MEM.LFT.01.html).

## 2. Formatting, style and static checks (FMT)

### FMT-01
**Run `cargo fmt --all` before you finish. Do not format by hand. Use `#[rustfmt::skip]` only for deliberate alignment (tables, matrices) or generated code, with a comment that says why.**
- Why: one automatic format removes style debates and diff noise.
- Check: `cargo fmt --all -- --check`.
- From: [Style Guide](https://doc.rust-lang.org/style-guide/), [Google: Use the appropriate formatting tool](https://google.github.io/styleguide/rustguide.html#use-the-appropriate-formatting-tool), [RCG P.FMT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/fmt/P.FMT.01.html), [Google: Opting out of automatic formatting](https://google.github.io/styleguide/rustguide.html#opting-out-of-automatic-formatting).

### FMT-02
**Keep rustfmt defaults. In `rustfmt.toml`, set `style_edition` to the crate edition; add only stable options, and do not depend on unstable ones.**
- Why: `cargo fmt` takes the style edition from `Cargo.toml`, but plain `rustfmt` defaults to 2015, so results differ unless it is set. Unstable options need nightly.
- Check: rustfmt `style_edition` (stable); see [lints/rustfmt.toml](lints/rustfmt.toml).
- From: [rustfmt Configurations](https://github.com/rust-lang/rustfmt/blob/main/Configurations.md).

### FMT-03
**Run Clippy on the crates you changed (`cargo clippy -p <crate> --all-targets`) and leave no warnings in changed code.**
- Why: Clippy finds bugs and non-idiomatic code that review misses.
- Check: `cargo clippy ... -- -D warnings` in the gate; the default-on Clippy groups plus Microsoft's rustc additions `redundant_imports` (rustc, allow by default), `redundant_lifetimes` (rustc, allow by default), `unused_lifetimes` (rustc, allow by default), `ambiguous_negative_literals` (rustc, allow by default).
- From: [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION), [Clippy: lint groups](https://doc.rust-lang.org/clippy/lints.html).

### FMT-04
**Configure lints once in `[workspace.lints]`, and set `[lints] workspace = true` in every member. Do not enable whole `restriction` or `nursery` groups; cherry-pick lints.**
- Why: one source of truth; `restriction` lints contradict each other and `nursery` lints are unfinished.
- Check: review; template in [lints/Cargo.lints.toml](lints/Cargo.lints.toml).
- From: [Microsoft M-CARGO-WORKSPACE](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-CARGO-WORKSPACE), [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION), [Clippy: lint groups](https://doc.rust-lang.org/clippy/lints.html).

### FMT-05
**Silence a lint locally with `#[expect(lint, reason = "...")]`, not `#[allow]`. Use `#[allow]` only on generated code or inside macros. Never weaken lint levels, `clippy.toml`, `rustfmt.toml` or CI checks to make a change pass.**
- Why: an `expect` warns when it is no longer needed, so stale suppressions do not pile up; the gate exists to catch the change that fails it.
- Check: `clippy::allow_attributes_without_reason` (restriction); strict tier: `clippy::allow_attributes` (restriction); stale ones: `unfulfilled_lint_expectations` (rustc, warn by default); review the diff of lint configuration files.
- From: [Microsoft M-LINT-OVERRIDE-EXPECT](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-LINT-OVERRIDE-EXPECT), [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION), [Clippy: lint groups](https://doc.rust-lang.org/clippy/lints.html).

### FMT-06
**Use Rust's expression form: bind the result of `if`/`match` directly instead of declaring a variable and assigning it in branches. Use a plain `for` loop when an iterator chain needs side effects or several branches.**
- Why: expression form keeps a value immutable and its origin in one place; long chains hide control flow.
- Check: `clippy::needless_late_init` (style); `clippy::needless_for_each` (pedantic).
- From: [Style Guide: advice](https://doc.rust-lang.org/style-guide/advice.html), [RCG P.VAR.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/variables/P.VAR.01.html), [RCG P.CTF.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/control-flow/P.CTF.01.html), [Google: combinator chains](https://google.github.io/styleguide/rustguide.html#dont-use-iterator-combinator-chains-when-direct-iteration-is-clearer).

```rust
// Bad
let level;
if verbose { level = Level::Debug; } else { level = Level::Info; }
// Good
let level = if verbose { Level::Debug } else { Level::Info };
```

### FMT-07
**Build strings with `format!` or `write!`, not chains of `+`.**
- Why: one allocation and the template shows the result shape.
- Check: strict tier `clippy::string_add` (restriction), `clippy::format_push_string` (pedantic).
- From: [Google: Prefer format!/write!](https://google.github.io/styleguide/rustguide.html#prefer-format--write-for-concatenation-and-substitution), [RCG P.STR.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/strings/P.STR.05.html).

### FMT-08
**Use stable Rust. Do not add `#![feature(...)]`.**
- Why: unstable features can change or disappear and force a nightly toolchain on every user.
- Check: build with the stable toolchain.
- From: [Google: No unstable features](https://google.github.io/styleguide/rustguide.html#no-unstable-features).

## 3. Modules, visibility and crate layout (MOD)

### MOD-01
**Keep items private by default. Use `pub(crate)` for crate-internal sharing; use `pub` only for items another crate uses.**
- Why: a smaller public surface is easier to change and to review.
- Check: strict tier `unreachable_pub` (rustc, allow by default).
- From: [RCG P.MOD.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/module/P.MOD.01.html), [API C-STRUCT-PRIVATE](https://rust-lang.github.io/api-guidelines/future-proofing.html#c-struct-private).

### MOD-02
**Give each public item exactly one public path. Re-export items one by one (`pub use inner::Item;`), never with a glob, and do not define a `prelude` module.** [lib]
- Why: several paths to one item confuse readers and agents and make refactors break callers.
- Check: review; `clippy::wildcard_imports` (pedantic) with `warn-on-all-wildcard-imports = true` also flags `pub use x::*`.
- From: [Microsoft M-SINGLE-ITEM-PATH](https://microsoft.github.io/rust-guidelines/guidelines/ai/#M-SINGLE-ITEM-PATH), [Microsoft M-NO-GLOB-REEXPORTS](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-NO-GLOB-REEXPORTS), [Microsoft M-NO-PRELUDE](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-NO-PRELUDE).

### MOD-03
**Do not use glob imports, except `use super::*;` in a test module and `use MyEnum::*;` inside a function that mostly matches on that enum.**
- Why: globs hide where names come from and can change meaning when the source module grows.
- Check: `clippy::wildcard_imports` (pedantic), `clippy::enum_glob_use` (pedantic).
- From: [Google: Don't use glob imports](https://google.github.io/styleguide/rustguide.html#dont-use-glob-imports), [RCG G.MOD.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/module/G.MOD.03.html).

### MOD-04
**Keep non-leaf modules (`lib.rs` and module files that declare submodules) to declarations, re-exports and module docs. Put logic in leaf modules named after their main item. Use one module file layout in a project (`foo.rs` + `foo/`, or `foo/mod.rs`) and avoid `#[path]`.**
- Why: readers find code by path; structure and content do not mix.
- Check: review; strict tier, pick one layout: `clippy::mod_module_files` (restriction) or `clippy::self_named_module_files` (restriction).
- From: [Google: non-leaf modules](https://google.github.io/styleguide/rustguide.html#dont-implement-apis-in-non-leaf-modules), [Google: name leaf modules](https://google.github.io/styleguide/rustguide.html#name-source-files-and-leaf-modules-after-the-primary-user-defined-item), [Microsoft M-BALANCED-MODULES](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-BALANCED-MODULES), [RCG G.MOD.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/module/G.MOD.04.html), [Style Guide: advice](https://doc.rust-lang.org/style-guide/advice.html).

### MOD-05
**Import types by name; call free functions through their module (`fs::read`, not bare `read`); import traits in the narrowest scope that uses them.**
- Why: the call site shows where a function comes from; a type name is unambiguous on its own.
- Check: review.
- From: [Google: qualify calls](https://google.github.io/styleguide/rustguide.html#qualify-calls-to-top-level-functions-with-a-module-name), [Google: import type names](https://google.github.io/styleguide/rustguide.html#type-names-should-be-imported-and-used-unqualified), [Google: local trait imports](https://google.github.io/styleguide/rustguide.html#prefer-locally-scoped-trait-imports), [RCG G.MOD.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/module/G.MOD.01.html).

### MOD-06
**Keep public signatures to std types and your own types. Do not re-export another crate's items, except to re-export your own helper crates such as `foo_proc`.** [lib]
- Why: a third-party type in your API makes that crate's version part of your semver contract.
- Check: review; for published crates, `cargo semver-checks` (tool).
- From: [Microsoft M-DONT-LEAK-TYPES](https://microsoft.github.io/rust-guidelines/guidelines/libs/interop/#M-DONT-LEAK-TYPES), [Microsoft M-FOREIGN-REEXPORTS](https://microsoft.github.io/rust-guidelines/guidelines/libs/interop/#M-FOREIGN-REEXPORTS), [API C-STABLE](https://rust-lang.github.io/api-guidelines/necessities.html#c-stable).

### MOD-07
**In a workspace, make every crate a member, keep crates as siblings in one folder (for example `crates/`), and do not nest crates inside crates.**
- Why: flat layouts are easy to navigate and keep the dependency graph visible.
- Check: review.
- From: [Microsoft M-CRATES-IN-WORKSPACE](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-CRATES-IN-WORKSPACE), [Microsoft M-CRATES-FLAT-FOLDER](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-CRATES-FLAT-FOLDER), [matklad: Large Rust workspaces](https://matklad.github.io/2021/08/22/large-rust-workspaces.html).

### MOD-08
**Split an executable into a thin `main.rs` (argument parsing, wiring, exit code) and a `lib.rs` that holds the logic.**
- Why: logic in a library can be tested and reused; `main.rs` cannot be imported.
- Check: review.
- From: [RCG G.CAR.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/G.CAR.01.html).

### MOD-09
**Avoid global mutable state: no `static mut`, no interior-mutable `const`, no code that runs before `main`. Pass state explicitly.**
- Why: globals hide dependencies, break test isolation and race under concurrency.
- Check: `static_mut_refs` (rustc, warn by default), `clippy::declare_interior_mutable_const` (suspicious).
- From: [Microsoft M-AVOID-STATICS](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-AVOID-STATICS), [RCG G.STV.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/statics/G.STV.01.html), [RCG G.CNS.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/consts/G.CNS.03.html), [Google: no code before main](https://google.github.io/styleguide/rustguide.html#do-not-run-code-before-main).

## 4. Types and traits (TYP)

### TYP-01
**Use a newtype (not a primitive or a type alias) for values with distinct meaning or invariants, and use the std type that fits the domain (`PathBuf` for paths, `Duration` for spans, `OsString` for OS strings).**
- Why: the compiler then rejects mixed-up arguments and wrong units.
- Check: review.
- From: [API C-NEWTYPE](https://rust-lang.github.io/api-guidelines/type-safety.html#c-newtype), [Microsoft M-STRONG-TYPES](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-STRONG-TYPES), [Google: Newtypes versus type aliases](https://google.github.io/styleguide/rustguide.html#newtypes-versus-type-aliases), [RCG P.TYP.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/P.TYP.01.html).

```rust
// Bad
fn transfer(from: u64, to: u64, cents: i64);
// Good
fn transfer(from: AccountId, to: AccountId, amount: Cents);
```

### TYP-02
**Give a newtype with invariants private fields and a fallible constructor (`TryFrom`, `FromStr` or `try_new`); do not add an infallible `From` from the raw type.**
- Why: every value of the type is then valid, so code that receives it needs no re-check.
- Check: review.
- From: [Microsoft M-STRONG-TYPES-GUARD](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-STRONG-TYPES-GUARD), [API C-VALIDATE](https://rust-lang.github.io/api-guidelines/dependability.html#c-validate).

### TYP-03
**Replace `bool` or `Option` parameters whose meaning is unclear at the call site with an enum. Group more than about four parameters into a struct or builder.**
- Why: `new(true, false)` cannot be read without the signature.
- Check: `clippy::fn_params_excessive_bools` (pedantic), `clippy::struct_excessive_bools` (pedantic), `clippy::too_many_arguments` (complexity).
- From: [API C-CUSTOM-TYPE](https://rust-lang.github.io/api-guidelines/type-safety.html#c-custom-type), [Google: Prefer enum over bool](https://google.github.io/styleguide/rustguide.html#prefer-enum-over-bool-in-apis), [RCG G.FUD.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/fn-design/G.FUD.01.html), [RCG G.FUD.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/fn-design/G.FUD.03.html), [RCG G.TYP.SCT.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/struct/G.TYP.SCT.02.html), [Microsoft M-INIT-CASCADED](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-INIT-CASCADED).

### TYP-04
**Use a builder for types with several optional inputs; validate in `build()` and return `Result` from it.**
- Why: avoids constructor explosion and keeps validation in one place.
- Check: review.
- From: [API C-BUILDER](https://rust-lang.github.io/api-guidelines/type-safety.html#c-builder), [Microsoft M-INIT-BUILDER](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-INIT-BUILDER), [Microsoft M-BUILD-RESULT](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-BUILD-RESULT).

### TYP-05
**Prefer structs with named fields to tuples and tuple structs, except single-field newtypes. Keep tuples to three elements or fewer.**
- Why: `.0`/`.1` say nothing about meaning.
- Check: review; `clippy::type_complexity` (complexity) catches deeply nested tuple types.
- From: [Google: Prefer named fields](https://google.github.io/styleguide/rustguide.html#prefer-named-fields-over-tuples-and-non-newtype-tuple-structs), [RCG G.TYP.TUP.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/tuple/G.TYP.TUP.01.html).

### TYP-06
**Implement `Debug` for all public types. Implement `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `PartialOrd`, `Ord`, `Default` and `Display` where the semantics fit. Put trait bounds on `impl` blocks, not on the type definition.** [lib]
- Why: the orphan rule stops users from adding these impls later; derives already add the bounds they need.
- Check: `missing_debug_implementations` (rustc, allow by default).
- From: [API C-COMMON-TRAITS](https://rust-lang.github.io/api-guidelines/interoperability.html#c-common-traits), [API C-DEBUG](https://rust-lang.github.io/api-guidelines/debuggability.html#c-debug), [API C-STRUCT-BOUNDS](https://rust-lang.github.io/api-guidelines/future-proofing.html#c-struct-bounds), [Microsoft M-PUBLIC-DEBUG](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-PUBLIC-DEBUG), [Microsoft M-PUBLIC-DISPLAY](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-PUBLIC-DISPLAY).

### TYP-07
**Implement `Debug` by hand for types that hold secrets (tokens, keys, passwords), print a redacted form, and add a test that the secret does not appear in the output.**
- Why: `Debug` output ends up in logs and panic messages.
- Check: review; the test.
- From: [Microsoft M-PUBLIC-DEBUG](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-PUBLIC-DEBUG).

### TYP-08
**Derive a trait when the derived impl is correct, instead of writing it by hand. Keep `Hash` consistent with `Eq` and `Ord` consistent with `PartialOrd`.**
- Why: derived impls cannot drift from the fields; inconsistent pairs break `HashMap` and sorting.
- Check: `clippy::derivable_impls` (complexity), `clippy::derived_hash_with_manual_eq` (correctness), `clippy::derive_ord_xor_partial_ord` (correctness), `clippy::expl_impl_clone_on_copy` (pedantic).
- From: [Google: Use derive](https://google.github.io/styleguide/rustguide.html#use-derive-if-possible), [RCG G.TRA.BLN.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/traits/std-builtin/G.TRA.BLN.03.html), [RCG G.TRA.BLN.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/traits/std-builtin/G.TRA.BLN.04.html), [RCG G.TRA.BLN.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/traits/std-builtin/G.TRA.BLN.05.html), [RCG G.TRA.BLN.09](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/traits/std-builtin/G.TRA.BLN.09.html).

### TYP-09
**Use std traits instead of ad-hoc methods: implement `From`/`TryFrom` (never `Into`/`TryInto`), `FromStr`, `AsRef`, `Display` (not an inherent `to_string`), `Default` alongside an argument-less `new`, and `FromIterator`/`Extend` for collections.**
- Why: std traits plug into `?`, `.parse()`, `.collect()`, `format!` and generic code.
- Check: `clippy::from_over_into` (style), `clippy::should_implement_trait` (style), `clippy::new_without_default` (style), `clippy::inherent_to_string` (style).
- From: [API C-CONV-TRAITS](https://rust-lang.github.io/api-guidelines/interoperability.html#c-conv-traits), [API C-COLLECT](https://rust-lang.github.io/api-guidelines/interoperability.html#c-collect), [Google: Implement std traits](https://google.github.io/styleguide/rustguide.html#implement-known-standard-library-traits-instead-of-custom-methods), [RCG G.TRA.BLN.08](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/traits/std-builtin/G.TRA.BLN.08.html).

### TYP-10
**Make trait impls unsurprising: implement `Deref`/`DerefMut` only for smart-pointer types (never to imitate inheritance), and overload operators only with their usual mathematical meaning.**
- Why: `Deref` takes part in method lookup; misuse makes calls resolve to unexpected methods.
- Check: review.
- From: [API C-DEREF](https://rust-lang.github.io/api-guidelines/predictability.html#c-deref), [API C-OVERLOAD](https://rust-lang.github.io/api-guidelines/predictability.html#c-overload), [RCG G.TRA.BLN.10](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/traits/std-builtin/G.TRA.BLN.10.html), [Google: operator overloading](https://google.github.io/styleguide/rustguide.html#avoid-overloading-for-non-mathematical-purposes).

### TYP-11
**Define a trait only for generic code, `dyn` dispatch, or an extension trait. Put a type's core behaviour in inherent methods, and computation that has no clear receiver in free functions.**
- Why: traits used only to group methods add indirection and imports for no gain.
- Check: review.
- From: [Google: trait vs inherent impl](https://google.github.io/styleguide/rustguide.html#when-to-define-a-trait-versus-an-inherent-implementation), [Microsoft M-ESSENTIAL-FN-INHERENT](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-ESSENTIAL-FN-INHERENT), [Microsoft M-REGULAR-FN](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-REGULAR-FN), [API C-METHOD](https://rust-lang.github.io/api-guidelines/predictability.html#c-method).

### TYP-12
**For dependencies of a type, use a concrete type first, then a generic parameter, then `dyn Trait`. Switch to `dyn Trait` (wrapped in your own named type) when generic parameters start to nest or spread through many types.**
- Why: generic parameters infect every containing type and multiply compile work (CMP-03).
- Check: review.
- From: [Microsoft M-DI-HIERARCHY](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-DI-HIERARCHY), [Microsoft M-SIMPLE-ABSTRACTIONS](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-SIMPLE-ABSTRACTIONS), [Google: Generics versus dyn Trait](https://google.github.io/styleguide/rustguide.html#generics-versus-dyn-trait), [RCG P.TRA.OBJ.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/traits/trait-object/P.TRA.OBJ.01.html).

### TYP-13
**Keep implementation choices out of public APIs: do not expose `Rc`, `Arc`, `Box`, `RefCell` or `Mutex` in signatures (make a shared service a cheap-to-clone type that holds an `Arc` to its inner state); seal traits that other crates must not implement; make a trait object-safe if it may be used as `dyn`.** [lib]
- Why: exposed wrappers freeze a design choice; sealing allows adding methods without a breaking change; object safety is hard to add later.
- Check: review.
- From: [Microsoft M-AVOID-WRAPPERS](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-AVOID-WRAPPERS), [Microsoft M-SERVICES-CLONE](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-SERVICES-CLONE), [API C-SEALED](https://rust-lang.github.io/api-guidelines/future-proofing.html#c-sealed), [API C-OBJECT](https://rust-lang.github.io/api-guidelines/flexibility.html#c-object).

### TYP-14
**Give public structs private fields unless they are plain data. Add `#[non_exhaustive]` only to published enums and structs that may grow; not to types shared inside one workspace.** [lib]
- Why: public fields and exhaustive enums freeze the layout; inside a workspace, exhaustive matches let the compiler find every site to update.
- Check: review; for published crates, optional `clippy::exhaustive_enums` (restriction), `clippy::exhaustive_structs` (restriction); `clippy::manual_non_exhaustive` (style).
- From: [API C-STRUCT-PRIVATE](https://rust-lang.github.io/api-guidelines/future-proofing.html#c-struct-private), [Google: non_exhaustive enums](https://google.github.io/styleguide/rustguide.html#apply-non_exhaustive-to-publicly-exposed-enums), [RCG G.TYP.SCT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/struct/G.TYP.SCT.01.html), [RCG G.TYP.ENM.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/enum/G.TYP.ENM.05.html).

### TYP-15
**Add `#[must_use]` to functions whose result is the whole point (pure functions, builders, guards) and to types that do nothing unless used.**
- Why: an ignored result is usually a bug.
- Check: `unused_must_use` (rustc, warn by default); strict tier `clippy::must_use_candidate` (pedantic), `clippy::return_self_not_must_use` (pedantic).
- From: [Google: must_use on functions](https://google.github.io/styleguide/rustguide.html#must_use-on-functions), [Google: must_use on types](https://google.github.io/styleguide/rustguide.html#must_use-on-types).

## 5. Ownership and borrowing (OWN)

### OWN-01
**Accept the most general borrowed form: `&str`, `&[T]`, `&T`, `Option<&T>`, `&Path`; not `&String`, `&Vec<T>`, `&Box<T>`, `&Option<T>`.**
- Why: callers can pass more types without converting.
- Check: `clippy::ptr_arg` (style), `clippy::borrowed_box` (complexity), `clippy::ref_option` (pedantic).
- From: [Google: flexible references](https://google.github.io/styleguide/rustguide.html#accept-flexible-references-for-immutable-data), [RCG G.MEM.BOX.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/memory/box/G.MEM.BOX.01.html).

### OWN-02
**Take a parameter by value when the function stores or consumes it; borrow it otherwise. Never borrow and then clone inside. Take generic readers and writers by value (`R: Read`, `W: Write`); callers pass `&mut r` to keep ownership.**
- Why: the caller decides whether to copy, and can move a value it no longer needs.
- Check: strict tier `clippy::needless_pass_by_value` (pedantic).
- From: [API C-CALLER-CONTROL](https://rust-lang.github.io/api-guidelines/flexibility.html#c-caller-control), [API C-RW-VALUE](https://rust-lang.github.io/api-guidelines/interoperability.html#c-rw-value), [Microsoft M-IMPL-IO](https://microsoft.github.io/rust-guidelines/guidelines/libs/interop/#M-IMPL-IO).

### OWN-03
**Clone only when you need an independent copy; restructure borrows instead of cloning to satisfy the borrow checker.**
- Why: needless clones cost allocations and hide ownership bugs.
- Check: review; strict tier `clippy::implicit_clone` (pedantic).
- From: [Google: borrows over deep copies](https://google.github.io/styleguide/rustguide.html#prefer-borrows-over-deep-copies).

### OWN-04
**Prefer unique ownership. Use `Rc`/`Arc` only when ownership is truly shared, and clone them as `Arc::clone(&x)`.**
- Why: shared ownership blurs lifetimes; the explicit form shows a cheap pointer copy, not a deep copy.
- Check: `clippy::clone_on_ref_ptr` (restriction).
- From: [Google: Avoid Rc and Arc](https://google.github.io/styleguide/rustguide.html#avoid-rc-and-arc), [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION).

### OWN-05
**Return results as values, tuples or structs; do not use out-parameters. Exception: a `&mut` buffer the caller owns and reuses (PRF-03).**
- Why: return values are clearer and compile efficiently.
- Check: review.
- From: [API C-NO-OUT](https://rust-lang.github.io/api-guidelines/predictability.html#c-no-out).

### OWN-06
**Pass small `Copy` types by value and large types by reference. Do not box locals or collections (`Box<Vec<T>>`, `Vec<Box<T>>` of sized `T`).**
- Why: avoids needless copies and extra indirection.
- Check: `clippy::box_collection` (perf), `clippy::boxed_local` (perf), `clippy::vec_box` (complexity); strict tier `clippy::trivially_copy_pass_by_ref` (pedantic), `clippy::large_types_passed_by_value` (pedantic).
- From: [RCG G.FUD.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/fn-design/G.FUD.02.html), [RCG G.FUD.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/fn-design/G.FUD.04.html), [RCG G.MEM.BOX.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/memory/box/G.MEM.BOX.02.html), [RCG G.MEM.BOX.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/memory/box/G.MEM.BOX.03.html).

### OWN-07
**Use `RefCell` only when borrows cannot be expressed statically, and use `try_borrow`/`try_borrow_mut` where a conflict is possible.**
- Why: a `RefCell` conflict is a runtime panic.
- Check: review.
- From: [RCG P.MEM.SPT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/memory/smart-ptr/P.MEM.SPT.01.html).

## 6. Error handling and panics (ERR)

### ERR-01
**Return `Result` for failures a correct program can meet (I/O, parsing, user input, network). Return `Option` when absence has one obvious cause.**
- Why: callers must decide how to handle expected failure; a panic takes that decision away.
- Check: review.
- From: [Google: Reporting errors](https://google.github.io/styleguide/rustguide.html#reporting-errors), [Microsoft M-PANIC-ON-BUG](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-PANIC-ON-BUG), [API C-VALIDATE](https://rust-lang.github.io/api-guidelines/dependability.html#c-validate).

### ERR-02
**Panic only on bugs: violated invariants and contract misuse. Never use a panic as control flow or as an error channel.**
- Why: a panic means "stop"; with `panic = "abort"` there is no recovery at all.
- Check: review.
- From: [Microsoft M-PANIC-IS-STOP](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-PANIC-IS-STOP), [Microsoft M-PANIC-ON-BUG](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-PANIC-ON-BUG).

### ERR-03
**Do not call `unwrap()` outside tests. Use `?`, `match`/`if let`/`let ... else` (never check-then-unwrap), a default (`unwrap_or_else`, `unwrap_or_default`), or `expect` (ERR-04). In a function that returns `Result`, propagate instead of panicking.**
- Why: an `unwrap` panic has no context, and the reader cannot tell whether failure was considered.
- Check: `clippy::unnecessary_unwrap` (complexity); `clippy::unwrap_used` (restriction) with `allow-unwrap-in-tests = true`; strict tier `clippy::unwrap_in_result` (restriction), `clippy::panic_in_result_fn` (restriction).
- From: [RCG G.ERR.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/error-handle/G.ERR.01.html), [RCG G.ERR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/error-handle/G.ERR.02.html), [Google: Prefer expect over unwrap](https://google.github.io/styleguide/rustguide.html#prefer-expect-over-unwrap), [Google: pattern matching over check-then-unwrap](https://google.github.io/styleguide/rustguide.html#prefer-pattern-matching-over-check-then-unwrap), [RCG P.CTF.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/control-flow/P.CTF.02.html).

### ERR-04
**Use `expect` only where an invariant guarantees success. Write the message as the invariant ("regex literal is valid"), not as the failure.**
- Why: the message documents why the panic cannot happen and points to the broken assumption if it does.
- Check: review; `clippy::expect_fun_call` (perf); strict tier `clippy::expect_used` (restriction) for crates that must not panic.
- From: [RCG P.ERR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/error-handle/P.ERR.02.html), [RCG G.ERR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/error-handle/G.ERR.02.html), [Google: Prefer expect over unwrap](https://google.github.io/styleguide/rustguide.html#prefer-expect-over-unwrap).

```rust
// Bad
let re = Regex::new(PATTERN).unwrap();
// Good
let re = Regex::new(PATTERN).expect("PATTERN is a valid regex literal");
```

### ERR-05
**Give every intentional panic (`panic!`, `assert!`, `unreachable!`) a message that states what went wrong and includes the relevant values.**
- Why: the panic message is often the only evidence of the bug.
- Check: strict tier `clippy::missing_assert_message` (restriction).
- From: [Microsoft M-PANIC-MESSAGE](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-PANIC-MESSAGE), [Google: context strings in assertions](https://google.github.io/styleguide/rustguide.html#use-context-strings-to-clarify-assertions).

### ERR-06
**Propagate errors with `?` and `From` impls, and keep the source error. Do not repeat `map_err` closures at call sites, and do not replace an error with a new one that drops the cause.**
- Why: the cause chain is what makes an error diagnosable.
- Check: `clippy::map_err_ignore` (restriction).
- From: [Microsoft M-FROM-ERROR](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-FROM-ERROR), [API C-GOOD-ERR](https://rust-lang.github.io/api-guidelines/interoperability.html#c-good-err).

### ERR-07
**Never discard a `Result` silently. `let _ = fallible();` and `.ok();` need a comment that says why ignoring the error is correct.**
- Why: a dropped error is a failure nobody sees.
- Check: `unused_must_use` (rustc, warn by default), `clippy::unused_result_ok` (restriction); strict tier `clippy::let_underscore_must_use` (restriction).
- From: [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION).

### ERR-08
**Give library errors a situation-specific type (enum or struct) that implements `std::error::Error`, `Display`, `Debug`, `Send`, `Sync` and `'static`, and exposes its cause through `source()`. Never use `()` or `String` as an error type. Write `Display` messages in lowercase, without trailing punctuation.** [lib]
- Why: callers need to match on the kind, chain the cause and send the error across threads.
- Check: review.
- From: [API C-GOOD-ERR](https://rust-lang.github.io/api-guidelines/interoperability.html#c-good-err), [Microsoft M-ERRORS-CANONICAL-STRUCTS](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-ERRORS-CANONICAL-STRUCTS).

### ERR-09
**In an application, you may use one dynamic error type (such as `anyhow::Error`) and add context at boundaries. Do not mix several such error crates, and do not return them from library crates.**
- Why: applications mostly report errors; libraries must let callers match on them.
- Check: review.
- From: [Microsoft M-APP-ERROR](https://microsoft.github.io/rust-guidelines/guidelines/apps/#M-APP-ERROR).

### ERR-10
**Do not use `catch_unwind` to continue after a panic. The only exception is a task or request boundary in a server, so that other work can finish before a restart.**
- Why: state observed after a caught panic may be broken; `catch_unwind` does nothing under `panic = "abort"`.
- Check: review.
- From: [Microsoft M-PANIC-CONTINUATION](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-PANIC-CONTINUATION), [Google: Don't call catch_unwind](https://google.github.io/styleguide/rustguide.html#dont-call-paniccatch_unwind).

### ERR-11
**Destructors must not fail or block. Provide an explicit `close()` or `shutdown()` that returns `Result`; `Drop` then cleans up best-effort and logs errors.**
- Why: a panic in `drop` during unwinding aborts the process.
- Check: review.
- From: [API C-DTOR-FAIL](https://rust-lang.github.io/api-guidelines/dependability.html#c-dtor-fail), [API C-DTOR-BLOCK](https://rust-lang.github.io/api-guidelines/dependability.html#c-dtor-block).

### ERR-12
**Leave no `todo!()`, `unimplemented!()` or `dbg!()` in finished code.**
- Why: they are placeholders that panic or print at runtime.
- Check: `clippy::todo` (restriction), `clippy::unimplemented` (restriction), `clippy::dbg_macro` (restriction).
- From: [RCG G.MAC.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/G.MAC.01.html), [Microsoft M-LOG-NOT-PRINT](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-LOG-NOT-PRINT).

### ERR-13
**Emit diagnostics through a logging or tracing facility with structured fields and stable event names, not `println!`/`eprintln!`. Redact secrets. (Command-line output that is the program's product is exempt.)**
- Why: structured events can be filtered and searched; prints cannot.
- Check: strict tier `clippy::print_stdout` (restriction), `clippy::print_stderr` (restriction) (expect them at the CLI output layer).
- From: [Microsoft M-LOG-NOT-PRINT](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-LOG-NOT-PRINT), [Microsoft M-LOG-STRUCTURED](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-LOG-STRUCTURED).

## 7. Unsafe code (UNS)

### UNS-01
**Do not write `unsafe` unless no safe alternative exists: FFI, a new abstraction safe code cannot express, or a hot path with a benchmark. Never use it to bypass borrow, lifetime or `Send`/`Sync` checks. Forbid `unsafe` in crates that do not need it.**
- Why: every `unsafe` block is a proof obligation that the compiler no longer checks.
- Check: `unsafe_code` (rustc, allow by default) (`deny` in the workspace; `#![forbid(unsafe_code)]` per crate).
- From: [Microsoft M-UNSAFE](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-UNSAFE), [Google: Unsafe code](https://google.github.io/styleguide/rustguide.html#unsafe-code), [RCG P.UNS.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/P.UNS.01.html), [RCG P.UNS.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/P.UNS.02.html).

### UNS-02
**Precede every `unsafe` block with a `// SAFETY:` comment that shows why each precondition holds.**
- Why: the reviewer checks the argument, not just the code.
- Check: `clippy::undocumented_unsafe_blocks` (restriction), `clippy::unnecessary_safety_comment` (restriction); strict tier `clippy::multiple_unsafe_ops_per_block` (restriction).
- From: [RCG P.UNS.SAS.09](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/safe_abstract/P.UNS.SAS.09.html), [Google: Unsafe code](https://google.github.io/styleguide/rustguide.html#unsafe-code), [Microsoft M-UNSAFE](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-UNSAFE).

### UNS-03
**Document every `unsafe fn` and `unsafe trait` with a `# Safety` section that lists the caller's obligations. Mark a function `unsafe` only if misuse can cause undefined behaviour.**
- Why: `unsafe` is a contract; it must say what the caller promises, and must not be used for "dangerous but safe".
- Check: `clippy::missing_safety_doc` (style), `clippy::unnecessary_safety_doc` (restriction).
- From: [RCG G.UNS.SAS.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/safe_abstract/G.UNS.SAS.01.html), [Microsoft M-UNSAFE-IMPLIES-UB](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-UNSAFE-IMPLIES-UB), [API C-FAILURE](https://rust-lang.github.io/api-guidelines/documentation.html#c-failure).

### UNS-04
**Inside an `unsafe fn`, still wrap each unsafe operation in an explicit `unsafe {}` block.**
- Why: `unsafe fn` states the caller's obligations; the explicit blocks mark each place the body relies on them, so each gets its own `SAFETY:` argument.
- Check: `unsafe_op_in_unsafe_fn` (rustc, allow by default) (warn by default from edition 2024).
- From: [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION), [Edition Guide: unsafe_op_in_unsafe_fn](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html).

### UNS-05
**Wrap unsafe code in a small module with a safe API that is sound for all inputs, including when a panic happens in the middle of an operation. Safe code must never be able to cause undefined behaviour through it. Guard memory-safety conditions with `assert!`, not `debug_assert!`.**
- Why: soundness is a property of the API, not of the call sites you know about; `debug_assert!` disappears in release builds.
- Check: review.
- From: [Microsoft M-UNSOUND](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-UNSOUND), [RCG P.UNS.SAS.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/safe_abstract/P.UNS.SAS.01.html), [RCG P.UNS.SAS.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/safe_abstract/P.UNS.SAS.02.html), [Google: Unsafe code](https://google.github.io/styleguide/rustguide.html#unsafe-code), [RCG G.UNS.SAS.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/safe_abstract/G.UNS.SAS.02.html).

### UNS-06
**Run the tests of unsafe code under Miri.**
- Why: Miri detects many kinds of undefined behaviour that tests pass over.
- Check: `cargo +nightly miri test` (tool).
- From: [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION), [Microsoft M-UNSAFE](https://microsoft.github.io/rust-guidelines/guidelines/correctness/#M-UNSAFE).

### UNS-07
**Cast pointers with `pointer::cast`, `cast_mut`/`cast_const` and `NonNull`, not `as`. Do not `transmute` between numbers, characters or pointers when a safe conversion exists (`from_bits`, `to_ne_bytes`, `char::from_u32`).**
- Why: the explicit methods keep the pointee type visible and cannot silently change mutability.
- Check: strict tier `clippy::ptr_as_ptr` (pedantic), `clippy::as_pointer_underscore` (restriction); `unnecessary_transmutes` (rustc, warn by default).
- From: [RCG G.UNS.PTR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/raw_ptr/G.UNS.PTR.02.html), [RCG G.UNS.PTR.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/unsafe_rust/raw_ptr/G.UNS.PTR.03.html), [RCG G.TYP.CHR.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/char/G.TYP.CHR.03.html), [RCG G.TYP.FLT.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/float/G.TYP.FLT.05.html).

## 8. Concurrency and async (CON)

### CON-01
**Never hold a `std::sync` lock guard or a `RefCell` borrow across `.await`. Drop the guard in an inner block first, or use the runtime's async lock.**
- Why: the task may suspend while holding the lock, which blocks the thread or deadlocks.
- Check: `clippy::await_holding_lock` (suspicious), `clippy::await_holding_refcell_ref` (suspicious); other guard types via `await-holding-invalid-types` and `clippy::await_holding_invalid_type` (suspicious).
- From: [RCG G.ASY.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/async-await/G.ASY.02.html), [RCG G.ASY.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/async-await/G.ASY.03.html).

### CON-02
**Do not block an async executor thread: no `std::fs`, `std::thread::sleep`, blocking database or HTTP clients, or heavy computation in async code. Use the runtime's async API or move the work to a blocking pool.**
- Why: one blocked worker stalls every task scheduled on it.
- Check: `clippy::disallowed_methods` (style) with a project list (example in [lints/clippy.toml](lints/clippy.toml)); review.
- From: [RCG G.ASY.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/async-await/G.ASY.05.html), [RCG P.ASY.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/async-await/P.ASY.01.html).

### CON-03
**Make long CPU-bound async work yield regularly (Microsoft suggests every 10 to 100 µs of work), or move it off the async executor.**
- Why: cooperative scheduling starves other tasks when one task never yields.
- Check: review.
- From: [Microsoft M-YIELD-POINTS](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-YIELD-POINTS).

### CON-04
**Make a function `async` only if it awaits something. Write `async fn` instead of returning a hand-written `impl Future`, unless the signature requires it.**
- Why: needless `async` forces callers to await and adds state machines.
- Check: `clippy::unused_async` (pedantic), `clippy::manual_async_fn` (style).
- From: [RCG G.ASY.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/async-await/G.ASY.04.html), [Microsoft M-ASYNC-FN](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-ASYNC-FN).

### CON-05
**Await or explicitly spawn every future. Never let a future drop unpolled.**
- Why: futures are lazy; an unawaited future does nothing.
- Check: `unused_must_use` (rustc, warn by default), `clippy::let_underscore_future` (suspicious), `clippy::async_yields_async` (correctness).
- From: [RCG G.ASY.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/async-await/G.ASY.01.html).

### CON-06
**Keep lock scopes short. Do not call unknown code (callbacks, I/O, other locks) while holding a lock. When code needs several locks, take them in one fixed, documented order.**
- Why: long or nested lock scopes cause contention and deadlocks.
- Check: review.
- From: [RCG P.MTH.LCK.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/threads/lock/P.MTH.LCK.01.html), [Microsoft M-THROUGHPUT](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-THROUGHPUT).

### CON-07
**Handle a poisoned mutex deliberately: treat it as a bug (panic with a message) or return an error. Do not take the data out of a poisoned lock and continue.**
- Why: poisoning means another thread panicked in the middle of an update.
- Check: review.
- From: [Google: Handling poison](https://google.github.io/styleguide/rustguide.html#handling-poison).

### CON-08
**Use atomics for a single flag or counter, not `Mutex<bool>` or `Mutex<usize>`. Prefer locks over hand-written lock-free code, and justify every memory ordering weaker than `SeqCst` in a comment.**
- Why: atomics are cheaper for single values; lock-free code with wrong orderings is a silent data race.
- Check: strict tier `clippy::mutex_atomic` (restriction), `clippy::mutex_integer` (restriction); review for orderings.
- From: [RCG G.MTH.LCK.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/threads/lock/G.MTH.LCK.01.html), [RCG P.MTH.LKF.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/threads/lock-free/P.MTH.LKF.01.html), [RCG P.MTH.LKF.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/threads/lock-free/P.MTH.LKF.02.html).

### CON-09
**Use std or the async runtime's synchronization and channel types by default. Add `parking_lot`, `crossbeam` or similar only with a measured need.**
- Why: each dependency costs compile time and supply-chain review (see "Where the sources disagree").
- Check: review; `clippy::disallowed_types` (style) if a project bans a type.
- From: [RCG G.MTH.LCK.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/threads/lock/G.MTH.LCK.03.html) and [RCG G.MTH.LCK.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/threads/lock/G.MTH.LCK.04.html) (this rule takes the opposite side), [Google: Handling poison](https://google.github.io/styleguide/rustguide.html#handling-poison).

### CON-10
**Make public types and returned futures `Send` (and `Sync` where possible). Add a compile-time assertion for types whose thread safety is part of the contract.**
- Why: a non-`Send` future cannot be spawned on a multi-threaded runtime; regressions are silent without a test.
- Check: strict tier `clippy::future_not_send` (nursery); `clippy::arc_with_non_send_sync` (suspicious); the assertion test.
- From: [API C-SEND-SYNC](https://rust-lang.github.io/api-guidelines/interoperability.html#c-send-sync), [Microsoft M-TYPES-SEND](https://microsoft.github.io/rust-guidelines/guidelines/libs/interop/#M-TYPES-SEND).

```rust
#[test]
fn client_is_send_sync() {
    fn assert_send_sync<T: Send + Sync>() {}
    assert_send_sync::<Client>();
}
```

## 9. Arithmetic and conversions (NUM)

### NUM-01
**Convert numbers with `From`/`TryFrom` (`u64::from(x)`, `u32::try_from(n)?`). Use `as` only when truncation or wrapping is intended, and say so in a comment. Write byte literals as `b'a'`, not `'a' as u8`.**
- Why: `as` silently truncates, wraps and changes sign.
- Check: `clippy::char_lit_as_u8` (complexity); `trivial_numeric_casts` (rustc, allow by default); strict tier `clippy::cast_possible_truncation` (pedantic), `clippy::cast_sign_loss` (pedantic), `clippy::cast_possible_wrap` (pedantic), `clippy::cast_precision_loss` (pedantic), `clippy::cast_lossless` (pedantic), `clippy::checked_conversions` (pedantic).
- From: [RCG G.TYP.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/G.TYP.01.html), [RCG G.TYP.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/G.TYP.03.html), [RCG G.TYP.INT.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/int/G.TYP.INT.02.html), [RCG G.TYP.CHR.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/char/G.TYP.CHR.01.html).

### NUM-02
**For arithmetic on untrusted or unbounded values, choose the overflow behaviour explicitly: `checked_*`, `saturating_*`, `wrapping_*` or `overflowing_*`. Use `rem_euclid` when a remainder must be non-negative.**
- Why: plain operators panic on overflow in debug builds and wrap in release builds; `%` keeps the sign of the dividend.
- Check: `clippy::manual_saturating_arithmetic` (style); strict tier `clippy::arithmetic_side_effects` (restriction) (enable per crate that parses external input), `clippy::modulo_arithmetic` (restriction).
- From: [RCG G.TYP.INT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/int/G.TYP.INT.01.html), [RCG G.TYP.INT.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/int/G.TYP.INT.03.html).

### NUM-03
**Iterate instead of indexing where possible. Index with `.get()` when the index comes from input; use `[]` only where the bound is proven.**
- Why: an out-of-bounds index is a panic.
- Check: `clippy::needless_range_loop` (style), `clippy::out_of_bounds_indexing` (correctness); strict tier `clippy::indexing_slicing` (restriction) (per crate that handles external input).
- From: [RCG P.TYP.SLC.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/slice/P.TYP.SLC.01.html), [RCG G.TYP.ARR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/array/G.TYP.ARR.02.html).

### NUM-04
**Slice strings only at byte offsets known to be character boundaries (from `find`, `char_indices`); otherwise use `str::get(range)`.**
- Why: slicing inside a multi-byte character panics.
- Check: strict tier `clippy::string_slice` (restriction).
- From: [RCG G.STR.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/strings/G.STR.05.html).

### NUM-05
**Do not compare floats with `==`; compare within a tolerance that fits the domain, or use `total_cmp` for ordering.**
- Why: rounding makes exact equality unreliable.
- Check: strict tier `clippy::float_cmp` (pedantic).
- From: [RCG G.TYP.FLT.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/float/G.TYP.FLT.03.html).

### NUM-06
**Write the type explicitly where inference crosses `.into()`, `.parse()`, `.collect()` or `Default::default()` and the target type is not obvious from nearby code.**
- Why: the reader should not have to run type inference in their head.
- Check: review.
- From: [Google: Don't overuse type inference](https://google.github.io/styleguide/rustguide.html#dont-overuse-type-inference).

## 10. Macros (MAC)

### MAC-01
**Use a function, generic or trait first; a `macro_rules!` macro second; a procedural macro last.**
- Why: macros are harder to read, to debug and to refactor, and tools understand them poorly; proc macros also cost compile time (CMP-05).
- Check: review.
- From: [Google: Macros](https://google.github.io/styleguide/rustguide.html#macros), [Microsoft M-MACRO-LAST-RESORT](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-MACRO-LAST-RESORT), [Microsoft M-EXAMPLE-OVER-PROC](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-EXAMPLE-OVER-PROC), [RCG P.MAC.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/P.MAC.01.html).

### MAC-02
**Make macro input look like the Rust it produces, and never hide changes to the shape of an item (an added `async`, a changed return type, extra generated items).**
- Why: readers must be able to predict the expansion.
- Check: review.
- From: [API C-EVOCATIVE](https://rust-lang.github.io/api-guidelines/macros.html#c-evocative), [Microsoft M-MACROS-DONT-LIE](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-MACROS-DONT-LIE), [Microsoft M-PROC-IMPLIED-ITEMS](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-PROC-IMPLIED-ITEMS), [RCG P.MAC.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/P.MAC.02.html).

### MAC-03
**In exported `macro_rules!` macros, refer to items through `$crate::`, put helpers in a `#[doc(hidden)]` private module, accept attributes and visibility on generated items, and order arms from most specific to most general.**
- Why: the macro must work wherever it is invoked, and the first matching arm wins.
- Check: review.
- From: [API C-MACRO-ATTR](https://rust-lang.github.io/api-guidelines/macros.html#c-macro-attr), [API C-MACRO-VIS](https://rust-lang.github.io/api-guidelines/macros.html#c-macro-vis), [Microsoft M-MACRO-HELPERS](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-MACRO-HELPERS), [Microsoft M-MACRO-MAIN-CRATE](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-MACRO-MAIN-CRATE), [RCG P.MAC.DCL.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/decl/P.MAC.DCL.02.html).

### MAC-04
**Write a proc macro as a thin `foo_proc` crate that calls a normal, testable `foo_proc_impl` library. Test the expansions, and report errors as compile errors with correct spans, not panics.**
- Why: the logic becomes unit-testable and users get errors that point at their code.
- Check: review; snapshot or compile-fail tests.
- From: [Microsoft M-PROC-IMPL](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-PROC-IMPL), [RCG P.MAC.PRO.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/proc/P.MAC.PRO.02.html), [RCG P.MAC.PRO.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/proc/P.MAC.PRO.04.html).

### MAC-05
**Pin a companion proc-macro crate to an exact version (`=x.y.z`) from its main crate.** [lib]
- Why: the two crates share private interfaces that must change together.
- Check: review.
- From: [Microsoft M-MACRO-VERSION-PIN](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-MACRO-VERSION-PIN).

## 11. Dependencies and features (DEP)

### DEP-01
**Add a dependency only when it removes significant code or risk. First check that std or an existing dependency does not already cover it, that it is maintained, and that its licence fits the project.**
- Why: every dependency adds compile time, supply-chain risk and code that runs at build time (build scripts, proc macros).
- Check: review; `cargo tree`.
- From: [RCG P.SEC.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/security/P.SEC.01.html), [API C-PERMISSIVE](https://rust-lang.github.io/api-guidelines/necessities.html#c-permissive).

### DEP-02
**Declare dependency versions once in `[workspace.dependencies]` and use `name.workspace = true` in members. Never use wildcard (`*`) versions.**
- Why: one version per dependency avoids duplicate builds and inconsistent behaviour.
- Check: `clippy::wildcard_dependencies` (cargo).
- From: [Microsoft M-CARGO-WORKSPACE](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-CARGO-WORKSPACE), [RCG G.CAR.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/G.CAR.04.html).

### DEP-03
**Declare workspace dependencies with `default-features = false`, and enable in each member only the features it uses.**
- Why: default features pull in code and transitive crates that nobody uses (CMP-10).
- Check: review; `cargo tree -e features`.
- From: [Microsoft M-CARGO-WORKSPACE](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-CARGO-WORKSPACE), [Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html).

### DEP-04
**Keep features additive: enabling a feature must not remove items or change existing behaviour incompatibly, and every combination must compile.**
- Why: Cargo unifies features across the dependency graph, so any crate can turn a feature on for everyone.
- Check: `cargo hack check --feature-powerset` (tool).
- From: [Microsoft M-FEATURES-ADDITIVE](https://microsoft.github.io/rust-guidelines/guidelines/libs/building/#M-FEATURES-ADDITIVE), [RCG P.CAR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/P.CAR.02.html).

### DEP-05
**Use Cargo features, not custom `--cfg` flags, for optional functionality. Prefer `if cfg!(...)` to `#[cfg(...)]` when both branches compile on every target.**
- Why: features fit Cargo's resolver; `cfg!` keeps both branches type-checked.
- Check: review.
- From: [RCG P.CAR.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/P.CAR.03.html), [RCG P.CAR.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/P.CAR.04.html).

### DEP-06
**Remove unused dependencies and unused dependency features when you touch a manifest.**
- Why: unused dependencies still compile and link.
- Check: `cargo udeps` or `cargo machete` (tools); Cargo's own check needs nightly (`-Zcargo-lints`).
- From: [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION), [Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html).

### DEP-07
**Check dependencies for security advisories, licences and duplicate versions before release.**
- Why: vulnerable or incompatible crates ship with your binary.
- Check: `cargo audit` (tool, named by Microsoft and RCG); `cargo deny` (tool, not named by the sources); strict tier `clippy::multiple_crate_versions` (cargo).
- From: [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION), [RCG P.SEC.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/security/P.SEC.01.html), [API C-PERMISSIVE](https://rust-lang.github.io/api-guidelines/necessities.html#c-permissive).

### DEP-08
**Start new crates on the latest stable edition. Set `rust-version` (MSRV) and raise it deliberately, a few releases behind the current stable.**
- Why: Clippy checks API use against the MSRV, and users on older toolchains get a clear error.
- Check: `clippy::incompatible_msrv` (suspicious).
- From: [Microsoft M-LATEST-EDITION](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-LATEST-EDITION), [Microsoft M-MSRV](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-MSRV).

### DEP-09
**Make library crates build with plain `cargo build`: no external tools, environment variables or network access at build time. A `-sys` crate builds bundled C sources with `cc` and ships pre-generated bindings.** [lib]
- Why: every user and CI machine must be able to build it.
- Check: build in a clean container.
- From: [Microsoft M-OOBE](https://microsoft.github.io/rust-guidelines/guidelines/libs/building/#M-OOBE), [Microsoft M-SYS-CRATES](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-SYS-CRATES).

### DEP-10
**Give published crates complete `[package]` metadata: `description`, `license` (an SPDX expression), `repository`, `keywords`, `categories`.** [lib]
- Why: registries, licence scanners and users read these fields.
- Check: strict tier `clippy::cargo_common_metadata` (cargo).
- From: [API C-METADATA](https://rust-lang.github.io/api-guidelines/documentation.html#c-metadata), [Style Guide: Cargo.toml](https://doc.rust-lang.org/style-guide/cargo.html), [RCG G.CAR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/G.CAR.02.html).

## 12. Documentation and comments (DOC)

### DOC-01
**Document every public item.** [lib]
- Why: the docs are the contract users and agents code against.
- Check: `missing_docs` (rustc, allow by default) (strict tier of the workspace table, or `#![warn(missing_docs)]` in each library crate).
- From: [Google: documentation](https://google.github.io/styleguide/rustguide.html#when-and-how-to-write-documentation), [Microsoft M-CANONICAL-DOCS](https://microsoft.github.io/rust-guidelines/guidelines/docs/#M-CANONICAL-DOCS), [API C-CRATE-DOC](https://rust-lang.github.io/api-guidelines/documentation.html#c-crate-doc).

### DOC-02
**Start each doc comment with a one-line summary sentence of about 15 words, followed by a blank line.**
- Why: rustdoc shows the first paragraph in item lists and search results.
- Check: strict tier `clippy::too_long_first_doc_paragraph` (nursery).
- From: [Microsoft M-FIRST-DOC-SENTENCE](https://microsoft.github.io/rust-guidelines/guidelines/docs/#M-FIRST-DOC-SENTENCE).

### DOC-03
**Add the canonical sections where they apply: `# Examples`, `# Errors` (for `Result`), `# Panics` (if it can panic), `# Safety` (for `unsafe`). Do not add parameter tables.**
- Why: callers look for these headings to learn the failure contract.
- Check: `clippy::missing_safety_doc` (style); strict tier `clippy::missing_errors_doc` (pedantic), `clippy::missing_panics_doc` (pedantic).
- From: [Microsoft M-CANONICAL-DOCS](https://microsoft.github.io/rust-guidelines/guidelines/docs/#M-CANONICAL-DOCS), [API C-FAILURE](https://rust-lang.github.io/api-guidelines/documentation.html#c-failure), [RCG G.CMT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/comments/G.CMT.01.html), [RCG G.CMT.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/comments/G.CMT.02.html).

### DOC-04
**Write doc examples that compile and run, and that use `?` instead of `unwrap`.** [lib]
- Why: users copy examples verbatim; doctests keep them correct.
- Check: `cargo test --doc`.
- From: [API C-EXAMPLE](https://rust-lang.github.io/api-guidelines/documentation.html#c-example), [API C-QUESTION-MARK](https://rust-lang.github.io/api-guidelines/documentation.html#c-question-mark).

### DOC-05
**Give each crate and public module a `//!` doc that says what it contains, when to use it, and shows an example.** [lib]
- Why: module docs are where readers learn how the items fit together.
- Check: review; rustdoc lint `rustdoc::missing_crate_level_docs` (allow by default).
- From: [Microsoft M-MODULE-DOCS](https://microsoft.github.io/rust-guidelines/guidelines/docs/#M-MODULE-DOCS), [API C-CRATE-DOC](https://rust-lang.github.io/api-guidelines/documentation.html#c-crate-doc).

### DOC-06
**Link referenced items with intra-doc links (``[`Item`]``), and keep `cargo doc` free of warnings.**
- Why: links let readers navigate and break loudly when an item is renamed.
- Check: `rustdoc::broken_intra_doc_links` (rustdoc, warn by default), `rustdoc::private_intra_doc_links` (rustdoc, warn by default).
- From: [API C-LINK](https://rust-lang.github.io/api-guidelines/documentation.html#c-link).

### DOC-07
**Write comments that explain why: constraints, invariants, non-obvious decisions. Do not restate the code, explain basic Rust, or narrate how the code was produced or changed ("changed X to Y", "added for step 3"). Use `///` for items, `//!` for modules and crates, `//` for implementation notes; avoid `/* */`.**
- Why: history belongs in version control; comments must stay true as the code changes.
- Check: review.
- From: [Google: documentation](https://google.github.io/styleguide/rustguide.html#when-and-how-to-write-documentation), [Microsoft M-NO-META-DESIGN-DOCUMENTATION](https://microsoft.github.io/rust-guidelines/guidelines/ai/#M-NO-META-DESIGN-DOCUMENTATION), [RCG P.CMT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/comments/P.CMT.01.html), [RCG P.CMT.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/code_style/comments/P.CMT.03.html), [Style Guide: comments](https://doc.rust-lang.org/style-guide/index.html).

### DOC-08
**Replace magic values with named constants, and document why the value was chosen.**
- Why: the reason for a number is lost once its author is gone.
- Check: review.
- From: [Microsoft M-DOCUMENTED-MAGIC](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-DOCUMENTED-MAGIC).

### DOC-09
**Mark your own `pub use` re-exports `#[doc(inline)]`. Mark items that must be `pub` only for technical reasons `#[doc(hidden)]`. Never use another crate's `#[doc(hidden)]` items.**
- Why: hidden items are not part of the other crate's stable API.
- Check: review.
- From: [Microsoft M-DOC-INLINE](https://microsoft.github.io/rust-guidelines/guidelines/docs/#M-DOC-INLINE), [API C-HIDDEN](https://rust-lang.github.io/api-guidelines/documentation.html#c-hidden), [Google: doc(hidden)](https://google.github.io/styleguide/rustguide.html#do-not-access-items-marked-dochidden-that-you-dont-own).

## 13. Testing (TST)

### TST-01
**Ship every behaviour change with a test that fails without the change.**
- Why: the test proves the change and guards it against regression.
- Check: review (run the test against the old code).
- From: [Microsoft M-DESIGN-FOR-AI](https://microsoft.github.io/rust-guidelines/guidelines/ai/#M-DESIGN-FOR-AI), [Microsoft M-TAUTOLOGICAL-TESTS](https://microsoft.github.io/rust-guidelines/guidelines/ai/#M-TAUTOLOGICAL-TESTS).

### TST-02
**Do not write tautological tests: no assertions of constants against themselves, and no tests that recompute the expected value with the code under test. Derive expected values independently.**
- Why: such tests pass whatever the code does.
- Check: review.
- From: [Microsoft M-TAUTOLOGICAL-TESTS](https://microsoft.github.io/rust-guidelines/guidelines/ai/#M-TAUTOLOGICAL-TESTS).

### TST-03
**Put unit tests in a `#[cfg(test)] mod tests` next to the code (move large ones to a separate file). Put tests that use only the public API under `tests/`.**
- Why: test code then stays out of normal builds, and public-API tests check what users see.
- Check: `clippy::items_after_test_module` (style).
- From: [Microsoft M-INTEGRATION-TESTS](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-INTEGRATION-TESTS), [RCG P.MOD.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/module/P.MOD.02.html).

### TST-04
**Make I/O, clocks, randomness and external services injectable so tests are deterministic. Do not use sleeps to synchronize tests.**
- Why: tests that depend on the real clock or network are flaky.
- Check: review.
- From: [Microsoft M-MOCKABLE-SYSCALLS](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-MOCKABLE-SYSCALLS).

### TST-05
**Gate test helpers and mocks behind `#[cfg(test)]` or a `test-util` feature, so production builds cannot reach them.**
- Why: a mock or a check bypass in production is a security hole.
- Check: review.
- From: [Microsoft M-TEST-UTIL](https://microsoft.github.io/rust-guidelines/guidelines/libs/resilience/#M-TEST-UTIL).

### TST-06
**Assert on values, not on `is_ok()`/`is_err()`: use `assert_eq!`, `assert!(matches!(...))` or `?` in tests that return `Result`.**
- Why: the failure message then shows the actual error or value.
- Check: strict tier `clippy::assertions_on_result_states` (restriction).
- From: [Microsoft M-STATIC-VERIFICATION](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-STATIC-VERIFICATION).

## 14. Performance (PRF)

### PRF-01
**Optimise only measured hot paths. Benchmark and profile before and after, and put the numbers in the change description.**
- Why: unmeasured optimisations add complexity without proven gain.
- Check: review; benchmark tools such as `criterion` or `divan`.
- From: [Microsoft M-HOTPATH](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-HOTPATH), [Google: Unsafe code](https://google.github.io/styleguide/rustguide.html#unsafe-code).

### PRF-02
**Preallocate with `with_capacity` when the size is known or bounded, and prefer `collect()` to push loops.**
- Why: avoids repeated reallocation and copying.
- Check: `clippy::vec_init_then_push` (perf).
- From: [Microsoft M-INITIAL-CAPACITY](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-INITIAL-CAPACITY), [RCG P.TYP.VEC.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/vec/P.TYP.VEC.02.html), [RCG P.STR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/strings/P.STR.02.html), [RCG P.CLT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/collections/P.CLT.01.html).

### PRF-03
**In repeated work, reuse buffers: let callers pass a `&mut Vec<T>` or `&mut String` that is cleared and refilled.**
- Why: allocation in a loop often dominates the cost.
- Check: review.
- From: [Microsoft M-MEM-REUSE](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-MEM-REUSE).

### PRF-04
**Store immutable owned sequences as `Box<[T]>`, `Box<str>`, `Arc<[T]>` or `Arc<str>`, and avoid stacked indirection (`Arc<Box<T>>`, `Box<Vec<T>>`). Shrink long-lived collections after building them. Prefer `Vec`/`VecDeque` to `LinkedList`, and `sort_unstable` for primitive elements.**
- Why: removes pointer hops and spare capacity; contiguous storage is faster.
- Check: `clippy::redundant_allocation` (perf); strict tier `clippy::rc_buffer` (restriction), `clippy::linkedlist` (pedantic), `clippy::stable_sort_primitive` (pedantic).
- From: [Microsoft M-BOX-DST](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-BOX-DST), [Microsoft M-AVOID-INDIRECTION](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-AVOID-INDIRECTION), [Microsoft M-SHRINK-TO-FIT](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-SHRINK-TO-FIT), [RCG G.MTH.LCK.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/threads/lock/G.MTH.LCK.02.html), [RCG G.CLT.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/collections/G.CLT.01.html), [RCG G.TYP.ARR.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/array/G.TYP.ARR.03.html).

### PRF-05
**Use a faster hasher (for example `foldhash` or `FxHash`) only for keys an attacker cannot choose; keep the std hasher for untrusted keys.**
- Why: the std hasher resists hash-flooding attacks; faster hashers do not.
- Check: review.
- From: [Microsoft M-FAST-HASHER](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-FAST-HASHER).

### PRF-06
**Keep large values off the stack: box big arrays and large futures, and put large constant tables in a `static`, not a `const`. Keep the enum variants of a type close in size.**
- Why: large stack values can overflow the stack and are expensive to move; a `const` is copied into every use.
- Check: `clippy::large_enum_variant` (perf), `clippy::large_const_arrays` (perf); strict tier `clippy::large_stack_arrays` (pedantic), `clippy::large_futures` (pedantic).
- From: [RCG G.VAR.04](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/variables/G.VAR.04.html), [RCG G.TYP.ARR.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/array/G.TYP.ARR.01.html), [RCG G.TYP.ENM.06](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/data-type/enum/G.TYP.ENM.06.html), [Microsoft M-ASYNC-STACK-SIZE](https://microsoft.github.io/rust-guidelines/guidelines/performance/#M-ASYNC-STACK-SIZE).

### PRF-07
**Do not add `#[inline(always)]` without a benchmark. Use `#[inline]` only on small functions that are called across crates on a hot path.**
- Why: forced inlining grows code and compile time in every caller.
- Check: strict tier `clippy::inline_always` (pedantic).
- From: [RCG G.FUD.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/fn-design/G.FUD.05.html).

### PRF-08
**Change the global allocator or `target-cpu` only for applications, only with a benchmark, and only when you control the deployment hardware.**
- Why: gains depend on the workload; a too-new `target-cpu` crashes on older machines.
- Check: review.
- From: [Microsoft M-MIMALLOC-APPS](https://microsoft.github.io/rust-guidelines/guidelines/apps/#M-MIMALLOC-APPS), [Microsoft M-TARGET-CPU](https://microsoft.github.io/rust-guidelines/guidelines/apps/#M-TARGET-CPU).

## 15. Compile time (CMP)

Code-level and workspace-level rules. Machine and tool setup is in the appendix.

**Generics and monomorphization**

### CMP-01
**Keep generic functions thin: convert generic arguments (`impl AsRef<Path>`, `impl Into<String>`) at the top, then call a non-generic inner function that holds the body.**
- Why: each distinct type argument compiles the whole generic body again, in every crate that uses it.
- Check: `cargo llvm-lines` (tool) shows the most-instantiated functions.
- From: [matklad: Fast Rust builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html), [Microsoft M-IMPL-ASREF](https://microsoft.github.io/rust-guidelines/guidelines/libs/interop/#M-IMPL-ASREF), [cargo-llvm-lines](https://github.com/dtolnay/cargo-llvm-lines).

```rust
pub fn load(path: impl AsRef<Path>) -> io::Result<Config> {
    fn inner(path: &Path) -> io::Result<Config> {
        let text = fs::read_to_string(path)?;
        parse(&text)
    }
    inner(path.as_ref())
}
```

### CMP-02
**Keep the number of generic parameters and trait bounds on widely used items small. Do not make a function generic for one or two caller types; application code should almost never take `impl Into<T>`.**
- Why: generic parameters multiply instantiations; RCG reports a large compile-time difference in its example.
- Check: review.
- From: [RCG P.GEN.03](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/generic/P.GEN.03.html), [Google: Accepting Into](https://google.github.io/styleguide/rustguide.html#accepting-into).

### CMP-03
**Use `&dyn Trait` or `Box<dyn Trait>` at crate boundaries and in code that is not a measured hot path, instead of a generic parameter.**
- Why: a `dyn` function compiles once in its own crate; a generic one compiles again in each user crate.
- Check: review.
- From: [matklad: Fast Rust builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html), [Google: Generics versus dyn Trait](https://google.github.io/styleguide/rustguide.html#generics-versus-dyn-trait), [Microsoft M-DI-HIERARCHY](https://microsoft.github.io/rust-guidelines/guidelines/libs/ux/#M-DI-HIERARCHY).

### CMP-04
**Measure before you restructure for compile time: use `cargo build --timings` for the crate graph and `cargo llvm-lines` for generic bloat.**
- Why: build costs differ widely between projects; intuition is often wrong.
- Check: the `--timings` report (stable Cargo option).
- From: [Cargo: Reporting build timings](https://doc.rust-lang.org/cargo/reference/timings.html), [Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html).

**Proc macros and derives**

### CMP-05
**Derive only the traits you use. Keep heavy derives (serialization, schema, ORM and binding generators) in the leaf crates that need them, and gate them in shared crates with `#[cfg_attr(feature = "serde", derive(...))]`. Before adding a proc-macro dependency, check whether `macro_rules!` or plain code does the job.**
- Why: derive expansion is not cached between builds, and proc-macro crates (and `syn`) must build before their users can start; `macro_rules!` compiles faster.
- Check: review; `cargo tree -e normal --invert <macro-crate>`.
- From: [Microsoft M-EXAMPLE-OVER-PROC](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-EXAMPLE-OVER-PROC), [Rust survey 2025](https://blog.rust-lang.org/2025/09/10/rust-compiler-performance-survey-2025-results), [matklad: Fast Rust builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html), [Nethercote, Dec 2025](https://nnethercote.github.io/2025/12/05/how-to-speed-up-the-rust-compiler-in-december-2025.html).

### CMP-06
**Keep macro expansions small: have the macro call a function that holds the bulk of the logic, and move rarely taken paths into a `#[cold]` function.**
- Why: the expansion is compiled at every call site.
- Check: review.
- From: [RCG G.MAC.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/G.MAC.02.html), [Microsoft M-MACRO-LAST-RESORT](https://microsoft.github.io/rust-guidelines/guidelines/macros/#M-MACRO-LAST-RESORT).

**Crate size and layering**

### CMP-07
**Split code into crates along parts that can be used on their own, and make the crate graph wide rather than deep: a small base crate of shared types, independent feature crates on top, and a thin binary that ties them together.**
- Why: crates are the unit of parallel and incremental compilation; a long chain compiles one crate at a time.
- Check: `cargo build --timings` concurrency graph.
- From: [Microsoft M-SMALLER-CRATES](https://microsoft.github.io/rust-guidelines/guidelines/universal/#M-SMALLER-CRATES), [RCG P.CAR.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/P.CAR.01.html), [matklad: Fast Rust builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html), [Feldera: one thousand crates](https://www.feldera.com/blog/cutting-down-rust-compile-times-from-30-to-2-minutes-with-one-thousand-crates).

### CMP-08
**Make dependencies between workspace crates point one way, and keep frequently edited code out of crates that many others depend on.**
- Why: a change to a crate rebuilds everything that depends on it.
- Check: review; `cargo tree --invert <crate>` lists everything that depends on a crate.
- From: [RCG P.CAR.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/P.CAR.01.html), [Rust survey 2025](https://blog.rust-lang.org/2025/09/10/rust-compiler-performance-survey-2025-results).

### CMP-09
**Keep the number of final artifacts small: fewer binaries, examples and test targets.**
- Why: each one links separately, and linking always runs from scratch.
- Check: review `[[bin]]`, `[[example]]`, `[[test]]` entries and `tests/*.rs` files.
- From: [matklad: Fast Rust builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html), [Rust survey 2025](https://blog.rust-lang.org/2025/09/10/rust-compiler-performance-survey-2025-results).

**Feature hygiene**

### CMP-10
**Turn off default features of dependencies and enable only what you use. Look for duplicate versions with `cargo tree --duplicates` and unused features with `cargo tree -e features`.**
- Why: unused features compile extra code and pull in extra crates.
- Check: the `cargo tree` commands; tools such as `cargo-features-manager`.
- From: [Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html), [Microsoft M-CARGO-WORKSPACE](https://microsoft.github.io/rust-guidelines/guidelines/project/#M-CARGO-WORKSPACE), [corrode: compile-time tips](https://corrode.dev/blog/tips-for-faster-rust-compile-times/).

### CMP-11
**Keep the feature sets of shared dependencies the same across workspace members, so that each dependency builds once. Add a feature only when a real user needs it.**
- Why: building different member sets with different features rebuilds the shared dependencies; each feature multiplies the build and test matrix.
- Check: review; measure first, then consider `cargo-hakari` or Cargo's nightly `feature-unification = "workspace"`.
- From: [RCG P.CAR.02](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/cargo/P.CAR.02.html), [Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html), [cargo-hakari](https://docs.rs/cargo-hakari/latest/cargo_hakari/about/index.html).

**Test binary layout**

### CMP-12
**Combine a crate's integration tests into one test binary: `tests/it/main.rs` that declares the other test files as modules, instead of many top-level `tests/*.rs` files.**
- Why: each file in `tests/` is a separate binary that must be linked.
- Check: review the `tests/` directory.
- From: [corrode: compile-time tips](https://corrode.dev/blog/tips-for-faster-rust-compile-times/), [matklad: Fast Rust builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html).

```text
tests/
  it/
    main.rs      // mod api; mod storage;
    api.rs
    storage.rs
```

### CMP-13
**When you run tests with `cargo nextest`, also run `cargo test --doc`.**
- Why: nextest does not run doctests.
- Check: the test command list.
- From: [nextest: Running tests](https://nexte.st/docs/running/).

**Build scripts**

### CMP-14
**Avoid `build.rs` unless the build needs it. When you use one, print `cargo::rerun-if-changed=` (or `rerun-if-env-changed=`; single colon `cargo:` before Cargo 1.77) for exactly its inputs, keep it free of warnings, and do no network access.**
- Why: without rerun instructions Cargo reruns the script whenever any file in the package changes; build scripts also run arbitrary code on every builder.
- Check: review.
- From: [Cargo: Build scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html), [RCG P.MAC.PRO.06](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/proc/P.MAC.PRO.06.html), [RCG P.SEC.01](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/security/P.SEC.01.html).

### CMP-15
**Do not regenerate large, rarely changing code in `build.rs` on every build; generate it once with a separate command (for example `cargo xtask codegen`), commit the output, and check in CI that it is up to date. Choose a build script or a proc macro by what the generator needs.**
- Why: a build script runs before its crate compiles, on every builder; committed output costs nothing to build.
- Check: review. Note: no source measured this; it is this document's position, based on the cited ones.
- From: [RCG P.MAC.PRO.05](https://rust-coding-guidelines.github.io/rust-coding-guidelines/safe-guides/coding_practice/macros/proc/P.MAC.PRO.05.html), [matklad: Large Rust workspaces](https://matklad.github.io/2021/08/22/large-rust-workspaces.html).

**Workspace profile**

### CMP-16
**In the workspace `Cargo.toml`, reduce dev debug info: `debug = "line-tables-only"` for workspace members and `debug = false` for dependencies, with an opt-in profile that restores full debug info.**
- Why: less debug info speeds up code generation and linking and shrinks `target/`; line tables keep panic backtraces useful.
- Check: review `[profile.dev]`.
- From: [Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html), [Kobzol: disable debuginfo](https://kobzol.github.io/rust/rustc/2025/05/20/disable-debuginfo-to-improve-rust-compile-times.html).

```toml
[profile.dev]
debug = "line-tables-only"

[profile.dev.package."*"]
debug = false

[profile.debugging]
inherits = "dev"
debug = true
```

---

## Appendix: machine and tool setup for faster builds

These settings live on the developer machine or in CI, not in the code. Measure each change on your own workflow first; the sources report gains on their own projects only.

- Use `cargo check -p <crate>` while editing, and scope `cargo test`/`cargo clippy` with `-p` and test filters. ([corrode](https://corrode.dev/blog/tips-for-faster-rust-compile-times/))
- Give rust-analyzer its own target directory (`rust-analyzer.cargo.targetDir`) so it does not block or invalidate command-line builds. ([rust-analyzer FAQ](https://rust-analyzer.github.io/book/faq.html))
- Keep `RUSTFLAGS` and environment variables the same for all commands; a change invalidates cached build results. Put shared flags in `.cargo/config.toml`. ([Cargo: Build cache](https://doc.rust-lang.org/cargo/reference/build-cache.html))
- Linker: on x86_64 Linux, `lld` is the default since Rust 1.90. On other platforms, measure before you switch linkers; the Cargo book notes the default is often fast enough. ([Rust blog: lld](https://blog.rust-lang.org/2025/09/01/rust-lld-on-1.90.0-stable/), [Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html))
- Nightly-only options (Cranelift backend, parallel front end `-Zthreads`, workspace feature unification) can help but are unstable; Cranelift may not support unwinding on all targets. ([Cargo: Build performance](https://doc.rust-lang.org/cargo/guide/build-performance.html))
- `sccache` shares compiled dependencies across checkouts, but does not cache incrementally compiled workspace crates, proc-macro crates or binaries, and its cache keys depend on paths. ([sccache: Rust](https://github.com/mozilla/sccache/blob/main/docs/Rust.md))
- Each git worktree or checkout should have its own target directory; a shared one serialises builds on Cargo's lock. ([rust-analyzer FAQ](https://rust-analyzer.github.io/book/faq.html))
- With several builds in parallel on one machine, lower the jobs per build (`CARGO_BUILD_JOBS` or `build.jobs`). ([Cargo: Configuration](https://doc.rust-lang.org/cargo/reference/config.html))
- macOS: security scanning of freshly built binaries slows build scripts and test launches; adding the terminal to Developer Tools removes it, at the cost of an OS security check. ([nextest: macOS](https://nexte.st/docs/installation/macos/), [Nethercote: faster builds on Mac](https://nnethercote.github.io/2025/09/04/faster-rust-builds-on-mac.html))
- Update the toolchain regularly; the compiler gets faster with most releases. ([Nethercote, Dec 2025](https://nnethercote.github.io/2025/12/05/how-to-speed-up-the-rust-compiler-in-december-2025.html))
