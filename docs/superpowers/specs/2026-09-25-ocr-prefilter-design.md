# open-code-review pre-filter for the CR gate — design

**Date**: 2026-09-25
**Status**: approved for implementation planning
**Owner**: Pezzutti Group (this fork), alberto.camilli@pezzutti.it

## Intent

Reduce the token cost of `execute-round`'s inline CR step by giving the
`cr-reviewer` agent a cheap, deterministic pre-filter pass before it does its
own full review. [open-code-review](https://github.com/alibaba/open-code-review)
(`ocr`) reads a diff and produces structured, line-level findings using
~1/9 the tokens of a generic coding agent, because file selection and rule
matching are deterministic — only the semantic judgment goes to an LLM.

Success looks like: `cr-reviewer` receives ocr's candidate findings as
context and spends its own reasoning verifying/extending them against the
BA design, instead of re-deriving everything from a blank diff. If `ocr`
is unavailable, the round must behave exactly as it does today — this is
an accelerator, never a dependency.

This becomes a standing rule for every arcgentic round run from this fork,
not a one-off for a single project.

## Why a fork

`execute-round`'s CR dispatch is not Claude Code's own sub-agent
resolution — `ClaudeCodeAdapter.dispatch_agent` (`toolkit/src/arcgentic/
adapters/claude_code.py:50-90`) spawns a plain `claude -p "Acting as the
cr-reviewer agent:\n\n{brief}"` subprocess. `{brief}` is assembled as a
plain Python f-string in `_phase_dev_body`
(`toolkit/src/arcgentic/skills_impl/execute_round.py:506-512`). There is no
project-level override point (`.claude/agents/` shadowing does not apply
to this dispatch path) and no plugin-documented hook or env-var
convention for injecting extra context. The only way to add a pre-filter
pass is to change what the brief string contains, which means changing
this function.

Upstream is `Arch1eSUN/Arcgentic`, MIT-licensed. This fork
(`Pezzutti-Group-S-p-A/arcgentic-fork`) carries one isolated patch and
periodically merges upstream `main`.

## Patch scope

One change, in `toolkit/src/arcgentic/skills_impl/execute_round.py`,
inside `_phase_dev_body`, before `cr_brief` is built:

- If env var `ARCGENTIC_OCR_PREFILTER=1` is set, run `ocr scan` against
  the dev-body diff (base SHA → head SHA) via `adapter.shell(...)`,
  capturing stdout as JSON.
- On success, append a new `OCR PRE-FILTER FINDINGS (verify, do not
  re-derive independently):` section to `cr_brief`, followed by the raw
  JSON.
- On any failure — `ocr` not on PATH, non-zero exit, timeout, malformed
  JSON — swallow the error, leave `cr_brief` exactly as it is today, and
  append a one-line entry to the `ExecuteRoundResult.warnings` list (e.g.
  `"ocr pre-filter skipped: <reason>"`) so it surfaces in the round
  summary without failing the phase.
- If the env var is unset, the code path is untouched — this is strictly
  additive and defaults to off until turned on deliberately per machine.

No other function, phase, or agent brief changes. `se-contract`'s
CONTRACT-ONLY isolation (mandate #20) is unaffected — the patch only
touches the CR brief construction.

To make the pre-filter actually reduce `cr-reviewer`'s own work (not just
add input), `agents/cr-reviewer.md` gets one additive paragraph: when an
`OCR PRE-FILTER FINDINGS` section is present in the brief, verify each
listed finding against the diff and BA design (confirm, adjust severity,
or reject) rather than performing an independent from-scratch scan;
still self-enforce the existing 4-dimensional coverage and 3-7 finding
count. This is the only change to agent instructions.

## ocr installation and credentials

- Installed once, globally, on any machine that runs arcgentic rounds:
  `npm install -g @alibaba-group/open-code-review`.
- One shared config file (provider: Anthropic, model: Claude Haiku,
  rules) — not per-repo. Haiku is chosen over Sonnet to maximize the
  net token saving; the pre-filter only needs to catch candidate issues,
  not adjudicate them.
- API key: `pzt-secret run --env ANTHROPIC_API_KEY=<secret-name> -- ocr
  scan ...`. Never in the shared config file, never in `execute_round.py`,
  never logged. If the secret does not yet exist in `kv-pzt-agent`, its
  name (`anthropic-ocr-prefilter`, tags `owner=alberto.camilli`,
  `rotation`, `source=anthropic-console`) is created by Alberto via
  `pzt-secret put`, per the credentials rule in the global CLAUDE.md —
  this session never requests or sees the value.

## Distribution and upkeep

- Every machine that runs `execute-round` switches from `pipx install
  arcgentic` to `pipx install git+https://github.com/Pezzutti-Group-S-p-A/
  arcgentic-fork.git`.
- Upstream sync: `git fetch upstream && git merge upstream/main`,
  periodically (no fixed cadence yet — triggered by upstream releases
  worth pulling in). The patch is ~20-30 lines confined to one function
  plus one paragraph in one agent file, so merge conflicts are expected
  to be rare and small.
- `ARCGENTIC_OCR_PREFILTER` defaults unset (off). Turned on per machine
  once `ocr` is installed and the credential is live, so rollout is
  incremental and never breaks a round that hasn't been set up yet.

## Testing

- Unit test for the new helper (`_run_ocr_prefilter` or similar): mocked
  `adapter.shell` returning success JSON, non-zero exit, and missing
  binary — asserts `cr_brief` content and `warnings` list in each case.
- Existing `toolkit/tests/unit/skills_impl/test_execute_round.py` cases
  must keep passing unmodified with the env var unset (default-off
  behavior verified).
- One dry-run round exercised locally with the env var set and a real
  `ocr` install, to confirm the appended section reaches the spawned
  `claude -p` process and cr-reviewer's output references it.

## Out of scope

- Laya Agent Kit (typed decision engine) — considered and explicitly
  deferred; different problem (classification/routing, not diff review),
  and its base checkpoints are near-chance without domain fine-tuning.
- CI/CD gate integration for repos with existing pipelines — a separate,
  independent use of `ocr` that doesn't touch this fork; not part of this
  round.
- Automatic upstream-merge tooling (e.g. a bot) — manual merges are
  sufficient at the current patch size.
