---
name: adversarial-code-review
description: |
  Read-only adversarial code review of an entire repo as it stands now. GPT-6
  Astra (via `codex exec`) finds the ways the system will fail in production,
  then one fresh Claude agent per finding tries to disprove it, and the main
  session writes one ranked report. Use when the user asks for an adversarial
  code review, a whole-codebase code review, or a Codex review with disprovers.
  Not for a diff or PR: use /review or /codex challenge for those. For a
  security-only audit, use /dstack:security-audit.
allowed-tools:
  - Agent
  - Read
  - Grep
  - Glob
  - Bash
  - Write
---

# Adversarial Code Review

Two models with different blind spots: Astra finds, Claude disproves. Run this in
the main session, because subagents may not be able to launch their own.

**Never modify the reviewed code.** The only files you write are the Astra output
and the report, both outside the repo.

## Input

The target is the repo the user names, defaulting to the current git repo root.
If it's ambiguous, ask. Record the branch and short SHA (`git rev-parse --short HEAD`).

## 1. Find (Astra)

Fill in `<REPO>`, `<NAME>` and `<DATE>`, then run in the background (it took about
8 minutes on a 44k-line repo):

```bash
OUT=~/.claude/code-reviews/<NAME>-<DATE>-astra.md; mkdir -p ~/.claude/code-reviews
codex exec -C "<REPO>" -s read-only -m gpt-6-astra -c model_reasoning_effort=high \
  -o "$OUT" "$(cat <<'PROMPT'
<the prompt below>
PROMPT
)"
```

If `codex` is missing or not logged in, stop and tell the user. Don't fall back
to a Claude-only finder without asking, since that loses the second model.

Prompt (add one line naming the repo's trust surfaces if you know them, such as
"staff, portfolio, and public surfaces"):

```text
You are doing an adversarial code review of this repository as it stands now. It is not a diff review. Do not modify anything. Stay inside this repository. Do not read any .env file other than .env.example, and do not run anything that connects to a database or network.

Find the ways this system will fail in production or do the wrong thing. Think like a chaos engineer, a hostile user, and a maintainer who has to live with this code.

Cover:
- Correctness: behavior that contradicts what the code, tests, or docs claim
- Data integrity: data loss, silent corruption, partial writes, missing transactions, unsafe migrations
- Concurrency: races, duplicate job execution, non-idempotent webhooks and retries
- Failure modes: external integrations failing, timing out, rate limiting, or returning unexpected shapes; errors swallowed; resource leaks
- Security: broken access control across the app's trust boundaries, injection, secret handling, unsafe file handling, SSRF
- Tests that pass without proving the behavior they appear to cover

Start by mapping entry points (routes, API handlers, server actions, jobs, webhooks, integrations) and trust boundaries, then go deep where the damage would be worst.

Rules:
- Report only issues you traced through the actual code. If you cannot describe a concrete path to the failure, leave it out.
- Describe exploit paths in words. Do not write working exploits.
- No style comments, no compliments, no padding.
- Prefer ten real findings to fifty speculative ones. Do not invent findings to fill space.

Return findings ranked by impact and likelihood. For each one:

### <number>. <title>
- Severity: Critical / High / Medium / Low
- Category: correctness / data-integrity / concurrency / failure-mode / security / test-quality
- Location: file:line
- Claim: one sentence
- Path: the concrete input or state, the steps through the code, and the result
- Fix: the specific change

End with a section listing which areas you examined closely and which you did not.
```

When it finishes, read `$OUT`. Note run stats from the codex output (duration,
commands, tokens) for the report. If it returned nothing usable, retry once, then
stop and report the failure.

## 2. Verify (Claude)

Launch one fresh `general-purpose` agent per finding, in parallel, in a single
message. If there are more than 15, tell the user the count and ask first.

Give each agent only the claim, location, path and Astra's severity, not the rest
of Astra's output. Instruct it to:

- Read the actual code and try to disprove the claim: look for upstream
  validation, transactions, framework behavior, unreachable paths, and config
  that blocks it
- Not modify files, and not run anything that touches a database, network or
  real `.env`. Running a small isolated snippet (such as timing a regex) is fine.
- Return one verdict with `file:line` evidence: `CONFIRMED`, `PLAUSIBLE` (name
  what it depends on) or `REFUTED` (name what blocks it)
- Give its own severity, and correct any overstated detail in the claim

Don't re-verify in a second round. Record any verdict you override and why.

## 3. Report

Severity is the disprover's rating, with Astra's shown beside it. Drop `REFUTED`
findings to a Refuted section. Write the report to the reviewer app:

`~/Code/reviewer/content/<NAME>-adversarial-review-<DATE>.md`

If the file exists, append `-2`, `-3`, and so on. Structure:

```markdown
# <NAME> adversarial code review

**Repo:** path, branch at SHA
**Finder:** GPT-6 Astra via `codex exec`, whole repo, read-only
**Disprover:** one fresh Claude agent per finding
**Date:**

## Verdict
Counts by severity, how many survived, how many disprovers downgraded

## What to do first
Top 3, each with why

## Findings
Table: #, finding, severity, Astra's rating, category
Then per finding: Where, What happens, Why it's real, Limit, Fix

## Refuted

## How far to trust this
What was run vs only read, disprover anchoring bias, areas not examined, no spec given

## Run details
Astra duration, commands, tokens; repo size
```

Then reload the reviewer's cmux browser surface so the user sees it. Finish with
the verdict and top findings in under 50 words. Say it's an LLM code review:
mostly code reading, and it can miss issues.

To file findings as issues afterwards, ask first.
