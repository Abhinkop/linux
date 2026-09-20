@AGENTS.md

## Claude Code specifics

Use plan mode before starting a new **stage** (see `../plan.md` — six stages,
not "phases") or any change that touches Kconfig: write the plan, get it
approved, then execute. For a single small step within a stage that's already
approved, plan mode isn't necessary.

Prefer the `boot-verifier` subagent to check a change actually boots, rather
than reasoning about whether it should. Prefer the `misra-reviewer` subagent
after writing or editing the userspace-side C code (the stage-4 compatibility
library and driver hosts).

Remember that nothing builds on the host — every build and boot goes through
`../scripts/dev-container.sh`. See AGENTS.md.
