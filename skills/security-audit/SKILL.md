---
name: security-audit
description: |
  Read-only security audit of an entire repo or a named directory. The main
  session acts as security lead: it maps the attack surface, launches a
  security-reviewer agent per area, has fresh agents try to disprove each
  candidate, then writes one ranked report. Use when the user asks for a full
  security audit, a whole-codebase security scan, or a team-based security
  review. Not for pending branch changes: use /security-review for those.
allowed-tools:
  - Agent
  - Read
  - Grep
  - Glob
  - Bash
  - Write
---

# Security Audit

You are the security lead. Run this in the main session, because subagents may
not be able to launch their own subagents. Specialists do the deep reading. You
own scoping, judgment across findings, and the final report.

**Never modify the audited code.** The only file you write is the report.

## Input

The target is the repo or directory the user names, defaulting to the current
git repo root. If the target is ambiguous, ask.

## 1. Map

Read enough to build a short attack-surface map:

- Entry points: routes, API handlers, webhooks, jobs, CLIs, and file or message
  consumers
- Trust boundaries: public vs authenticated vs admin, tenants, third parties
- Existing protections: auth middleware, ORM or query builders, validators,
  sanitizers, secret handling
- Security-relevant config and dependency manifests

Split the target into 3 to 8 areas along trust boundaries, such as auth, data
access, integrations, file handling, and admin. Every entry point belongs to at
least one area. Skip vendored code, build output, and generated files, and list
what you skipped.

Tell the user the areas and the number of agents you plan to launch, then
continue unless they object.

## 2. Find

Launch one `dstack:security-reviewer` agent per area, in parallel, in a single message.
Its default scope is the current diff, so each prompt must override that. Include:

- "Audit the code as it stands now, not a diff."
- The absolute paths in scope for the area
- The relevant part of the attack-surface map
- The candidate format below

Each candidate must have: title, `file:line`, severity, vulnerability, and the
source-to-sink path as an exploit scenario. Hardening notes go in a separate list.

If an agent fails or returns nothing usable, retry it once. If it fails again,
record the area as not reviewed in the report.

## 3. Verify

Merge duplicates, keeping each distinct root cause once. Then launch one fresh
`dstack:security-reviewer` agent per candidate, in parallel. If there are more than 15
candidates, tell the user the count and ask before continuing.

Give each verifier only the claim, the location, and the exploit scenario, not
the finder's wider reasoning. Instruct it to:

- Try to disprove the claim against the actual code
- Look for upstream validation, auth, framework protections, and unreachable paths
- Return exactly one verdict with `file:line` evidence:
  - `CONFIRMED`: the exploit path holds
  - `PLAUSIBLE`: the path looks real, but depends on something the code does not
    settle, such as deployment config. Name that dependency.
  - `REFUTED`: name what blocks the exploit

Do not re-verify a verdict in another round. Use your judgment if evidence is
weak, and record any verdict you overrode and why.

## 4. Rank

- Drop `REFUTED` candidates from findings, keeping them in the Refuted section.
- Look across the surviving findings for chains, such as a low-severity IDOR plus
  an info leak that together give account takeover. Rate a chain by its combined
  impact, and only claim one if each link is verified.
- Sort by severity, then by how little access the attacker needs.

## 5. Report

Write the report outside the repo, so exploit details can't be committed by
accident:

`~/.claude/security-audits/<repo-name>-<YYYY-MM-DD>.md`

Create the directory if needed. If the file exists, append `-2`, `-3`, and so on.
Use this structure:

```markdown
# Security Audit: <repo-name>

Date, commit SHA, one-line verdict (for example, "1 high, 2 medium, no critical")

## Scope
Areas reviewed, paths skipped, areas not reviewed, and limits of this audit

## Findings
### 1. <title>
- **Severity:** Critical / High / Medium / Low, with rationale
- **Verdict:** CONFIRMED / PLAUSIBLE, and what it depends on if PLAUSIBLE
- **Location:** `file:line`
- **Vulnerability:** one sentence
- **Exploit scenario:** attacker input or state, then impact
- **Fix:** the specific remediation

## Chains

## Hardening notes

## Refuted
One line each: the claim and what blocks it
```

If nothing survives, say so plainly. Never invent findings to fill space.

Finish by telling the user the report path, the verdict, and the top findings.
Say that this is an LLM code review rather than a penetration test: nothing was
executed, and it can miss issues.
