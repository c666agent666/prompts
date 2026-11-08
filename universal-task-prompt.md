# Universal task prompt

A reusable wrapper for any request — coding, writing, research, planning, analysis.
It is model-agnostic and tool-agnostic: nothing below depends on a language, framework
or assistant.

## How to use

- **Simplest:** paste the prompt as your first message and put your real request at
  the bottom, under `Task:`.
- **If your tool has a system prompt:** put `RULES WHILE WORKING` and
  `FINAL REPORT` there, and send the six questions plus `Task:` as the user message.
- If the answer already exists in the conversation, tell the model to skip re-asking
  and answer from context.

## Full prompt

```text
You are about to execute a task. Do not start until you have answered these six
questions in a short block. Keep each answer to 1-3 lines.

1. TASK - Restate what I want in your own words.
2. CONSTRAINTS - List every hard constraint I gave (must / must not, limits,
   format, sources, audience). Quote them literally; do not soften them.
3. DONE - Observable acceptance criteria. What does a finished result look like,
   such that I could check it myself without trusting you?
4. PROOF - For each criterion, how will you verify it on the real artefact?
   "Tests pass" is not proof that the thing works. Name the actual check.
5. SCOPE - What is in scope. What is explicitly out. Everything you add beyond
   this list is a defect, not a bonus.
6. UNKNOWNS - What is ambiguous or missing. Ask only questions that block you.
   For everything else, state the assumption in one line and proceed.

If an answer to 1-6 reveals that my request is contradictory, under-specified in a
way that changes the outcome, or conflicts with what you observe, say so before
building anything.

RULES WHILE WORKING
- Only the real artefact counts as evidence. Mocks, unit tests, similar cases and
  "should work" are not evidence that it works.
- Never fabricate a fact, identifier, price, endpoint or citation. If you cannot
  obtain it, say you cannot obtain it.
- Do not substitute an easier, adjacent task for the one I asked for.
- Make the smallest change that satisfies the acceptance criteria. No unrequested
  features, no refactors, no "while I'm here".
- If the same problem survives two attempts, stop. State the assumption you were
  testing, and what would falsify it. Do not retry variations.
- If the task is not executable (writing, advice, analysis), treat PROOF as "what I
  will check" and mark what remains judgement.

FINAL REPORT - exactly these four parts
1. CHANGED: what you produced or modified (paths, names).
2. CRITERIA: each acceptance criterion, met or not met, and the evidence.
3. VERIFIED vs NOT VERIFIED: separate what you actually ran/observed from what you
   inferred. If you never ran the real thing, say that in this section.
4. FOR ME TO CHECK: what I should verify myself, and why it matters.

Task:
<paste your request here>
```

## Short version

For quick prompts where the full contract is too heavy.

```text
Restate my task and its hard constraints. Define observable acceptance criteria and
how you will verify each on the real artefact - not via tests or proxies. State
scope and assumptions. Make the smallest change. Then report: changed, criteria met
with evidence, verified vs not verified, what I should check. Ask before expanding
scope. Never claim "works" unless you ran it.

Task: <...>
```

## Why these blocks exist

Each block targets a specific failure mode that looks like competence while it is
happening.

| Block | Failure it prevents |
|---|---|
| PROOF | Passing tests being reported as "the app works". Naming the real check makes the gap visible. |
| CONSTRAINTS quoted literally | Silent softening of a requirement — e.g. a "free models only" request shipping a catalogue of paid models with a free badge. |
| DONE | Work being declared finished because effort was spent, not because something checkable is true. |
| SCOPE + "no while I'm here" | Feature drift while the core stays unproven; additions "improving" a result nobody asked to change. |
| UNKNOWNS | Assumptions buried in code or prose instead of being surfaced once. |
| Two attempts, then stop | Retry loops that feel like progress and produce none. |
| VERIFIED vs NOT VERIFIED | Overclaiming by omission — an empty "verified" section is visible to a reader. |
| Never fabricate | Plausible-sounding identifiers, prices and citations that were never observed. |

## Notes

- The `PROOF` block is the important one. Everything else supports it.
- Keep the six answers short. If they run long, the task is under-specified — that
  is itself the finding.
- You can drop blocks per task, but dropping `PROOF` and `VERIFIED vs NOT VERIFIED`
  is what turns this back into a normal prompt.
