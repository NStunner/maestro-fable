# Maestro Fable

A Claude Code skill that turns Fable into a conductor: **Fable plans, routes and reviews; Opus and Sonnet subagents do the work.** Model and effort are picked per task difficulty, and the routing rules adapt to each project over time, with every rule change approved by you. When Fable quota runs out, or when you ask, Opus takes over as manager under the same rules.

*Türkçe açıklama aşağıda.*

## Why

Fable is the strongest model but has the tightest quota. Maestro Fable spends Fable only on decisions and quality control, and hands the actual implementation to Opus (most work) or Sonnet (trivial work). You keep Fable-level judgement at a fraction of the Fable usage.

## How it works

1. **Score.** Each subtask gets a 0-10 difficulty score across five axes: scope, ambiguity, risk, reasoning, verifiability.
2. **Route.**

   | Score | Model | Effort |
   |---|---|---|
   | 0-1 | sonnet | low |
   | 2-3 | sonnet | medium |
   | 4-5 | opus | medium |
   | 6-7 | opus | high |
   | 8 | opus | xhigh |
   | 9-10 | opus | max |

   Floors apply on top of the sum: high risk means at least `opus/medium`, hard reasoning (concurrency, algorithms, architecture) means at least `opus/high`. High-risk work (auth, payments, migrations, RLS, multi-tenant isolation) never goes to Sonnet. Easy-to-verify, low-risk work starts one effort level lower and is re-run one level up if tests fail. Small neighbouring tasks are bundled into one brief so delegation overhead doesn't eat the savings.
3. **Brief.** Every subagent gets the user's request verbatim, acceptance criteria, what must still fail, scope limits, stop conditions (brief contradicts the code, verification fails twice, out-of-scope files needed, a permission denial) and a report format: status (done / concerns / blocked), a short summary with headings, changed files, test results with failing names and error summaries, and file:line evidence for key claims. Agent descriptions show the route, e.g. `Opus · high: add webhook handler`.
4. **Review.** Fable reads the full `git diff`, runs tests, lint and typecheck, and checks correctness, scope, consistency, security, performance and simplicity. For score 6+ or high-risk work an independent read-only reviewer (your project's own reviewer if it has one, otherwise an Opus reviewer that doesn't see the worker's report) looks first. Fixes go back to the same agent; after two failed rounds the task is escalated; if `opus/max` still fails, Fable stops and asks you. Infrastructure errors (rate limits, timeouts) and refusals are reported to you instead of escalated, and logged separately so learning isn't skewed.
5. **Learn.** Results are logged per project, with a task id and approximate tokens. When a pattern shows up at least three times across at least two different tasks (e.g. Sonnet keeps getting escalated in `billing/`, or `opus/high` always passes first try on UI work), Fable proposes a rule change with evidence. Rules record the model version they were validated on, and are re-checked when a model alias moves to a new release. **Nothing is written to the rules until you approve.**

### Manager mode: Fable or Opus

Fable manages by default. When Fable quota is running low or out, Maestro Fable warns you and offers three options: switch the session to the latest Opus (`/model opus`, `/effort max` or `xhigh`) and continue from the saved plan, wait for the reset, or keep going on Fable. You can also ask for Opus as manager at any time. The main session model is checked at the start of every task, since Fable can fall back to Opus silently.

### Works with your project's workflow

Maestro Fable doesn't replace your project's own agents, reviewer gates, security checks or commit steps. It adds model/effort routing and manager-level review on top: it calls the agents your project prescribes, always with an explicit `model` and `effort` (an agent without `model` would inherit the main session model, i.e. Fable), and keeps every gate your `CLAUDE.md` requires.

### Per-project memory

```
.claude/maestro-fable/
  profile.md   # stack, modules, risky areas, test/lint/typecheck commands, your workflow and agents
  rules.md     # project rules you approved
  log.md       # outcome of every subtask
  plan.md      # the running task's plan and status (survives compaction and quota resets)
```

On first use Fable offers a **full project analysis** (Sonnet maps the repo cheaply, Fable reads the risky core files) and proposes project-specific rules, or a **quick start** that learns as it goes. Both run a setup check: environment variables that override per-call model/effort (`CLAUDE_CODE_SUBAGENT_MODEL[_FORCE]`, `CLAUDE_CODE_EFFORT_LEVEL`), your project's agents and gates, and whether your Claude Code version honours `effort`. Commit the folder to share rules with your team, or add it to `.gitignore` to keep them personal.

### Locked rules

Learning can never change these: the manager does not write implementation code (reading code is fine and is how review works), except tiny edits it may make itself: score 0-1, no risk, one file, at most 5 lines, no new files, dependencies, tests or logic changes; every subagent call sets `model` and `effort`, high-risk work never goes to Sonnet, forks are never used for implementation (they run on the main model), and rule changes always need your approval.

## Install

**As a plugin (recommended):**

```
/plugin marketplace add NStunner/maestro-fable
/plugin install maestro-fable@maestro-fable
```

**As a plain skill:** copy `skills/maestro-fable/` into `~/.claude/skills/` (all projects) or `<project>/.claude/skills/` (one project), so the file ends up at `.../skills/maestro-fable/SKILL.md`.

## Use

1. Start the session on Fable: `/model` → Fable. A medium effort (`/effort medium`) is usually enough for orchestration.
2. Ask for the work, e.g. *"Use maestro-fable to add Stripe webhooks for subscription renewals."*
3. Say *"review the rules"* any time for a full pass over the log and a batch of proposals.
4. Say *"use Opus as manager"* to run the same workflow with Opus in charge.

Requires a Claude Code version whose `Agent` tool supports the `model` and `effort` parameters, and a plan with Fable access (or Opus, for Opus manager mode).

## Changelog

- **1.1.0:** Opus manager mode and quota warning; tiny-edit exception (the manager fixes score 0-1, risk-free, one-file edits of up to 5 lines itself instead of spawning an agent); works alongside project workflows and reviewers; score floors, cheap-first runs and task bundling; stronger briefs with stop conditions and structured reports; independent second reviewer for risky work; infrastructure errors and refusals reported instead of escalated; `plan.md` for resuming; cross-task evidence, token column and model-version tracking for learning; worktree and fork safeguards.
- **1.0.0:** First release.

---

## Türkçe

Maestro Fable, Fable'ı bir şef gibi kullanan bir Claude Code skill'i: **Fable planlar, yönlendirir ve denetler; işi Opus ve Sonnet alt ajanları yapar.** Her iş için model ve effort zorluk puanına göre seçilir. Yönlendirme kuralları her projede sonuçlara göre gelişir, ama hiçbir kural değişikliği sen onaylamadan yazılmaz. Fable kotası biterse ya da sen istersen yönetimi aynı kurallarla Opus üstlenir.

- **Puanlama:** Kapsam, belirsizlik, risk, akıl yürütme ve doğrulanabilirlik eksenlerinde 0-10; riskli ya da zor akıl yürütme gerektiren işler için taban seviyeler.
- **Yönlendirme:** Kolay işler Sonnet'e, geri kalanı Opus'a; effort puana göre. Puanı 0-1, risksiz, tek dosyada en çok 5 satırlık düzenlemeleri yönetici ajan açmadan kendisi yapar. Riskli işler asla Sonnet'e gitmez. Kolay doğrulanan işler bir alt kademeden başlar, test düşerse bir üst kademede yeniden çalışır; küçük işler tek brief'te birleşir.
- **Brief:** Kullanıcının isteği aynen, kabul kriterleri, durma koşulları ve yapılanların kısa özetini, test sonuçlarını ve kanıtları içeren rapor biçimi.
- **Denetim:** Fable diff'i okur, test/lint/typecheck çalıştırır, gerekirse düzelttirir veya üst seviyeye çıkarır. Riskli işlerde önce bağımsız bir denetçi bakar (projenin kendi reviewer'ı varsa o). Altyapı hataları ve retler yükseltilmez, sana bildirilir.
- **Yönetici modu:** Fable kotası azalınca uyarır; en güncel Opus'a (`/model opus`, `/effort max` ya da `xhigh`) geçme, bekleme ya da Fable'la sürdürme seçeneği sunar. İstediğin zaman Opus'u yönetici yapabilirsin.
- **Projenle uyum:** Projenin kendi ajanlarını, reviewer kapısını ve commit adımlarını değiştirmez; onlara model/effort seçimi ve denetim ekler.
- **Öğrenme:** Aynı desen en az 3 kayıtla ve en az 2 farklı görevde görülünce kanıtıyla kural önerir; Onayla / Reddet / Düzenle diye sorar.
- **İlk kurulum:** İstersen proje komple analiz edilir ve projeye özel kurallar önerilir; ayrıca ortam değişkenleri ve projenin iş akışı denetlenir.

Kurulum ve kullanım için yukarıdaki *Install* ve *Use* bölümlerine bak.

## License

MIT
