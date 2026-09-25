# ocr Pre-Filter for the CR Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give `execute-round`'s inline CR step a deterministic, cheap pre-filter (`open-code-review`, aka `ocr`) so `cr-reviewer` verifies candidate findings instead of scanning the diff from a blank slate — opt-in, never blocking, byte-identical behavior when disabled.

**Architecture:** One new pure helper (`_run_ocr_prefilter`) in `execute_round.py` runs `ocr review` via the existing `IDEAdapter.shell` abstraction, gated by env var `ARCGENTIC_OCR_PREFILTER`. Its output is appended as a labeled section to the existing `cr_brief` f-string inside `_phase_dev_body`, immediately before the `cr-reviewer` dispatch. Any failure (missing binary, non-zero exit, empty output) degrades to a warning string threaded back through `_phase_dev_body`'s return tuple into `ExecuteRoundResult.warnings` — it never raises `ExecuteRoundError`. `agents/cr-reviewer.md` gets one additive paragraph telling the agent how to use the section when present.

**Tech Stack:** Python 3.13+, pytest, the existing `InlineAdapter`/`_MultiStubAdapter` test-double pattern already used in `test_execute_round.py`.

**Spec:** `docs/superpowers/specs/2026-09-25-ocr-prefilter-design.md`

## Global Constraints

- `ARCGENTIC_OCR_PREFILTER` defaults unset (off); only the literal string `"1"` enables the pre-filter — any other value (`"true"`, `"yes"`, `""`) leaves it disabled.
- The patch touches exactly two files: `toolkit/src/arcgentic/skills_impl/execute_round.py` and `agents/cr-reviewer.md`. No other phase, function, or agent brief changes.
- The `se-contract` CONTRACT-ONLY isolation (mandate #20) is untouched — the pre-filter only feeds the CR brief.
- ocr failure of any kind (binary missing, non-zero exit, empty output, timeout) must degrade to a one-line entry in `ExecuteRoundResult.warnings` and never raise `ExecuteRoundError` — a broken or unconfigured `ocr` install must never block a round.
- No credential ever appears in `execute_round.py`, `cr-reviewer.md`, or any committed config — the ocr Anthropic API key is supplied by the environment via `pzt-secret run` outside this codebase entirely; this plan does not add any code that reads or stores a secret value.

## Review Focus

- **Env var set to a near-miss value** (`"true"`, `"yes"`, `""`, `"0"`) — must behave exactly like unset (disabled), not accidentally enable on any truthy-looking string. Pinned in Task 1.
- **`ocr` binary missing or crashing** (non-zero exit from `adapter.shell`) — must degrade to a warning and leave `cr_brief` unchanged, never raise. Pinned in Task 1 and Task 2.
- **`ocr` exits 0 but prints nothing** (misconfigured provider, empty diff it silently skips) — must be treated as "unavailable" (warning), not silently accepted as "zero findings work fine." Pinned in Task 1.
- **The pre-filter's output must actually reach the dispatched `cr-reviewer` prompt** — computing the section and discarding it (e.g. a wiring bug) would defeat the entire point of the patch. Pinned in Task 2 via a prompt-capturing test double.
- **Default (var unset) behavior must be byte-identical to pre-patch behavior** — every machine that hasn't opted in yet must see zero change: same `cr_brief` content, empty `warnings`. Pinned in Task 2.

---

### Task 1: `_run_ocr_prefilter` helper

**Files:**
- Modify: `toolkit/src/arcgentic/skills_impl/execute_round.py:16` (imports), `toolkit/src/arcgentic/skills_impl/execute_round.py:202` (insert new function after `_run_quality_gates`)
- Test: `toolkit/tests/unit/skills_impl/test_execute_round.py:24-38` (imports), append new tests near the end of the file

**Interfaces:**
- Consumes: `IDEAdapter.shell(command: str, timeout_seconds: int = 120) -> tuple[str, int]` (existing, `toolkit/src/arcgentic/adapters/base.py:92`); `shquote` from `arcgentic.adapters._local_env` (already imported at `execute_round.py:25`)
- Produces: `_run_ocr_prefilter(adapter: IDEAdapter, repo_root: Path) -> tuple[str, str | None]` — `(section_text, warning)`. Task 2 calls this and appends `section_text` to `cr_brief`, and threads `warning` into `ExecuteRoundResult.warnings`.

- [ ] **Step 1: Write the failing tests**

Open `toolkit/tests/unit/skills_impl/test_execute_round.py`. In the import block (lines 24-38), add `_run_ocr_prefilter` (alphabetically before `_run_quality_gates`):

```python
from arcgentic.skills_impl.execute_round import (
    ExecuteRoundError,
    ExecuteRoundResult,
    PhaseResult,
    _audit_handoff_path,
    _ba_design_path,
    _compose_self_audit_skeleton,
    _extract_ba_brief_from_handoff,
    _extract_se_threat_surfaces,
    _phase_dev_body,
    _phase_entry_admin,
    _round_to_upper,
    _run_ocr_prefilter,
    _run_quality_gates,
    run,
)
```

Append these tests at the end of the file:

```python
# ---------------------------------------------------------------------------
# ocr pre-filter helper
# ---------------------------------------------------------------------------


def test_ocr_prefilter_disabled_by_default(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """When ARCGENTIC_OCR_PREFILTER is unset, the pre-filter is a no-op."""
    monkeypatch.delenv("ARCGENTIC_OCR_PREFILTER", raising=False)
    stub = _MultiStubAdapter(canned_outputs={})
    section, warning = _run_ocr_prefilter(stub, tmp_path)
    assert section == ""
    assert warning is None


def test_ocr_prefilter_ignores_near_miss_env_values(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """Only the literal string '1' enables the pre-filter — 'true' must not."""
    monkeypatch.setenv("ARCGENTIC_OCR_PREFILTER", "true")
    stub = _MultiStubAdapter(
        canned_outputs={},
        shell_overrides={"ocr review": ("should not be seen", 0)},
    )
    section, warning = _run_ocr_prefilter(stub, tmp_path)
    assert section == ""
    assert warning is None


def test_ocr_prefilter_appends_findings_when_enabled(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """When enabled and ocr succeeds, its output is wrapped in a labeled section."""
    monkeypatch.setenv("ARCGENTIC_OCR_PREFILTER", "1")
    stub = _MultiStubAdapter(
        canned_outputs={},
        shell_overrides={"ocr review": ("file.py:12 possible off-by-one", 0)},
    )
    section, warning = _run_ocr_prefilter(stub, tmp_path)
    assert "OCR PRE-FILTER FINDINGS" in section
    assert "file.py:12 possible off-by-one" in section
    assert warning is None


def test_ocr_prefilter_warns_on_nonzero_exit(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """A non-zero ocr exit code is a soft warning, not a raised error."""
    monkeypatch.setenv("ARCGENTIC_OCR_PREFILTER", "1")
    stub = _MultiStubAdapter(
        canned_outputs={},
        shell_overrides={"ocr review": ("ocr: command not found", 127)},
    )
    section, warning = _run_ocr_prefilter(stub, tmp_path)
    assert section == ""
    assert warning is not None
    assert "127" in warning


def test_ocr_prefilter_warns_on_empty_output(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """A zero exit code with no output is treated as unavailable, not as zero findings."""
    monkeypatch.setenv("ARCGENTIC_OCR_PREFILTER", "1")
    stub = _MultiStubAdapter(canned_outputs={}, shell_overrides={"ocr review": ("", 0)})
    section, warning = _run_ocr_prefilter(stub, tmp_path)
    assert section == ""
    assert warning is not None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd toolkit && pytest tests/unit/skills_impl/test_execute_round.py -k ocr_prefilter -v`
Expected: FAIL with `ImportError: cannot import name '_run_ocr_prefilter'`

- [ ] **Step 3: Implement the helper**

In `toolkit/src/arcgentic/skills_impl/execute_round.py`, add `import os` to the import block at the top of the file (line 16, alongside the existing `import re`):

```python
import os
import re
```

Then insert the new function immediately after `_run_quality_gates` (after its closing `return results` at line 202, before the `_compose_self_audit_skeleton` def):

```python
def _run_ocr_prefilter(adapter: IDEAdapter, repo_root: Path) -> tuple[str, str | None]:
    """Run the open-code-review pre-filter on the workspace diff, if enabled.

    Controlled by ARCGENTIC_OCR_PREFILTER (opt-in; only the literal "1" enables
    it — unset or any other value leaves this a no-op). Never raises: any
    failure degrades to a warning and an empty section, so callers can fall
    back to today's cr_brief construction unchanged.

    Returns (section_text, warning):
    - section_text: "" when disabled/unavailable/failed, otherwise a labeled
      block ready to append to the cr-reviewer brief.
    - warning: None on success or when disabled; a one-line reason otherwise.
    """
    if os.environ.get("ARCGENTIC_OCR_PREFILTER") != "1":
        return "", None

    rr = shquote(str(repo_root))
    stdout, code = adapter.shell(f"cd {rr} && ocr review", timeout_seconds=120)
    if code != 0:
        return "", f"ocr pre-filter skipped: ocr review exited {code}"
    findings = stdout.strip()
    if not findings:
        return "", "ocr pre-filter skipped: ocr review produced no output"
    section = (
        "\n\nOCR PRE-FILTER FINDINGS (verify each against the diff and BA "
        "design; do not re-derive independently):\n\n" + findings
    )
    return section, None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd toolkit && pytest tests/unit/skills_impl/test_execute_round.py -k ocr_prefilter -v`
Expected: 5 passed

- [ ] **Step 5: Run the full existing suite to confirm no regression**

Run: `cd toolkit && pytest tests/unit/skills_impl/test_execute_round.py -v`
Expected: all tests pass, including the pre-existing ones (they don't reference `_run_ocr_prefilter` and the env var is unset in CI, so `_phase_dev_body`/`run` behavior is untouched by this task — Task 2 wires the call site).

- [ ] **Step 6: Commit**

```bash
git add toolkit/src/arcgentic/skills_impl/execute_round.py toolkit/tests/unit/skills_impl/test_execute_round.py
git commit -m "feat(execute-round): add opt-in ocr pre-filter helper"
```

---

### Task 2: Wire the pre-filter into the CR dispatch

**Files:**
- Modify: `toolkit/src/arcgentic/skills_impl/execute_round.py:463-586` (`_phase_dev_body` signature, body, both return statements) and `:700-704` (`run()`'s Phase 3 call site)
- Test: `toolkit/tests/unit/skills_impl/test_execute_round.py` (`_MultiStubAdapter.dispatch_agent`, append new tests)

**Interfaces:**
- Consumes: `_run_ocr_prefilter(adapter, repo_root) -> tuple[str, str | None]` (Task 1)
- Produces: `_phase_dev_body(...) -> tuple[PhaseResult, dict[str, str], int, int, str, str, str | None]` (7th element `ocr_warning`, was 6-tuple before). `run()` appends `ocr_warning` to its local `warnings` list when truthy, which already flows into `ExecuteRoundResult.warnings` (existing field, previously always empty — this is its first producer).

- [ ] **Step 1: Write the failing tests**

First, give the test double a way to inspect what prompt each agent actually received — `_MultiStubAdapter` currently discards `prompt`. In `test_execute_round.py`, modify `_MultiStubAdapter.__init__` and `dispatch_agent`:

```python
    def __init__(
        self,
        canned_outputs: dict[str, str],
        exit_codes: dict[str, int] | None = None,
        shell_overrides: dict[str, tuple[str, int]] | None = None,
        git_commit_sha: str = "abcd1234ef5678901234abcd1234ef5678901234",
    ) -> None:
        self._canned = canned_outputs
        self._exit_codes = exit_codes or {}
        self._shell_overrides = shell_overrides or {}
        self._git_commit_sha = git_commit_sha
        self._dispatched: list[str] = []
        self._dispatched_prompts: dict[str, str] = {}

    def dispatch_agent(
        self,
        agent_name: str,
        prompt: str,
        timeout_seconds: int = 600,
        isolation: Literal["worktree"] | None = None,
    ) -> AgentDispatchResult:
        self._dispatched.append(agent_name)
        self._dispatched_prompts[agent_name] = prompt
        return AgentDispatchResult(
            output=self._canned.get(agent_name, ""),
            exit_code=self._exit_codes.get(agent_name, 0),
            duration_ms=10,
            agent_type=agent_name,
            error=None if self._exit_codes.get(agent_name, 0) == 0 else "stub error",
        )
```

(Only the two new lines — `self._dispatched_prompts: dict[str, str] = {}` in `__init__` and `self._dispatched_prompts[agent_name] = prompt` in `dispatch_agent` — are additions; everything else in this block is unchanged and shown for exact placement.)

Then append these tests at the end of the file:

```python
# ---------------------------------------------------------------------------
# ocr pre-filter wiring into the CR dispatch
# ---------------------------------------------------------------------------


def test_ocr_prefilter_disabled_leaves_cr_brief_unchanged(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """Default (env var unset) behavior is unchanged: no section, no warnings."""
    monkeypatch.delenv("ARCGENTIC_OCR_PREFILTER", raising=False)
    handoff = tmp_path / "handoff.md"
    handoff.write_text(_MINIMAL_HANDOFF, encoding="utf-8")
    stub = _make_default_stub()
    result = run(
        round_name="R10-L3-aletheia",
        handoff_path=handoff,
        dry_run=True,
        adapter=stub,
        repo_root=tmp_path,
    )
    assert result.exit_code == 0, result.error
    assert result.warnings == []
    cr_prompt = stub._dispatched_prompts["cr-reviewer"]
    assert "OCR PRE-FILTER" not in cr_prompt


def test_ocr_prefilter_section_reaches_cr_reviewer_prompt(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """When enabled, ocr's findings are appended to the actual cr-reviewer brief."""
    monkeypatch.setenv("ARCGENTIC_OCR_PREFILTER", "1")
    handoff = tmp_path / "handoff.md"
    handoff.write_text(_MINIMAL_HANDOFF, encoding="utf-8")
    stub = _make_default_stub(
        shell_overrides={"ocr review": ("file.py:9 unchecked null deref", 0)},
    )
    result = run(
        round_name="R10-L3-aletheia",
        handoff_path=handoff,
        dry_run=True,
        adapter=stub,
        repo_root=tmp_path,
    )
    assert result.exit_code == 0, result.error
    assert result.warnings == []
    cr_prompt = stub._dispatched_prompts["cr-reviewer"]
    assert "OCR PRE-FILTER FINDINGS" in cr_prompt
    assert "file.py:9 unchecked null deref" in cr_prompt


def test_ocr_prefilter_failure_surfaces_as_warning_not_error(
    tmp_path: Path, monkeypatch: _pytest.MonkeyPatch
) -> None:
    """A broken ocr install degrades to a warning; the round still succeeds."""
    monkeypatch.setenv("ARCGENTIC_OCR_PREFILTER", "1")
    handoff = tmp_path / "handoff.md"
    handoff.write_text(_MINIMAL_HANDOFF, encoding="utf-8")
    stub = _make_default_stub(
        shell_overrides={"ocr review": ("ocr: command not found", 127)},
    )
    result = run(
        round_name="R10-L3-aletheia",
        handoff_path=handoff,
        dry_run=True,
        adapter=stub,
        repo_root=tmp_path,
    )
    assert result.exit_code == 0, result.error
    assert any("ocr pre-filter skipped" in w for w in result.warnings)
    cr_prompt = stub._dispatched_prompts["cr-reviewer"]
    assert "OCR PRE-FILTER" not in cr_prompt
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd toolkit && pytest tests/unit/skills_impl/test_execute_round.py -k ocr_prefilter -v`
Expected: the 3 new `run()`-level tests FAIL (`AttributeError: '_MultiStubAdapter' object has no attribute '_dispatched_prompts'` for the two enabled cases, and the disabled case fails on the same attribute or on `result.warnings` / brief content depending on execution order) — the 5 Task 1 tests still PASS.

- [ ] **Step 3: Wire the helper into `_phase_dev_body` and `run()`**

In `toolkit/src/arcgentic/skills_impl/execute_round.py`, update the `_phase_dev_body` signature and docstring (starting at line 463):

```python
def _phase_dev_body(
    adapter: IDEAdapter,
    round_name: str,
    ba_design: str,
    handoff_md: str,
    repo_root: Path,
    dry_run: bool,
) -> tuple[PhaseResult, dict[str, str], int, int, str, str, str | None]:
    """Phase 3: dev body — dispatch developer + run quality gates + inline CR + SE.

    Returns (PhaseResult, quality_gates, cr_findings_count, se_findings_count,
             cr_findings_md, se_findings_md, ocr_warning).

    ocr_warning is None unless the opt-in ocr pre-filter was enabled and
    failed; when set, it belongs in ExecuteRoundResult.warnings — it never
    raises.

    Mandate #20: SE brief MUST NOT contain ba_design — pass only contract-extracted text.
    """
```

Replace the CR-brief construction block (the code between the quality-gate-failure check and the `cr_result = adapter.dispatch_agent(...)` call, originally lines 506-512):

```python
    # Opt-in ocr pre-filter (ARCGENTIC_OCR_PREFILTER=1) — never blocks the round
    ocr_section, ocr_warning = _run_ocr_prefilter(adapter, repo_root)

    # Inline CR step — sees BA design (per spec § 5.4)
    cr_brief = (
        f"Review the dev-body diff for round {round_name}. BA design was at "
        f"{ba_path}; dev output follows. Produce a "
        f"P0/P1/P2/P3 findings table.\n\nDev output:\n\n{dev_result.output}\n\n"
        f"BA design:\n\n{ba_design}"
        f"{ocr_section}"
    )
```

Update both `_phase_dev_body` return statements to add `ocr_warning` as the 7th tuple element. The dry-run return (originally lines 560-567):

```python
    if dry_run:
        return PhaseResult(
            phase_name="dev-body",
            commit_sha=None,
            files_touched=["<dev-body files; dry_run>"],
            sub_agent_dispatched="developer",
            quality_gates=quality_gates,
        ), quality_gates, cr_findings_count, se_findings_count, cr_findings_md, se_findings_md, ocr_warning
```

And the committed return (originally lines 578-586):

```python
    subject = f"feat({round_name}): {round_name} dev body"
    sha = adapter.git_commit(subject)
    return PhaseResult(
        phase_name="dev-body",
        commit_sha=sha,
        files_touched=files_touched,
        sub_agent_dispatched="developer",
        quality_gates=quality_gates,
    ), quality_gates, cr_findings_count, se_findings_count, cr_findings_md, se_findings_md, ocr_warning
```

Finally, update the Phase 3 call site inside `run()` (originally lines 700-704):

```python
        # Phase 3
        p3, quality_gates, cr_count, se_count, cr_md, se_md, ocr_warning = _phase_dev_body(
            adapter, round_name, ba_design, handoff_md, repo_root, dry_run
        )
        phases.append(p3)
        if ocr_warning:
            warnings.append(ocr_warning)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd toolkit && pytest tests/unit/skills_impl/test_execute_round.py -v`
Expected: all tests pass, including the full pre-existing suite (the `_phase_dev_body` direct call in `test_phase_dev_body_raises_on_empty_staged` raises before reaching any return statement, so the new 7th tuple element never affects it).

- [ ] **Step 5: Run mypy and ruff**

Run: `cd toolkit && mypy --strict src/ tests/ && ruff check .`
Expected: no errors (the new code is fully typed; `tuple[str, str | None]` and the 7-element return tuple are explicit).

- [ ] **Step 6: Commit**

```bash
git add toolkit/src/arcgentic/skills_impl/execute_round.py toolkit/tests/unit/skills_impl/test_execute_round.py
git commit -m "feat(execute-round): feed ocr pre-filter findings into the cr-reviewer brief"
```

---

### Task 3: Teach `cr-reviewer` to consume the pre-filter section

**Files:**
- Modify: `agents/cr-reviewer.md:34-36` (insert new section between "Input" and "Output")

**Interfaces:**
- Consumes: the `OCR PRE-FILTER FINDINGS` section text produced by Task 2's `_run_ocr_prefilter` / `cr_brief` construction — this task only needs to know that exact heading string so its instructions match what actually appears in the brief.
- Produces: no new interface — this is an instruction-only change to the agent's own system prompt; nothing downstream depends on new types or function names.

- [ ] **Step 1: Insert the new section**

In `agents/cr-reviewer.md`, insert the following between the closing code fence of the "Input — what you receive" section (line 34, the ` ``` ` after the `TASK:` block) and the `## Output — what you produce` heading (line 36):

```markdown

## OCR pre-filter (optional)

Some briefs include an `OCR PRE-FILTER FINDINGS` section after the BA design
text — output from `open-code-review`'s deterministic diff scan, run before
you were dispatched. When present:

- Verify each listed finding against the actual diff and the BA design:
  confirm it, adjust its severity, or reject it as a false positive.
- Do NOT perform an independent from-scratch scan of file regions the
  pre-filter already covered — spend your reasoning on verification and on
  dimensions/regions it did not cover, not on rediscovering the same lines.
- Still self-enforce the existing quality bar below (4-dimensional coverage,
  3-7 findings, concrete file:line references) — the pre-filter narrows
  where you look, it does not change what "done" means.

When no such section is present in the brief, proceed exactly as documented
below.
```

- [ ] **Step 2: Verify placement**

Run: `grep -n "OCR pre-filter\|## Output" agents/cr-reviewer.md`
Expected: the `## OCR pre-filter (optional)` heading appears once, immediately before `## Output — what you produce`.

- [ ] **Step 3: Commit**

```bash
git add agents/cr-reviewer.md
git commit -m "docs(cr-reviewer): document how to consume the ocr pre-filter section"
```

---

### Task 4: Document the fork, install path, and ocr setup

**Files:**
- Modify: `README.md` (badges block at line 16, CLI install section at lines 152-159)
- Modify: `skills/execute-round/SKILL.md:18-24` (Prerequisites)

**Interfaces:**
- Consumes: nothing (documentation only)
- Produces: nothing (documentation only) — this task has no code dependents; it can run independently of Tasks 1-3, though it reads more sensibly after them since it documents the env var they introduce.

- [ ] **Step 1: Add a fork notice to the README**

In `README.md`, insert this block immediately after the badges line (after line 16, before the `Arcgentic helps Codex and Claude Code run software work...` paragraph):

```markdown

> **This is the Pezzutti Group fork** of upstream Arcgentic. It carries one
> addition: an opt-in `ocr` (`open-code-review`) pre-filter on the CR gate
> (see "ocr pre-filter" under Install below). Everything else tracks
> upstream `Arch1eSUN/Arcgentic` via periodic merges.
```

- [ ] **Step 2: Point the CLI install at this fork and document the env var**

Replace the "### CLI install" section (lines 152-159):

```markdown
### CLI install

Use this if you only need the command-line helper. Install from this fork
(not the upstream PyPI package) to get the ocr pre-filter patch:

```bash
pipx install git+https://github.com/Pezzutti-Group-S-p-A/arcgentic-fork.git
arcgentic --help
```

### ocr pre-filter (optional, this fork only)

`execute-round`'s inline CR step can run
[open-code-review](https://github.com/alibaba/open-code-review) (`ocr`) as a
deterministic pre-filter before dispatching `cr-reviewer`, cutting review
tokens by having the agent verify candidate findings instead of scanning the
diff cold. It is off by default and never blocks a round if missing or
broken.

Setup, once per machine:

```bash
npm install -g @alibaba-group/open-code-review
ocr config provider   # select Anthropic
ocr config model      # select Claude Haiku
```

The Anthropic API key `ocr` uses must come from `pzt-secret` — never placed
in `ocr`'s config file or any committed file:

```bash
pzt-secret run --env ANTHROPIC_API_KEY=anthropic-ocr-prefilter -- ocr review
```

To turn the pre-filter on for a round:

```bash
ARCGENTIC_OCR_PREFILTER=1 pzt-secret run --env ANTHROPIC_API_KEY=anthropic-ocr-prefilter -- arcgentic execute-round-impl --round=$ROUND --handoff=$HANDOFF_PATH
```

If the secret `anthropic-ocr-prefilter` does not yet exist in `kv-pzt-agent`,
create it via `pzt-secret put` (tags: `owner=alberto.camilli`, `rotation`,
`source=anthropic-console`) before enabling the env var.
```

- [ ] **Step 3: Note the optional setup in `execute-round`'s own prerequisites**

In `skills/execute-round/SKILL.md`, after the existing "Requires the `arcgentic` CLI" bullet block (after line 22, before the "Requires a planned handoff doc" line), add:

```markdown

Optional: set `ARCGENTIC_OCR_PREFILTER=1` (with `ocr` installed and its
Anthropic key supplied via `pzt-secret`) to enable the ocr pre-filter on the
inline CR step. See README.md § "ocr pre-filter" for setup. Unset by
default — no behavior change if you skip this.
```

- [ ] **Step 4: Verify the docs read correctly**

Run: `grep -n "ocr pre-filter\|ARCGENTIC_OCR_PREFILTER\|Pezzutti Group fork" README.md skills/execute-round/SKILL.md`
Expected: each phrase appears in the expected file/section; no leftover reference to `pipx install arcgentic` (without the fork URL) remains in the CLI install section.

- [ ] **Step 5: Commit**

```bash
git add README.md skills/execute-round/SKILL.md
git commit -m "docs: document the fork, ocr pre-filter setup, and install path"
```
