---
name: compass
description: User-invoked mid-session audit skill. Detects (1) drift between current task and original session intent via transcript analysis, and (2) codebase rot — git uncommitted accumulation, test/lint state from cached artifacts, file bloat, TODO accumulation, circular dependencies, module boundary degradation. Manually triggered via `/compass` only — never auto-invoked. Use when the user types `/compass` optionally followed by a baseline intent string. Without arguments, derive baseline via cascade — most recent spec (24h), then task-init heuristic, then explicit user prompt. Never use the literal first user message of the transcript as baseline.
---

# /compass — Mid-Session Drift & Rot Audit

## §1. Trigger Gate

**Hard rule:** activate ONLY when the user types the literal `/compass` slash command. Never activate from natural-language mention of "drift", "audit", "compass", "long session", "rot", or similar concepts.

Both forms activate:
- `/compass` (no argument)
- `/compass <intent string>` (explicit baseline)

If the user discusses these concepts without typing `/compass`, do NOT activate this skill — answer the question normally.

**Why so strict:** prior cascades from over-eager hook/skill activation are documented failures (Stop hook MCP cascade, subprocess recursion). `/compass` follows `/decide`'s manual-only philosophy.

## §2. Input Resolution

The drift baseline (= "what was this session/task supposed to be about") is determined by this fallback chain in order. Use the FIRST step that succeeds. Quote the source in the output for transparency.

> ⚠️ **Do NOT use the literal first user message of the transcript as baseline.** Long sessions span multiple tasks; the first message is usually unrelated to the current one. compaction makes this worse. This rule is mandatory.

**Cascade (auto-extract when no argument given):**

1. **Explicit argument** — if `/compass <intent>` was invoked with text, that text IS the baseline. Skip the rest of the cascade.

2. **Most recent spec (≤24h)** — list `~/docs/superpowers/specs/` and find any file with mtime within the last 24 hours.
   - `ls -lt ~/docs/superpowers/specs/ | head -5`
   - If a recent spec exists, read its §1 (Problem & Motivation) and §2 (Concept) as baseline source. Quote the spec filename in the output.

3. **Task-init heuristic** — read the current session transcript jsonl in REVERSE order. Find the most recent task-init signal:
   - Skill invocations: `/decide`, `/compass`, `superpowers:brainstorming`, `/team`, `/research-team`, `/investigate`, `/ship`
   - The user message *immediately following* such a signal is the baseline.
   - If no skill signal found: find the most recent user message that introduces a NEW noun-phrase topic following an assistant-side conclusion or summary (heuristic — Opus judges).
   - Quote the transcript line number in the output.

4. **Explicit user prompt (last resort)** — output exactly:
   > "이번 task 의도가 뭐야? 한 줄로 알려줘." (or English equivalent matching the user's language)

   This is a one-shot turn. Do NOT loop. Wait for user response, then proceed with that as baseline.

**Recovery:** if step 3's transcript scan fails (file unreadable, wrong session id), fall through to step 4 immediately.

## §3. Drift Check

[content in Task 4]

## §4. Rot Check

[content in Task 5]

## §5. Synthesizer & Severity

[content in Task 6]

## §6. Output Template & Action Chain

[content in Task 7]

## §7. Trust Boundary

[content in Task 8]

## §8. Failure Modes & Considered Line

[content in Task 8]
