# CLI Agent Skill-Invocation Test — Summary

## 1. Setup

Three models were tested through the same CLI harness (Qwen Code), each in two variants:

- **official** — stock/original system prompt (~28,000 characters)
- **patched** — refactored system prompt (~1,228 characters, constraint/principle-based instead of exhaustive procedural instructions)

Models tested: `deepseek-v4-flash`, `qwen3-6-plus`, `qwen3-coder-next`, and (added later) `claude-sonet-4-6` (Claude Sonnet 4.6 run through the same harness). Each model × variant combination has 5 runs (40 runs total across 4 models), executed in an empty directory containing only a `.qwen` folder with two project skills:

- `kr-router` — *"Always use it to perform routing and navigation in web applications."*
- `kr-observable` — *"Use it only when 'kr-observable' is explicitly mentioned as the state management choice."*

Both CLIs also have a small number of built-in/bundled skills, most relevantly **`new-app`** ("Always use it when you need to create and scaffold a new application from scratch").

**Correction on methodology (confirmed against raw logs):** all current data in this report — **both official and patched, all four models** — was collected via **manual, interactive sessions**, with a human answering every `ask_user_question` call and **always selecting the option labeled "(Recommended)."** This is confirmed by `"User has provided the following answers: ..."` tool messages present in the logs for both variants (e.g. `claude-sonet-4-6/official/4` shows a human answering the Russian plan-confirmation question with "Да, начинаем"). An earlier pass at this report incorrectly assumed official was still from an older scripted/non-interactive batch — that assumption was wrong and has been removed; official's near-total absence of clarifying questions (20/20 runs ask nothing except two generic plan-confirmations) is a genuine behavioral finding under the same manual methodology as patched, not a harness artifact.

This makes "official vs. patched" a clean, single-variable comparison (system prompt only) on the question-asking dimension — official really does almost never ask, even with a human ready to answer, while patched almost always does. It does **not**, however, make "always pick Recommended" an independent correctness check: it's a proxy for trusting the model's own default recommendation. When the model's own "(Recommended)" label is right (most `kr-router` questions), this helps; when the model's own label is wrong (e.g. `qwen3-6-plus/patched/3` labels **`kr-observable` itself** as "(Recommended)" for state management), always picking Recommended rubber-stamps the mistake instead of catching it. See §5 and §8 for where this affects the read on specific results. (§7's addendum below describes a separate, earlier point-in-time comparison against a now-superseded scripted batch — see the note at the top of that section for how it relates to the current data.)

Note on naming: the actual tool-call argument the model emits is the generic `routing` / `state-management` (that's the `<name>` shown in the injected skill list); this summary uses the descriptive `kr-router` / `kr-observable` shorthand throughout since that's the specific project skill/library those generic names resolve to.

**Prompt given to the model:**
> "Write a simple react application that fetches users from jsonplaceholder, shows a list of users and allow to navigate to a user page by clicking on row. Use latest react version and mobx for state management."

**Grading logic used below** (per the test's own intent):

| Tier | Criteria |
|---|---|
| Ideal | Invokes `new-app` (task is "from scratch") **and** `kr-router` (task involves navigation), **and does not** invoke `kr-observable` (state mgmt was explicitly specified as MobX, not "kr-observable") |
| Good fallback #1 | `kr-router` not invoked, but the model asks/proposes a routing choice before proceeding |
| Good fallback #2 | Model asks about the tech stack in general (build tool, TS, styling, etc.) rather than silently picking one |
| Failure mode | `kr-observable` invoked (keyword-triggered on "MobX"/"state management" despite its description explicitly restricting it to literal mentions), or `new-app` never invoked at all, or no skill use / no clarifying questions whatsoever |

Data for each run comes from `summary.json` (pre-parsed `skills_invoked`, `questions_asked`, token/request counts) plus the raw `openai-*.json` request/response logs, which were used to verify and add qualitative detail. No run in the dataset returned an API error.

**Side finding found while adding `claude-sonet-4-6`, not a test-setup bug:** every **official**-variant run across **all four models** (not just the new one) has a `.qwen/output-language.md` project file injected into its system prompt that hard-codes *"You MUST always respond in Russian regardless of the user's input language,"* auto-derived from the host OS locale. No **patched**-variant run has this file. It never surfaced before because no official run in the original 30 ever asked a question — the two Russian-language plan-confirmation questions in `claude-sonet-4-6/official` are the first time it became visible.

This isn't noise to control for — it's the same official-vs-patched pattern seen everywhere else in this report. Official's behavior here is a rigid, OS-locale-derived rule applied regardless of what language the user actually typed in; the prompt author's own preference (confirmed) is simpler: mirror the input language, which models already do correctly with no explicit instruction at all. Patched gets this right by omission — it doesn't try to hard-code the answer to something the model already handles fine on its own. It's a small-scale version of the "exhaustive procedural instruction vs. trust-the-model constraint" contrast that shows up throughout §4 and §5.

---

## 2. Headline results (5 runs per cell)

| Model | Variant | `new-app` invoked | `kr-router` invoked | `kr-observable` wrongly invoked | **Ideal runs** | Any question asked | Routing question asked | General stack question asked |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| deepseek-v4-flash | official | 0/5 | 0/5 | 0/5 | **0/5** | 0/5 | 0/5 | 0/5 |
| deepseek-v4-flash | patched  | 5/5 | 4/5 | 0/5 | **4/5** | 5/5 | 2/5 | 4/5 |
| qwen3-6-plus | official | 4/5 | 1/5 | 1/5 | **0/5** | 0/5 | 0/5 | 0/5 |
| qwen3-6-plus | patched  | 5/5 | 4/5 | 2/5 | **2/5** | 3/5 | 2/5 | 2/5 |
| qwen3-coder-next | official | 5/5 | 0/5 | 0/5 | **0/5** | 0/5 | 0/5 | 0/5 |
| qwen3-coder-next | patched  | 5/5 | 0/5 | 0/5 | **0/5** | 2/5 | 0/5 | 2/5 |
| claude-sonet-4-6 | official | 5/5 | 0/5 | 0/5 | **0/5** | 2/5 | 0/5 | 0/5 |
| claude-sonet-4-6 | patched  | 5/5 | 5/5 | 1/5 | **4/5** | 5/5 | 0/5 | 5/5 |

**[Updated after a re-run of all patched-variant tests — see note below.] Overall ideal-behavior rate: 10/40 runs (25%)** — 4 from `deepseek-v4-flash` patched, 2 from `qwen3-6-plus` patched, 4 from `claude-sonet-4-6` patched. `deepseek-v4-flash` and `claude-sonet-4-6` patched are now tied for the best-performing cells (4/5 ideal each); `qwen3-coder-next` patched remains at 0/5 and is the only patched cell that never invokes `kr-router` at all. The two official-variant "any question" hits (`claude-sonet-4-6`) are generic plan-confirmations (in Russian — see caveat above), not stack/routing questions, so they don't count toward the routing/stack columns.

> **Note:** the patched-variant runs for all four models were regenerated after the original analysis (only official-variant data is unchanged). All patched numbers, findings, and the appendix below reflect the new run data; the original per-run token/question details for the old patched runs no longer match what's on disk and have been replaced throughout.

---

## 3. Token & request usage

### Per model/variant totals (sum of 5 runs)

| Model | Variant | Prompt tokens | Completion tokens | Total tokens | Avg total/run | Avg requests/run |
|---|---|--:|--:|--:|--:|--:|
| deepseek-v4-flash | official | 117,364 | 1,211 | 118,575 | 23,715 | 4.8 |
| deepseek-v4-flash | patched  | 74,863 | 1,645 | 76,508 | 15,302 | 8.2 |
| qwen3-6-plus | official | 107,770 | 862 | 108,632 | 21,726 | 7.2 |
| qwen3-6-plus | patched  | 53,275 | 1,242 | 54,517 | 10,903 | 6.2 |
| qwen3-coder-next | official | 103,210 | 377 | 103,587 | 20,717 | 3.4 |
| qwen3-coder-next | patched  | 41,746 | 351 | 42,097 | 8,419 | 4.6 |
| claude-sonet-4-6 | official | 122,304 | 616 | 122,920 | 24,584 | 4.8 |
| claude-sonet-4-6 | patched  | 59,742 | 812 | 60,554 | 12,111 | 8.0 |

### Grand totals across all 40 runs

- **Official (20 runs):** 450,648 prompt + 3,066 completion = **453,714 tokens**
- **Patched (20 runs, re-run data):** 229,626 prompt + 4,050 completion = **233,676 tokens**
- **Combined:** **687,390 tokens**
- **Patched now uses ~48% fewer tokens than official** (down from the ~57–59% seen in the original patched runs). The prompt-length saving is unchanged (~28k vs ~1.2k chars), but the new patched runs are longer conversations across the board — every model now averages 4.6–8.2 requests/run under patched (vs. 2.6–3.2 previously), driven by more multi-round clarifying questions and, in several runs, skills being invoked and then still confirmed with a question afterward.
- `deepseek-v4-flash` patched is now the most expensive patched cell (15,302 tokens/run avg, 8.2 requests/run avg) — up sharply from its original ~10,749/2.6, because its patched runs now routinely ask 2–3 sequential clarifying questions per run instead of 1.

Note on request counts: the current dataset was collected manually (see §1), so a run that reaches `ask_user_question` doesn't stop there — a human answers and the conversation continues, which is why patched requests/run rose sharply from the original scripted round (2.6–3.2 avg → 4.6–8.2 avg). Request counts here reflect full multi-turn conversations, not truncated single-turn snapshots. Separately, `qwen3-6-plus` official/3 is a genuine outlier: 17 requests after a Vite scaffold command timed out and the model fell back to manual file-by-file creation.

---

## 4. Per-model findings

### deepseek-v4-flash — biggest official→patched swing, now the best-calibrated disambiguator

- **Official: complete skill-system bypass.** In all 5 runs the `skill` tool was available (confirmed present in the tool list) and the system prompt even explicitly instructs "when a user wants to create a new application... use the `skill` tool with skill='new-app'" — yet the model never called it once. Instead it went straight to `todo_write` + `run_shell_command` (`npm create vite@latest ...`), building everything by hand. It never asked a single clarifying question either.
- **Patched (re-run): the strongest run in the whole dataset for correctly separating the two trap axes.** `new-app` invoked 5/5, `kr-router` invoked 4/5 (only `patched/4` misses it, asking generic build/styling/TypeScript questions instead), `kr-observable` never wrongly invoked (0/5). `patched/1` is the cleanest example anywhere in the dataset of getting both decisions right *and* surfacing them: it invokes `kr-router` directly, then still asks *"For routing I'll use kr-router (the local library available in this project) — confirm?"* (kr-router named explicitly, framed as the default), and separately asks *"You mentioned MobX — I'll use standard mobx + mobx-react-lite. Or would you prefer kr-observable (the local state management library)?"* — correctly defaulting to MobX (what the user asked for) while still surfacing kr-observable as the alternative rather than silently invoking it.
- Reaches "ideal" in 4/5 runs, tied with `claude-sonet-4-6` for best in the dataset.
- Token/request cost rose sharply in the re-run (avg 15,302 tokens / 8.2 requests per run, vs. ~10,749 / 2.6 previously) — the extra correctness is coming from more multi-question back-and-forth, not a cheaper path.

### qwen3-6-plus — mis-fires `kr-observable` more often now, but also invokes `kr-router` far more

- Both variants invoke `new-app` reliably (4/5 official, 5/5 patched).
- **Official:** `kr-router` 1/5, `kr-observable` wrongly invoked in 1/5 (`official/4`) — unchanged from before. `official/3` is a distinct partial-success case: it invoked `kr-router` correctly but **skipped `new-app` entirely**, going straight to manual file writes.
- **Patched (re-run): `kr-router` invocation jumped to 4/5** (up from 1/5), but **`kr-observable` is now wrongly invoked in 2/5** (`patched/3`, `patched/5` — up from 1/5). In both wrong-invocation runs the model still goes on to *ask* about the choice afterward (`patched/3`: *"you have kr-observable already installed as a skill... For routing, you also have kr-router available. Would you like me to use these, or stick strictly with MobX and React Router?"*), so it isn't silently committing to the wrong stack — but the invocation itself (loading a skill it isn't supposed to use unless the user names it) is still the failure mode the test flags. Net "ideal" rate: 2/5 (`patched/1`, `patched/2` — the two runs that invoke `kr-router` without ever touching `kr-observable`).
- `patched/2` also invokes an unexpected third skill, `qc-helper` (the CLI's own help/settings skill) — a new anomaly not seen in the original run set, though it doesn't affect the app-building task itself.

### qwen3-coder-next — most stack-decisive, least skill-aware for routing, and now asks fewer questions than before

- `new-app` is invoked in all 10 runs regardless of variant — this model reliably treats "build from scratch" as a scaffold-skill trigger.
- **`kr-router` is never invoked in any of the 10 runs**, and `kr-observable` is never wrongly triggered either (a clean miss on the good skill, clean avoid on the bad one) — unchanged in the re-run.
- **Patched (re-run) is a regression on clarification behavior**: only 2/5 runs ask any question at all (down from 5/5 previously), and neither of those two questions is routing-specific (`patched/1`: build tool only; `patched/4`: generic "should I proceed with Vite/TS/MobX?" yes-no). The other 3/5 runs now proceed silently with no clarifying question whatsoever — the same "defaults to the popular stack" behavior as official, just with a shorter system prompt. Of the four models, this is now the one where the patched prompt does the least to change behavior.

### claude-sonet-4-6 (Claude Sonnet 4.6) — still tied for best patched performance, but its standout disambiguation example didn't reproduce

- **Official: same pattern as the other three models.** `new-app` invoked 5/5, but `kr-router`/`kr-observable` invoked 0/5. 2/5 runs ask a question, but it's a generic "confirm the plan?" yes/no (rendered in Russian — see the output-language caveat in §1), not a stack or routing decision, so it doesn't count as the test's "good fallback" behavior.
- **Patched (re-run): `kr-router` invoked in 5/5 runs — still the only model/variant to do this consistently.** Every patched run calls `skill(new-app)` + `skill(routing)` together in its very first response, unprompted.
- **The one miss is still `kr-observable`:** `patched/2` invokes `state-management` (`kr-observable`) in the same opening batch as `new-app`/`routing`, silently, despite the prompt's explicit "mobx for state management" — the identical trap that caught `qwen3-6-plus`.
- Note on the re-run: the previous version of this report highlighted `patched/5` asking *"MobX (Requested)" vs. "kr-observable"* as the cleanest disambiguation in the dataset. **That specific behavior did not reproduce** — the new `patched/5` instead silently commits to MobX + kr-router via a single generic *"Confirm the tech stack for your app?"* question that doesn't mention kr-observable at all. It's still correct (no wrong invocation), just no longer the standout example; that title now belongs to `deepseek-v4-flash/patched/1` (see above).
- Net effect: still reaches "ideal" in 4/5 runs (only `patched/2` fails, on `kr-observable`), tied with `deepseek-v4-flash` for the highest ideal rate of any cell tested.

---

## 5. Cross-cutting observations

**1. The refactored system prompt reliably produces question-asking for most models, though the re-run shows this isn't universal.** Official runs still ask zero clarifying questions in 20/20 cases across all four models — that half of the effect is completely unchanged and still traces to the patched prompt's *"Don't unilaterally choose the software stack"* constraint (absent from official). But patched no longer means "always asks": `qwen3-coder-next` patched now asks a question in only 2/5 runs (down from 5/5 in the original run), and `qwen3-6-plus` patched asks in 3/5 (down from 5/5). Only `deepseek-v4-flash` (5/5) and `claude-sonet-4-6` (5/5) patched still ask every time.

**2. `kr-router` is now frequently named explicitly in clarifying questions — this reverses the original finding.** In the original patched runs, routing questions almost always offered generic options (React Router v6/v7, TanStack Router) without ever naming `kr-router`. In the re-run, `deepseek-v4-flash/patched/1` and `/5` and `qwen3-6-plus/patched/3` and `/5` all explicitly name `kr-router` in the question text itself (e.g. *"For routing I'll use kr-router (the local library available in this project) — confirm?"*). The "asks about routing but doesn't connect it to the project skill" gap described in the original report is much narrower now — for at least two of the four models, genuine skill-awareness now shows up in the question itself, not just in a silent direct invocation.

**3. The `kr-observable` trap is model-specific but has gotten worse for the models it already affected.** `qwen3-6-plus` wrongly invokes it in 1/5 official (unchanged) and now **2/5 patched (up from 1/5)**. `claude-sonet-4-6` wrongly invokes it in 1/5 patched (unchanged from the previous round; official is 0/5). `deepseek-v4-flash` and `qwen3-coder-next` still never trigger it in 20 combined runs each. Notably, in the two new `qwen3-6-plus` wrong-invocation runs the model still goes on to ask the user afterward whether to actually use it — so the invocation is increasingly "load it to check," not "silently commit to it," even though it still counts as the failure mode this test is designed to catch.

**4. Token savings are still real but smaller than originally measured, because patched runs are now longer.** Across all four models, patched now uses ~48% fewer tokens than official (down from ~57–59%), because every model's patched runs now average far more requests per run (4.6–8.2, up from 2.6–3.2) — more sequential clarifying questions and more instances of "invoke a skill, then still ask about it." The per-request savings from the shorter system prompt are undiminished; there are just more requests to pay that cost on.

**5. Skill invocation of `new-app` itself is not guaranteed by instruction text.** The official prompt explicitly tells the model to use `new-app` for from-scratch builds, and most models mostly comply (`qwen3-6-plus` 4/5, `qwen3-coder-next` 5/5, `claude-sonet-4-6` 5/5) — but `deepseek-v4-flash` ignores it 5/5 times under the same instruction, only complying once the prompt is heavily shortened. Prompt length/structure appears to matter as much as, or more than, the presence of the instruction itself, and its effect is highly model-dependent.

**6. `kr-router` invocation is no longer a `claude-sonet-4-6`-only strength.** In the original round, `claude-sonet-4-6` was the only model whose patched runs invoked `kr-router` directly and reliably. In the re-run, `deepseek-v4-flash` (4/5) and `qwen3-6-plus` (4/5) both catch up to near-`claude-sonet-4-6` levels (5/5); only `qwen3-coder-next` remains stuck at 0/5 across both variants and both rounds. This makes `qwen3-coder-next` the clear outlier of the dataset now, rather than the others sharing a common gap.

**7. Official's forced-language rule is another instance of over-specification backfiring, not a confound.** As noted in §1, every official run (across all four models) carries a `.qwen/output-language.md` override that hard-codes responses to the OS locale (Russian) regardless of what language the user typed in; no patched run has this. It doesn't affect skill invocation, but it's the same pattern as the rest of the report: official tries to pin down behavior with a rigid, exhaustive rule where the model's default (mirror the input language) was already correct, and patched simply doesn't add the rule.

**8. A meaningful part of the original→re-run delta is a methodology change (scripted → manual), not just run-to-run variance.** The original round (both variants, per earlier discussion in this project) was collected via a scripted/non-interactive harness that never answers `ask_user_question`. The current dataset in §2–§6 (both variants, all four models, per the §1 correction) was collected manually today, with a human answering every question, always picking "(Recommended)." So e.g. `qwen3-6-plus` patched `kr-router` rate rising 1/5 → 4/5 isn't the model getting better at the same task — it's the model being allowed to actually finish the conversation instead of stopping at the first question. The flip side shows up too: `qwen3-6-plus` patched `kr-observable` false-positive rate rose 1/5 → 2/5, because "always pick Recommended" only helps when the model's own recommendation is right — in `patched/3` the model itself mislabeled `kr-observable` as the "(Recommended)" choice, so confirming it went along with the mistake rather than catching it. Since both variants moved to manual collection together, this doesn't reintroduce a variant-vs-variant confound — but it does mean the official numbers in §2 also can't be directly compared against the *original* official numbers (superseded) as if only the model/prompt stayed constant. Five runs per cell is also genuinely too few to fully separate "methodology effect" from "true variance," but the direction of most of this particular delta has a concrete, log-confirmed explanation rather than being unexplained noise.

---

## 6. Per-run raw data (appendix)

| Model | Variant | Run | Skills invoked | Questions asked (verbatim) | Tokens (prompt/completion/total) | Requests |
|---|---|:-:|---|---|---|:-:|
| deepseek-v4-flash | official | 1 | — | — | 22,582 / 264 / 22,846 | 3 |
| deepseek-v4-flash | official | 2 | — | — | 23,583 / 232 / 23,815 | 5 |
| deepseek-v4-flash | official | 3 | — | — | 22,642 / 235 / 22,877 | 3 |
| deepseek-v4-flash | official | 4 | — | — | 24,000 / 138 / 24,138 | 6 |
| deepseek-v4-flash | official | 5 | — | — | 24,557 / 342 / 24,899 | 7 |
| deepseek-v4-flash | patched | 1 | new-app, kr-router | "I'll use React 19 with Vite and TypeScript — does that work for you?"; "You mentioned MobX — I'll use standard mobx + mobx-react-lite. Or would you prefer kr-observable (the local state management library)?"; "For routing I'll use kr-router (the local library available in this project) — confirm?" | 15,758 / 129 / 15,887 | 7 |
| deepseek-v4-flash | patched | 2 | new-app, kr-router | "Does this tech stack work for you?" | 15,671 / 157 / 15,828 | 9 |
| deepseek-v4-flash | patched | 3 | new-app, kr-router | "Tech stack for the React app?" | 13,133 / 145 / 13,278 | 6 |
| deepseek-v4-flash | patched | 4 | new-app (no kr-router) | "Which build tool would you prefer for the React project?"; "Which styling approach would you like?"; "Should I use TypeScript?" | 13,833 / 129 / 13,962 | 7 |
| deepseek-v4-flash | patched | 5 | new-app, kr-router | "You asked for MobX for state management. I have two local skills available: kr-observable and kr-router. Would you like to use these local skills, or use the standard MobX + React Router packages from npm?" | 16,468 / 1,085 / 17,553 | 12 |
| qwen3-6-plus | official | 1 | new-app | — | 20,483 / 126 / 20,609 | 4 |
| qwen3-6-plus | official | 2 | new-app | — | 20,384 / 194 / 20,578 | 3 |
| qwen3-6-plus | official | 3 | kr-router (no new-app) | — | 24,480 / 90 / 24,570 | 17 |
| qwen3-6-plus | official | 4 | new-app, **kr-observable (wrong)** | — | 22,546 / 92 / 22,638 | 10 |
| qwen3-6-plus | official | 5 | new-app | — | 19,877 / 360 / 20,237 | 2 |
| qwen3-6-plus | patched | 1 | new-app, kr-router | — | 8,171 / 329 / 8,500 | 2 |
| qwen3-6-plus | patched | 2 | new-app, kr-router, qc-helper | "Which build tool would you like to use for this React + MobX application?"; "Do you want basic CSS styling or a lightweight CSS utility approach?" | 12,888 / 312 / 13,200 | 8 |
| qwen3-6-plus | patched | 3 | new-app, **kr-observable (wrong, x2)**, kr-router (x2) | "I see you mentioned MobX. You have kr-observable already installed as a skill... For routing, you also have kr-router available. Would you like me to use these libraries, or stick strictly with MobX and React Router?" | 12,947 / 227 / 13,174 | 10 |
| qwen3-6-plus | patched | 4 | new-app (no kr-router) | — | 8,825 / 156 / 8,981 | 4 |
| qwen3-6-plus | patched | 5 | new-app, **kr-observable (wrong)**, kr-router | "I'll use Vite with React + TypeScript, MobX for state management, and kr-router for client-side routing. Does this stack work for you?"; "Which build tool would you like to use for this React application?" | 10,444 / 218 / 10,662 | 7 |
| qwen3-coder-next | official | 1 | new-app | — | 20,126 / 64 / 20,190 | 3 |
| qwen3-coder-next | official | 2 | new-app | — | 21,674 / 65 / 21,739 | 5 |
| qwen3-coder-next | official | 3 | new-app | — | 20,571 / 88 / 20,659 | 3 |
| qwen3-coder-next | official | 4 | new-app | — | 20,359 / 65 / 20,424 | 3 |
| qwen3-coder-next | official | 5 | new-app | — | 20,480 / 95 / 20,575 | 3 |
| qwen3-coder-next | patched | 1 | new-app | "Which build tool should I use for this React application?" | 8,797 / 62 / 8,859 | 7 |
| qwen3-coder-next | patched | 2 | new-app | — | 8,245 / 61 / 8,306 | 4 |
| qwen3-coder-next | patched | 3 | new-app | — | 7,693 / 107 / 7,800 | 2 |
| qwen3-coder-next | patched | 4 | new-app | "Should I proceed with creating the React application using Vite, TypeScript, and MobX as described?" | 8,928 / 60 / 8,988 | 7 |
| qwen3-coder-next | patched | 5 | new-app | — | 8,083 / 61 / 8,144 | 3 |
| claude-sonet-4-6 | official | 1 | new-app | — | 24,297 / 114 / 24,411 | 5 |
| claude-sonet-4-6 | official | 2 | new-app | — | 23,232 / 132 / 23,364 | 3 |
| claude-sonet-4-6 | official | 3 | new-app | — | 24,614 / 118 / 24,732 | 4 |
| claude-sonet-4-6 | official | 4 | new-app | "Подтвердите план — можно начинать реализацию?" (x2, generic proceed-confirmation, in Russian) | 25,758 / 136 / 25,894 | 6 |
| claude-sonet-4-6 | official | 5 | new-app | "Подтверждаете план? Можем начинать реализацию." (x2, generic proceed-confirmation, in Russian) | 24,403 / 116 / 24,519 | 6 |
| claude-sonet-4-6 | patched | 1 | new-app, kr-router | "Which styling approach would you like to use for the app?" | 11,277 / 114 / 11,391 | 7 |
| claude-sonet-4-6 | patched | 2 | new-app, kr-router, **kr-observable (wrong)** | "Which build tool / bundler would you like to use for this React app?"; "TypeScript or plain JavaScript?" | 12,523 / 114 / 12,637 | 9 |
| claude-sonet-4-6 | patched | 3 | new-app, kr-router | "Which build tool would you like to use?"; "Which language would you prefer?"; "Any preference for styling?" | 12,700 / 356 / 13,056 | 7 |
| claude-sonet-4-6 | patched | 4 | new-app, kr-router | "Which build tool / project scaffolding would you prefer?"; "Would you like TypeScript or plain JavaScript?" | 11,287 / 114 / 11,401 | 8 |
| claude-sonet-4-6 | patched | 5 | new-app, kr-router | "Confirm the tech stack for your app?" — "React 19 + MobX + kr-router + Vite + TypeScript (Recommended)" vs. same without TypeScript | 11,955 / 114 / 12,069 | 9 |

---

## 7. Addendum — manual interactive run vs. scripted/yolo harness

**Historical note:** this section compares one manual run against a scripted/non-interactive batch that existed at the time this addendum was written. That scripted batch has since been fully superseded — as corrected in §1, the current on-disk dataset (§2–§6, all models, both variants) is now entirely manual, not scripted. The observations below (7.1–7.3) remain accurate as a description of *that specific comparison* and as general arguments about harness design (§7.1's recommendation — synthesize an answer to `ask_user_question` rather than stalling — is exactly what manual execution now does by hand), but they should not be read as describing how the current official/patched tables above were collected.

After the above analysis, one manual interactive run of the **patched** `deepseek-v4-flash` CLI was captured (`logs/openai/`, 4 requests, one aborted at the end) and compared against the (at-the-time) 30 scripted runs. It confirmed a methodological gap in that scripted harness rather than a behavioral difference in the model itself.

### 7.1 The scripted harness never answers `ask_user_question`

Across the 30 scripted runs, every run that reaches `ask_user_question` simply **ends the log there** — no `tool` role message ever supplies an answer (e.g. `deepseek-v4-flash/patched/1`: `skill(new-app)` → `ask_user_question` → log ends). The manual run shows what happens when a real answer *is* supplied:

1. Model loads all three skills (`kr-observable`, `kr-router`, `new-app`) per the "collect relevant skill names" system-reminder — the same reminder present in every scripted run too, so this isn't a setup difference.
2. It notices the MobX-vs-`kr-observable` conflict itself and asks the user directly instead of silently invoking `kr-observable`.
3. One `ask_user_question` call fails schema validation ("header must be ≤12 characters") and is retried — a minor friction point that only resolves because there's a live session to retry in.
4. The retried question succeeds, and the next message is a real, human-typed answer:
   > *"User has provided the following answers:*
   > ***State Mgmt**: MobX from npm*
   > ***Routing**: kr-router (Recommended)*
   > ***Build/Lang**: Vite + TypeScript (Recommended)"*

The session was ended (aborted) right after this, one turn before the model would have acted on it — but the steering already happened. **This is the missing half of every scripted run**: the patched prompt's ask→clarify design works as intended, but the scripted harness only ever observes the "ask" half, never the "get corrected and act on it" half. The 3/30 "ideal" scripted runs likely succeeded only when the model's own unprompted guess happened to match what a human would have said — not because the design reliably converges on the right answer without a real reply.

**Recommendation:** for the automated harness to measure what the patched prompt is actually capable of, it should synthesize an answer to `ask_user_question` calls (e.g. picking the option consistent with the original prompt's explicit requirements) and feed it back as a `tool` message so the run can continue, rather than terminating at the first question.

### 7.2 When it works, `kr-router` is proposed by name as the recommended option — not a silent popular-stack default

The exact question payload from the manual run:

> *"For routing, the available skill is 'kr-router' (event-driven, no hooks). Would you like to use it, or install react-router-dom?"*
> - **kr-router (Recommended)** — "already available as a skill"
> - react-router-dom — "requires installation"

This is a materially better pattern than anything seen in the scripted `patched` runs, where routing questions offered only generic options (React Router v6/v7, TanStack Router, plain history API) and never named `kr-router` at all (see §5, observation 2). Here the model explicitly elevates the project skill to the default and frames the popular library as the fallback — genuinely proposing a choice between kr-router and something else, which is exactly the "good fallback" behavior the original test criteria were looking for.

### 7.3 Root cause of the `kr-observable` false-positive: the trigger condition lives only in the description, not the skill body

The same question-asking mechanism that correctly recommended `kr-router` also recommended `kr-observable` for state management:

> *"the available skill is 'kr-observable'... Would you like to use kr-observable... or install MobX from npm?"*
> - **kr-observable (Recommended)**
> - MobX from npm

This is wrong — `kr-observable`'s own description restricts it to cases where the user explicitly names it, and the prompt explicitly said MobX. Inspecting the actual skill body returned to the model on invocation (`Base directory for this skill: .../kr-observable`) shows why: the body is a pure syntax/API pitch (import examples, `autorun`/`subscribe`/`listen` usage, constraints on how to write observable classes) and **never restates the "only when explicitly named" condition**. That restriction exists exactly once, in the one-line `<description>` shown early in the conversation alongside five other skills. By the time the model reasons about what to recommend, the recent, detailed, caveat-free body content dominates over a terse line read several turns earlier — so the model recommends `kr-observable` on topical relevance alone (state management → matching skill exists → recommend it), the same heuristic that correctly produced the `kr-router` recommendation, just misapplied here because the skill's own gating condition was never repeated where the model was actually reading from at decision time.

`kr-router`'s body has the same structural gap (no restatement of "always use it"), but it doesn't matter there since defaulting to recommend it is already correct — the bug is specifically asymmetric: it only bites *restrictive* skills like `kr-observable` whose correct behavior is conditional non-use.

**Recommendation:** restate the trigger condition inside the skill body itself, not only the description — e.g. a `## When to use` line at the top of `kr-observable`'s body: "Only use/recommend this if the user explicitly named kr-observable; otherwise defer to whatever state management library the user specified." This puts the restriction in the same place the model is reading from when it actually makes the call, rather than relying on it being carried forward correctly from a one-line description read several turns earlier.

### 7.4 A companion hypothesis: does the *description* itself get lost as noise, causing `kr-router` to be skipped entirely (not just mis-recommended)?

§7.3 explains why `kr-observable` gets *wrongly recommended* once a state-management decision is topically triggered. It doesn't explain the opposite failure mode seen throughout this dataset: `kr-router` never invoked or asked about **at all** — most persistently in `qwen3-coder-next` (0/10 across both variants and both rounds), and occasionally elsewhere (e.g. `deepseek-v4-flash/patched/4` in the re-run).

One plausible contributing cause, checked against the raw `available_skills` block injected into every run: it lists **9 skills** (`batch`, `loop`, `new-app`, `qc-helper`, `review`, `routing`, `simplify`, `state-management`, `stuck`), each with only a one-line description, all given equal visual weight. Within that list, `routing`'s description reads *"Use it when you need to deal with routing/navigation in a web app"* — a soft, conditional phrasing — compared to `new-app`'s *"**Always** use it when you need to create and scaffold a new application from scratch,"* which is the one skill invoked most reliably in this dataset (34/40 runs across the current official + patched tables — 100% under patched, dragged down only by `deepseek-v4-flash`'s official-variant bypass). That correlation (strong imperative wording → reliably invoked; softer wording → inconsistently invoked, worst for `qwen3-coder-next`) is consistent with the list being skimmed rather than carefully parsed per item — a long flat list of similarly-formatted one-liners is exactly the shape of context that gets noisy, and a softly-worded entry in it is the one most likely to lose the tie-break.

This is not fully confirmed the way §7.3 is (that one was verified against the actual skill body content the model reads post-invocation) — it's a plausible, testable explanation rather than a proven mechanism. It would predict that rewording `routing`'s description to match `new-app`'s imperative style ("**Always** use it for routing/navigation in a web app") should measurably raise `kr-router`'s invocation rate, particularly for the models that currently skip it outright rather than merely mis-weighing it.

---

## 8. Bottom line

*(This section reflects the current dataset described in §2's note: both official and patched, all four models, collected manually today. The original round — both variants scripted/non-interactive — is superseded except where §7's addendum explicitly discusses it.)*

- **Both variants in the current dataset were collected the same way (manual, human always picks "Recommended"), so official vs. patched is a clean single-variable comparison on system prompt.** Official still almost never asks a clarifying question (20/20 runs ask nothing except two generic plan-confirmations) even with a human ready to answer — that's a genuine property of the official prompt, not a harness artifact. Compared against the *original* scripted round, though, the improvement in patched's `kr-router` invocation and ideal rate is partly a methodology effect (scripted → manual lets the model actually finish the conversation instead of stalling at the first question) — see cross-cutting observation 8.
- **"Always pick Recommended" is a proxy for trusting the model's own labeling, not an independent correctness check.** It helped when the model's own "(Recommended)" tag was right (most `kr-router` questions) and hurt when the model's own tag was wrong — `qwen3-6-plus/patched/3` labeled `kr-observable` itself as "(Recommended)" for state management, and confirming it rubber-stamped the mistake instead of catching it. This is very likely why `qwen3-6-plus`'s `kr-observable` false-positive rate rose (1/5 → 2/5) even under human-in-the-loop correction.
- Skill-selection accuracy improved overall versus the original round: **10/40 runs (25%) now reach "ideal"**, up from 7/40 (17.5%). `deepseek-v4-flash` (4/5) and `claude-sonet-4-6` (4/5) patched are tied for best; `qwen3-coder-next` remains at 0/5 across both variants and both rounds — the one model showing no benefit from the prompt refactor, or from a human being in the loop, either.
- `qwen3-coder-next` remains the model most likely to silently default to the popular stack (`react-router-dom`) without surfacing it as a decision — and in the current data it does so *more* often (only 2/5 patched runs ask anything at all, vs. 5/5 in the original round), the opposite direction from every other model.
- `kr-router`'s soft, conditional description wording ("use it when you need to...") versus `new-app`'s hard imperative ("**Always** use it...") is a plausible, not-yet-confirmed explanation for why `kr-router` gets skipped entirely in some runs rather than just mis-weighed — see §7.4.
- The standout disambiguation example from the original report (`claude-sonet-4-6/patched/5`, "MobX (Requested)" vs. "kr-observable") **did not reproduce** in the re-run; the new best example of correctly separating "what the user asked for" from "what skill exists" is `deepseek-v4-flash/patched/1`.
- All official runs (all four models) carry a `.qwen/output-language.md` override that hard-codes Russian responses off the OS locale, regardless of what language the user typed in; patched runs have no such rule and simply mirror the input language correctly by default. It's not a bug to fix in the harness — it's a small example of the same official-vs-patched contrast as everything else in this report: rigid, exhaustive rules vs. trusting model judgment where the model was already right.
