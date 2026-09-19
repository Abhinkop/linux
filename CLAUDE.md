@AGENTS.md

## Claude Code specifics

Use plan mode before starting a new phase (Phase 1, Phase 2, Phase 3) or
any change that touches Kconfig — write the plan, get it approved, then
execute. For a single small step within a phase that's already approved,
plan mode isn't necessary.

Prefer the `boot-verifier` subagent to check a change actually boots,
rather than reasoning about whether it should. Prefer the
`misra-reviewer` subagent after writing or editing userspace daemon C
code.
