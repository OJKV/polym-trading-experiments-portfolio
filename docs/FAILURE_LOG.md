# Failure log

Newest first. Never delete an entry. Ids are `F###`. A rule or test that a failure taught cites its id.

**Severity:** critical = money lost, or could have been · high = blocked or wrong results ·
medium = wasted time · low = annoyance.

**Inherited lessons** are not repeated here. Read them at the source:
- ARGUS (LLM pipeline): 31 relevant lessons summarised in [`research/07-argus-reuse-review.md`](research/07-argus-reuse-review.md) §4. Full text in `argus-project/docs/FAILURE_LOG.md`.
- ntx (deterministic engine): encoded as rules in [`RULES.md`](RULES.md). Full text in ntx `docs/05-failure-log.md`.

## Template

```markdown
## F### — <title>
**Date:** YYYY-MM-DD · **Phase:** 0–4 · **Severity:** critical|high|medium|low
**Symptom:** what was observed (exact error / number).
**Root cause:** the cause, proven (repro, test, log line).
**Fix:** what changed (commit SHA once committed).
**Prevention:** regression test / rule / doc added.
**Cost:** time and money lost.
```

---

## F002 — A re-test agent rewrote two canonical docs outside its brief
**Date:** 2026-10-08 · **Phase:** 0 · **Severity:** high
**Symptom:** `docs/DATA_SOURCES.md` appeared before the workflow's assembler had run, and a personal guide grew from ~70 to 255 lines, with unverified figures (e.g. "Gemini free tier 1,000 requests/day", a GDELT key).
**Root cause:** the Haiku agent `retest:llm`, whose only job was to curl a list of URLs, wrote both files. Its prompt never said "do not write files", and a general workflow agent has Write/Edit.
**Fix:** both files are rebuilt from validated workflow output; the agent's text is discarded.
**Prevention:** read-only agents run as the `Explore` type (no Write/Edit), and every read-only prompt starts with "you must not create, edit or delete any file". Each workflow names exactly one writer per file.
**Cost:** ~20 min of tracing; no money.

## F001 — Research workflow ran every agent on the top model and hit the session limit twice
**Date:** 2026-10-07 · **Phase:** 0 · **Severity:** medium
**Symptom:** The 15-agent research workflow consumed about 2.5M subagent tokens; two fact-checkers and the synthesis failed with "session limit"; the resumed run stopped during synthesis.
**Root cause:** Workflow and subagent calls inherited the main (most expensive) model for every task, including mechanical extraction and summarising.
**Fix:** Synthesis rerun on Sonnet, the rule-extraction task on Haiku (2026-10-08).
**Prevention:** Model triage is now a standing instruction (AGENTS.md §Agent economy): Haiku for mechanical work, Sonnet for research, synthesis and drafting, the top model only for design and hard debugging.
**Cost:** ~1 day of wall-clock delay; no money.
