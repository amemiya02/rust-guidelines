# Rust change checklist

Run this before you finish a Rust change. Skip a line only if it cannot apply to the change. Fix every failure, or name the rule ID and the reason in your summary. Rule IDs refer to [GUIDELINES.md](GUIDELINES.md).

## Tools (run on the crates you changed)

- [ ] `cargo fmt --all -- --check` passes. (FMT-01, FMT-02)
- [ ] `cargo clippy -p <crate> --all-targets` shows no warnings in changed code. (FMT-03)
- [ ] `cargo test -p <crate>` passes; with nextest, also `cargo test --doc`. (TST-01, CMP-13)
- [ ] No lint level, `clippy.toml`, `rustfmt.toml` or CI step was weakened; each new suppression is `#[expect(..., reason = "...")]`. (FMT-04, FMT-05)

## Errors and panics

- [ ] No `unwrap()` outside tests; each `expect` states an invariant. (ERR-03, ERR-04)
- [ ] Expected failures return `Result`; panics only mark bugs and carry a message with values. (ERR-01, ERR-02, ERR-05)
- [ ] No `Result` dropped silently (`let _ =`, `.ok();`) without a comment. (ERR-07)
- [ ] Errors keep their source; library errors implement `Error + Send + Sync`. (ERR-06, ERR-08)
- [ ] No `todo!`, `unimplemented!`, `dbg!`, or stray `println!` for diagnostics. (ERR-12, ERR-13)

## Types, names and API

- [ ] New names follow std casing and conventions (`as_`/`to_`/`into_`, no `get_`, `new`/`try_new`). (NAM-01 to NAM-04)
- [ ] No filler names (`Manager`, `Helper`, `Util`); no module name repeated in item names. (NAM-06)
- [ ] Distinct values use newtypes with fallible constructors; unclear `bool` parameters became enums. (TYP-01 to TYP-03)
- [ ] Public types implement `Debug`; types with secrets redact it and have a test. (TYP-06, TYP-07)
- [ ] Visibility is minimal; each public item has one path; no glob imports or re-exports. (MOD-01 to MOD-03)
- [ ] Parameters borrow the most general form (`&str`, `&[T]`); no clone to dodge the borrow checker. (OWN-01 to OWN-03)

## Conversions and arithmetic

- [ ] Numeric conversions use `From`/`TryFrom`; every lossy `as` has a comment. (NUM-01)
- [ ] Arithmetic and indexing on external input use checked forms or `.get()`. (NUM-02, NUM-03)

## Unsafe (only if the change touches `unsafe`)

- [ ] The `unsafe` is necessary, and each block has a `// SAFETY:` comment. (UNS-01, UNS-02)
- [ ] Each `unsafe fn` has a `# Safety` section; the safe wrapper is sound for all inputs. (UNS-03, UNS-05)
- [ ] The tests pass under Miri. (UNS-06)

## Concurrency and async (only if the change touches threads or async)

- [ ] No `std::sync` guard or `RefCell` borrow is held across `.await`. (CON-01)
- [ ] No blocking call runs on the async executor; long CPU work yields or moves off it. (CON-02, CON-03)
- [ ] Every future is awaited or spawned; `async fn` only where something is awaited. (CON-04, CON-05)
- [ ] Lock scopes are short, with no callbacks or I/O under a lock. (CON-06)

## Dependencies and build (only if `Cargo.toml` or `build.rs` changed)

- [ ] A new dependency is justified, declared in `[workspace.dependencies]`, with `default-features = false` and only needed features. (DEP-01 to DEP-03, CMP-10)
- [ ] Features stay additive; no unused dependency remains. (DEP-04, DEP-06)
- [ ] Heavy derives and proc macros stay in leaf crates; no new proc macro where `macro_rules!` works. (CMP-05)
- [ ] New crate edges point one way; no new binary or test target without need. (CMP-08, CMP-09)
- [ ] Integration tests go into the existing single test binary. (CMP-12)
- [ ] A build script prints exact `rerun-if-changed` lines and does no network access. (CMP-14)

## Compile time (code)

- [ ] New generic functions are thin wrappers over a non-generic body; `dyn` used where speed is not measured. (CMP-01 to CMP-03)

## Tests and docs

- [ ] A test fails without the change; it is not tautological; I/O and clocks are injected. (TST-01, TST-02, TST-04)
- [ ] New public items have docs with a short first line and `# Errors`/`# Panics`/`# Safety` as needed. (DOC-01 to DOC-03)
- [ ] Comments explain why; none narrates the change history. (DOC-07)
