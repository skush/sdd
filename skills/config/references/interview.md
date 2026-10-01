# The config interview — six questions, three calls, 22 of 26 keys

> **Reference-only.** Read by [`../SKILL.md`](../SKILL.md) steps 3 and 5. Every question here is
> phrased per [`../../_shared/ask-style.md`](../../_shared/ask-style.md): English, action-form
> labels, every technical term glossed inline, the trade-off spelled out in the description.
> Hosts without a native `AskUserQuestion` ask the same questions as **numbered plain text, one at
> a time, stop and wait** — same shape, same glosses, nothing skipped.

## Three standing rules

- **On a repeat run the current value is the first option, marked «(Recommended)»** — so re-running
  `/sdd:config` shows what the project is set to today and changes nothing by accident.
- **Group by decision, not by key.** A person tunes «how strict are the gates», not `gate_vet`.
  One answer therefore writes several keys, and the `old → new` output names each one.
- **When the current state matches no option exactly** — the normal case after a host clamp, or
  after someone hand-edited one key of a bundle — do **not** force it into the nearest option.
  Prepend a «Keep as is» option that spells out the actual current values of that group,
  mark it «(Recommended)», and say in the question text which key is the odd one out. Silently
  re-bundling a hand-edited value back into a preset is the way this skill loses a user's edit.

## Question → keys map

| # | Call | Question (the decision) | Keys it writes |
|---|---|---|---|
| 0 | opening | Stay on defaults, or tune? | none — the fork |
| 1 | 1 | Which tool is this session running in? | none directly; **clamps** `team_mode` · `workflow_mode` · `max_parallel_agents` on a non-Claude host |
| 2 | 1 | Which model tier should the agents run at? | `judgment_model` · `model_test_author` · `model_implementer` · `model_reviewer` · `effort_test_author` · `effort_implementer` · `effort_reviewer` |
| 3 | 2 | How strict should the tests and gates be? | `tdd` · `stop_on_red` · `max_red_retries` · `gate_lint` · `gate_vet` · `require_integration` |
| 4 | 2 | How should tasks execute and commit? | `team_mode` · `workflow_mode` · `max_parallel_agents` · `isolation` · `auto_commit` · `branch_strategy` |
| 5 | 3 | How deeply should the pipeline interview you? | `interview_depth` |
| 6 | 3 | Which language do you work in? | `language` · `artifact_language` |

**22 keys.** The remaining four — `cmd_test_unit`, `cmd_test_integration`, `cmd_lint`, `cmd_vet` —
are **never asked**. Empty means the command-detection cascade reads the repo's own Makefile,
package scripts and language manifests, which is a better answer than a command pinned once and
left to rot. They stay in the derived list of the step-6 output, named as «left empty for
autodetect».

---

## Q0 — the fork (step 3, before anything is asked)

> **CONTEXT.** `.claude/sdd.local.md` is already on disk with documented defaults, so the pipeline
> works right now — medium-depth interviews, English documents, TDD on, opus-tier judgment agents,
> a commit per task. The question is only whether we walk through the settings together and change
> some of them. **WHY IT MATTERS.** Nothing here is irreversible: every key can be changed later by
> running `/sdd:config` again, and each one carries its meaning inline in the file itself, so you
> can also just open it in an editor. The cost of tuning now is about six questions.
> **READ OPTIONS.**

- **«Keep the defaults» (Recommended)** — I change nothing. The file `.claude/sdd.local.md` is already in the repository with documented values, and every key in it has an explanatory comment. The pipeline works right away: medium-depth questions, documents in English, TDD on, a commit after every task. You can come back and tune it any time with the same `/sdd:config`.
- **«Go through the six questions»** — I ask six grouped questions and after each answer patch the matching keys, showing `old → new`. The groups: tool confirmation, model tier, test and gate strictness, execution and commit mode, interview depth, document language. Your comments and any unknown keys in the file stay untouched. Takes a few minutes.

---

## Call 1 — the ground the rest stands on

### Q1 — host confirmation

> **CONTEXT.** Two pipeline settings work only in Claude Code: `team_mode` (an agent team
> via `TeamCreate`) and `workflow_mode` (a dynamic `Workflow` that parallelises independent tasks).
> Codex CLI and Cursor don't have these mechanisms, so the engine there always runs sequentially with one agent.
> **WHY IT MATTERS.** If they're recorded as enabled on a host that lacks them, the setting will look
> working but behave differently — and the user will only find out at `implement`. So we confirm the
> tool explicitly. Here is what I saw: `<list the signals from detection.md and what they point to; on
> conflicting traces — name the conflict directly>`. **READ OPTIONS.**

- **«Claude Code»** — I keep the execution-mode question (Q4) in full: the agent team, Workflow, and parallelism greater than 1 are all available. Nothing is clamped.
- **«Codex CLI»** — I clamp `team_mode: false`, `workflow_mode: off`, `max_parallel_agents: 1` and say so on a separate line in the «derived, not asked» list. This isn't a loss: sequential single-agent TDD is the documented baseline that both modes degrade to anyway. The other questions stay unchanged.
- **«Cursor»** — The same as for Codex: the same three keys clamped for the same reason, the other questions unchanged.

### Q2 — model tier

> **CONTEXT.** The pipeline runs two different kinds of agents. Executors (`test-author`, `implementer`)
> write tests and code. Judges (`reviewer`, `critic`, `devils-advocate`, `strategist`, `analyst`)
> evaluate: they look for holes in the spec, review diffs, attack the idea. One key, `judgment_model`,
> sets the model tier for all five judges at once, and `model_<role>` sets it for each executor separately.
> **WHY IT MATTERS.** I ask directly because the session fundamentally can't see the list of models available to your
> account: guessing blindly would produce a setting that fails on the very first dispatch. If the chosen model
> is unavailable after all, the dispatch is retried once on `inherit` (the current session's model) and doesn't
> block the stage. **READ OPTIONS.**

- **«Judges on opus, executors on sonnet» (Recommended)** — The default: `judgment_model: opus`, `model_test_author` and `model_implementer` on `sonnet`, `model_reviewer` on `opus`. The logic is simple: judgment pays for a stronger model, mechanically writing a test doesn't. Effort (`effort_*`) stays `medium` for the executors and `high` for the reviewer, and on large features (L/XL) the engine raises the executors to `high` by itself. The `opus` default here is a floor, not a pin: if the session runs at a stronger tier, the judges go to `inherit` instead of silently downgrading.
- **«Everything on sonnet»** — `judgment_model: sonnet` plus all three `model_*` on `sonnet`. This is the supported path for accounts without Opus access: the pipeline runs fully, reviews get a bit shallower. Cheaper and faster. If you don't have Opus, pick this rather than the default, because then you see the real tier right away instead of via degradation.
- **«Inherit the session model»** — `judgment_model` and all three `model_*` become `inherit`: every agent runs on whatever model the session runs on. The `effort_*` keys are **not changed**: `inherit` isn't in their value set (`low | medium | high | xhigh | max` or a number), so they stay as they were — this will be named in the «derived, not asked» list. The simplest option when you don't want to think about tiers at all, and the most predictable in cost. Downside: the judges lose their separate lever, so a review can no longer be stronger than the rest of the work.
- **«Judges on fable»** — `judgment_model: fable` raises all five judges to the Fable tier, executors stay on `sonnet`. Makes sense when the pain point is specifically the quality of reviews and spec critique. Requires the account to have access to that tier, otherwise the dispatch falls back to `inherit` once.

---

## Call 2 — how the work is actually done

### Q3 — strictness and gates

> **CONTEXT.** After each task the engine runs the gates: unit tests, integration tests (if the
> Docker daemon responds), lint (a style check) and vet (static analysis that catches suspicious
> constructs without running the code). Plus the TDD cycle itself: a red test first, then the minimal code
> to turn it green. **WHY IT MATTERS.** Stricter gates catch more, but every extra tier
> adds time per task and can block work in a repository where lint isn't set up yet. Gates
> with no command found skip themselves, without failing. **READ OPTIONS.**

- **«Full gates, stop on red» (Recommended)** — `tdd: true`, `stop_on_red: true`, `max_red_retries: 3`, `gate_lint: true`, `gate_vet: true`, `require_integration: auto`. The test is written first; if it is still red after three attempts, the run stops and you see the problem right away. Integration tests run when Docker responds and are silently skipped when it doesn't. This is the default, and in most repositories there's no reason to change it.
- **«Don't stop on red»** — The same, but `stop_on_red: false`. A task that stays red is dropped, its dependents are blocked automatically, and the remaining branches of the DAG keep going. Useful on a long overnight run, where finishing 8 tasks out of 10 beats stopping on the second. Downside: you have to read the report carefully at the end, because part of the work isn't done.
- **«Light gates»** — `gate_lint: false`, `gate_vet: false`, `require_integration: never`, the rest as in the default. Only unit tests remain. Makes sense in a repository where lint and static analysis aren't set up yet and every run would otherwise spew noise. The price is obvious: style and static errors will be caught by review, not by the gates.
- **«No TDD»** — `tdd: false`: the engine writes code straight away, without a red test first. Faster, but you lose the safety net — it's the red test that proves the test really checks what it should rather than passing by accident. I'll warn about this in the banner every time. Pick it only if tests in this repository are written differently.

### Q4 — execution and commits

> **CONTEXT.** `implement` takes `tasks.json`, builds the dependency graph and executes the tasks. It can go
> sequentially with one agent, with an agent team (`team_mode`), or with a dynamic Workflow that parallelises
> independent branches. Parallel agents each work in their own git worktree — a separate working copy of the
> repository, so two agents never edit the same files. **WHY IT MATTERS.** Parallelism saves time
> on a wide graph and gives nothing on a narrow one, but it always makes the log harder to read. Commit
> granularity decides how finely you'll be able to roll the work back later.
> `<on a non-Claude host: the first two options are unavailable — say so plainly and make «Strictly
> sequential, one thread» the first option, marked «(Recommended)», because that is the actual state
> after the clamp>` **READ OPTIONS.**

- **«Let the engine decide, one commit per task» (Recommended)** — `team_mode: false`, `workflow_mode: auto`, `max_parallel_agents: 3`, `isolation: worktree`, `auto_commit: per_task`, `branch_strategy: feature`. This is the default, and the name is literal: the mode is chosen from the shape of the graph. A narrow chain of tasks runs sequentially, but **a wide independent graph runs as a Workflow and really does start up to three agents at once**, each in its own worktree under `.worktrees/`. If you want a guaranteed single thread, pick «Strictly sequential, one thread» below, not this one. Each task is closed by its own commit with `SDD-Task` and `SDD-AC` trailers, and the work goes on a separate feature branch.
- **«Agent team»** — `team_mode: true`: `test-author` → `implementer` → `reviewer` over the graph, coordinated through a shared task list, one worktree per agent. The fastest mode on a big feature with many independent tasks. Downside: the log gets harder to read, and on a narrow graph there's no gain at all. Works only in Claude Code.
- **«Strictly sequential, one thread»** — `team_mode: false`, `workflow_mode: off`, `max_parallel_agents: 1`, `isolation: inplace`: one working copy, no worktrees, tasks one after another, no exceptions. The easiest to read and debug, slower on a wide graph. This is also what a non-Claude host is clamped to, so on Codex and Cursor this option describes the actual behaviour rather than a choice. Pick it when you want to see exactly one thread of work.
- **«Leave commits to me»** — `auto_commit: off` plus the other defaults: the engine writes code and runs the gates but commits nothing. You decide what to commit and how at the end. Downside: you lose the `SDD-Task` / `SDD-AC` trailers that the «acceptance criterion → commit» traceability is later built from.

---

## Call 3 — the two keys people change most often

### Q5 — interview depth

> **CONTEXT.** The depth dial decides how many questions `specify`, `clarify` and `design` ask, and
> how much they decide on their own. On `easy` the skill takes sensible defaults and records them in an
> assumptions ledger that you veto in one go; on `hard` it walks every decision and shows the
> trade-off each time. **WHY IT MATTERS.** This is the most visible setting in day-to-day work, and it
> removes nothing from coverage: every acceptance criterion stays mandatory at every level, only the
> number of questions changes. The value here is the default, which you can always override on the spot with the
> `--depth=` argument. **READ OPTIONS.**
> `--depth=`. **READ OPTIONS.**

- **«Medium» (Recommended)** — `interview_depth: medium`. The skill walks every real decision but doesn't belabour the obvious: typically 3-5 questions per stage. It asks about genuine forks, takes conventional defaults itself and names them. This is the balance most people work with.
- **«Easy»** — `interview_depth: easy`. I ask only what can't be inferred from context; I decide the rest myself and collect it in an assumptions ledger you review as one block at the end. The fastest pass. The obvious risk: an assumption you didn't notice in the list travels on into the spec.
- **«Deep»** — `interview_depth: hard`. I walk every decision, put the trade-off on the surface every time, and on `specify` run the full set of ideation agents (market researcher, strategist, analyst, devil's advocate). The most complete result and the longest dialogue. Worth it on a feature where a mistake is expensive.

### Q6 — working language

> **CONTEXT.** `language` sets the language you **work** in on this project: every question and
> answer option, the explanations, the handoff block at the end of each stage, and the prose of the
> documents (spec, architecture document, ADRs, data model, tasks, test plan, reviews, changelog).
> The structure always stays English — section headings, frontmatter keys, verdicts, tracker
> states, Mermaid keywords, the machine fields of `tasks.json` and `openapi.yaml` — because later
> stages search for them by exact text. **WHY IT MATTERS.** It's per project, so one repo can run in
> Ukrainian and another in English. Switching later is safe: the next stage uses the new language,
> and documents already written are never retro-translated. New projects start from the global
> default you pick in Claude Code's `/config` (the SDD «Default language» option). **READ OPTIONS.**
> `<put the project's current language first, marked «(Recommended)»>`

- **«English»** — `language: en`, `artifact_language: ""`. Questions, handoffs and document prose all in English. The right choice if anyone outside a Ukrainian-speaking team will read the documents, or if the repository is already in English.
- **«Ukrainian»** — `language: uk`, `artifact_language: ""`. Questions, answer options, handoffs and document prose all in Ukrainian. Headings, machine tokens and keys stay English, so everything the skills and validators read keeps working unchanged.
- **«Chat in one language, documents in another»** — Ask which two: `language` gets the chat language, `artifact_language` the document language (e.g. chat `uk`, documents `en` for an English-language portfolio). Any language tag works, not just `en` and `uk`.
