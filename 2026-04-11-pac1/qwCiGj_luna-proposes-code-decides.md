---
source_code: https://github.com/raquelpanapalen/bitgn-pac1
run_ids:
  - run-22x59r1Rge2GZXPGp2qo4ATu8
  - run-22x5BYbXqBhQWRx2tw3Bt6bKM
  - run-22x7MgTRsTgsQ6E3XAqZEmuB6
  - run-22x7PqjQCbZcDyvKRUV2KaJYM
  - run-22x7TycjMVamwx5EygGUxy98r
  - run-22x7W3Er4YWQoL1GA5Gg9pCdm
  - run-22x9xnWCZirzCbv9MsnSo8ABa
  - run-22x9zpK9mXH2Qe97MD6qE8ayC
model_names:
  - gpt-6-luna
author: Raquel Panadero Palenzuela
author_linkedin: https://www.linkedin.com/in/raquel-panadero-palenzuela/
author_github: https://github.com/raquelpanapalen
impact: The cheapest OpenAI model in a tight Pydantic AI harness where the model proposes and code decides everything the grader checks; ~97% on PROD at $0.20 per run.
challenge: PAC
---

# Luna proposes, code decides (PAC1)

A post-competition PAC1 agent built on `gpt-6-luna`, the cheapest OpenAI model
available to us, inside a thin harness of our own on
[Pydantic AI](https://ai.pydantic.dev/). The model only reads, judges and
proposes. Code owns every graded action:
- which changes reach the vault;
- which references are cited;
- which outcomes are allowed;
- whether the trial ends with an answer at all.

**Results, with how they were obtained.** These runs were made after the
competition window, so none of them is a Hall of Fame entry.

| Stage | PROD (104 tasks) | What we had seen of PROD | Cost per PROD run |
|---|---|---|---|
| Milestone 1 (runs 1–2 above) | **87 / 104 (83.7%)**, mean of 86 and 88 | totals only | $0.18 |
| After phase 2 (runs 3–8 above) | **100.7 / 104 (96.8%)**, six runs: 104, 100, 96, 100, 101, 103 | per-task feedback, used to make fixes more general | $0.20 |

Milestone 1 is the number comparable with blind entries. Phase 2 used PROD's
per-task feedback as a guide. Every fix had to be a rule or check that would hold
in an unseen workspace, with no task ids, folder names, field names or record
kinds taken from the benchmark, but PROD is no longer an unbiased estimate after
that. On DEV the same code scores about 42.5 / 43 (99%) at $0.05 per run.

The full trail is in the repo:
- [technical report](https://github.com/raquelpanapalen/bitgn-pac1/blob/main/docs/technical-report.md);
- [every experiment](https://github.com/raquelpanapalen/bitgn-pac1/blob/main/docs/experiments.md), each with its BitGN run ids, result and verdict.

## How does it work?

```
task ─► grounding ─► gpt-6-luna tool loop ─► final answer ─► answer checks (code)
          │               │        │                          │  send back once
          │               ▼        ▼                          ▼
          │        in-memory   staged changes       OK: apply net changes
          │        vault copy                       other: discard changes
          ▼                                                   │
  tree · root rules file · nested rules list ·                ▼
  date · sender facts                         answer to BitGN, refs built by code
```

1. **Load the whole vault into memory** at trial start (up to 500 files, 8
   parallel reads; reads aren't graded, so this is free). Search and `count` run
   on the copy, so they are exact and never silently truncated. `find` matches
   substrings and globs, not only whole file names. Output that is cut shows an
   explicit `[N matching lines in total …]` marker.
2. **Ground the model** before the task:
   - the tree to depth 2;
   - the root `AGENTS.md`, found whatever its case;
   - a list of the nested rules files, to read before working in their folder;
   - the runtime date;
   - **sender facts** computed in code for every inbox message: exact contact,
     lookalike (by name or by domain edit distance) or unknown, where a chat
     handle appears in the vault, and which code-like tokens in the body appear
     in other files.
3. **Tool loop** with native tool calls on the Responses API: `tree`,
   `list_dir`, `find`, `search`, `count`, `read`, `today`, plus `write`,
   `delete`, `mkdir`, `move` and `unstage`. Every change is **staged on the
   copy**. A `delete` must quote, word for word, the instruction that requires it.
4. **Final answer** is a typed object: `answer` (only the exact value asked
   for), `summary`, `outcome` and `grounding_refs`.
5. **Answer checks in code** each send the answer back once with a reason:
   - `OUTCOME_ERR_INTERNAL` is refused unless a tool call actually failed;
   - staged JSON, YAML and frontmatter must parse;
   - before an OK answer with changes, the model reviews a compact diff of each
     file's net change;
   - an OK answer that consumes a trust code sent by a sender the vault already
     lists goes back;
   - an answer that acts on the inbox but relies on a later item than the first
     pending one goes back.
6. **Apply or discard.** Only an accepted `OUTCOME_OK` applies the staged changes,
   as their net effect: one write per file. Anything else leaves the vault
   untouched. References sent are the model's refs plus every file read, changed
   or found, as bare paths.

The judgement calls live in a short system prompt: an ordered outcome policy and
about eight general judgement rules.

## Models

- **Main solver:** `gpt-6-luna`, medium reasoning effort, through Pydantic AI
  (pinned) on the OpenAI Responses API. No router, classifier, planner or judge.
- **Reference:** `gpt-6.1-terra` on the same harness. At milestone 1 it scored
  about 3.5 tasks more on DEV, for about 7× the cost. After phase 2, at low
  effort, it scored level with luna on PROD (100.5 against 100.0 of 104) for
  about 5.4× the cost, failing on the same judgement calls.
- **Settings that mattered:**
  - an 8192-token output cap;
  - after a capped reply, retry with *lower* reasoning effort instead of
    repeating it (medium → low → none);
  - a 30-request limit, with a warning 5 requests before it;
  - a "report now" fallback that turns a stopped run into a final answer.
- All models are OpenAI models, by our own choice; none are open-weight.

## Which problems did we hit, and how did we solve them?

- **The sample loop made the model give up.** The BitGN sample replays history
  as calls to tools that aren't declared, and that cost more than model size did:
  the same `gpt-6.1-terra` went from 41% to 95% on DEV after we moved to native
  tool calls. luna went from 61% (sample) to 76% from the loop change alone.
- **Silent truncation and brittle `find`.** The sample capped search at 20
  results with no marker, so counting over a 1,025-line file failed every run.
  The in-memory copy fixes this by construction.
- **Stray edits on refusals.** Staging changes and applying them only on OK
  removed the whole class. In every passing trial we recorded, a non-OK outcome
  came with zero changes.
- **Rewrites the grader counts as unexpected.** The grader counts every write
  event and compares bytes. We therefore:
  - apply one net write per file;
  - keep a file's ending on whole-file rewrites;
  - keep text byte-identical when a write adds a block in front of it.
- **No answer, or a made-up internal error.** luna sometimes spent its whole
  reply budget on reasoning twice. The lower-effort retry and the "report now"
  fallback mean that every trial since has ended with a deliberate answer.
- **Outcome choice**, the largest class once the mechanical failures were gone:
  - denial only for instructions that come from untrusted content;
  - "unsupported" means not provided by *this workspace's* files and rules;
  - clarification for ambiguity, conflicts and not-found.

  Sender trust is decided from code-computed facts plus the workspace's own
  rules, which we never hard-code.

## What we learned

1. **A perfect DEV score overfits.** Experiments 6–18 took DEV from 95% to 99%,
   with six perfect runs, while PROD *fell* from 83.7% to 77.9%. Rules written
   against a few tasks carried their assumptions.
2. **Format-level code transfers.** "Records must parse", "kept text stays
   byte-identical", "rules files whatever their case": changes like these, plus
   judgement rules worded for any workspace, took PROD from 77.9% to 96–98%
   without losing DEV.
3. **A second opinion from a summary hurts.** A fresh-context outcome judge, run
   offline on 376 recorded trials, fixed 19 wrong outcomes and broke 86 right
   ones. The agent that did the work knows more than a reviewer working from a
   summary.
4. **Measure everything, on full runs.** Low reasoning effort looked like a free
   30% saving and cost 2.7 tasks. The same code varies by ±1–2 tasks per DEV run
   and ±4 per PROD run, so no change was decided on one run.
5. **Remove what stops earning its place.** Every check is logged per trial. A
   Decisions-API "not found?" check, a relative-date check and an
   alphabetical-order check were removed once the counts showed they caught less
   than they cost. Shortening the prompt, though, cost a security task
   immediately: prompt text is not free to prune.

## How could it be better?

- **A new blind set.** PROD has guided phase 2, so only an unseen task set would
  confirm that the general rules really generalise.
- **Choosing between candidate records.** Most remaining failures are asking for
  clarification, or picking a record, before reading and comparing all the
  records that could fit. Bigger models fail the same way, so this looks like a
  harness or policy fix, not a model-size one.
- **Move judgement into code where it is checkable.** Every rule that can become
  a format-level check is more reliable as code than as prompt text.
