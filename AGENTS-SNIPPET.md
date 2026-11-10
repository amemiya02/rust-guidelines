# AGENTS.md snippet

Paste the block below into your project's `AGENTS.md` (or equivalent agent instructions file). Adjust the first line if your project overrides some rules.

```markdown
## Rust code

Follow the Rust Guidelines: https://github.com/zichuanxu/rust-guidelines

- Before a Rust change, read "Priority when rules conflict" in GUIDELINES.md and the chapters that match the change.
- Before you finish, run every applicable line of CHECKLIST.md: `cargo fmt --all -- --check`, `cargo clippy -p <crate> --all-targets`, and the crate's tests.
- Rules in this file and in the project's lint configuration take precedence over the guideline.
- Do not weaken lint levels, clippy.toml, rustfmt.toml or CI checks to make a change pass. Suppress one finding with `#[expect(lint, reason = "...")]`.
- In reviews and summaries, cite rule IDs (for example `ERR-03 src/config.rs:42: unwrap on user input`), and list any rule you chose not to follow, with the reason.
```

To adopt the lint configuration as well, copy the files under `lints/` as described in the README ("Adopting the lint configuration").
