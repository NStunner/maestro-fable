# Maestro Fable

A Claude Code skill that turns Fable into a conductor: **Fable plans, routes and reviews; Opus and Sonnet subagents do the work.** Model and effort are picked per task difficulty, and the routing rules adapt to each project over time, with every rule change approved by you.

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

   High-risk work (auth, payments, migrations, RLS, multi-tenant isolation) never goes to Sonnet.
3. **Review.** Fable reads the full `git diff`, runs tests, lint and typecheck, and checks correctness, scope, consistency, security, performance and simplicity. It sends concrete fixes back to the same agent, escalates after two failed rounds, and stops to ask you if `opus/max` still fails.
4. **Learn.** Results are logged per project. When a pattern shows up at least three times (e.g. Sonnet keeps getting escalated in `billing/`, or `opus/high` always passes first try on UI work), Fable proposes a rule change with evidence. **Nothing is written to the rules until you approve.**

### Per-project memory

```
.claude/maestro-fable/
  profile.md   # stack, modules, risky areas, test/lint/typecheck commands
  rules.md     # project rules you approved
  log.md       # outcome of every subtask
```

On first use Fable offers a **full project analysis** (Sonnet maps the repo cheaply, Fable reads the risky core files) and proposes project-specific rules, or a **quick start** that learns as it goes. Commit the folder to share rules with your team, or add it to `.gitignore` to keep them personal.

### Locked rules

Learning can never change these: Fable does not write implementation code, high-risk work never goes to Sonnet, and rule changes always need your approval.

## Install

**As a plugin (recommended):**

```
/plugin marketplace add NStunner/maestro-fable
/plugin install maestro-fable@maestro-fable
```

**As a plain skill:** copy `skills/maestro-fable/` into `~/.claude/skills/` (all projects) or `<project>/.claude/skills/` (one project).

## Use

1. Start the session on Fable: `/model` → Fable. A medium effort (`/effort medium`) is usually enough for orchestration.
2. Ask for the work, e.g. *"Use maestro-fable to add Stripe webhooks for subscription renewals."*
3. Say *"review the rules"* any time for a full pass over the log and a batch of proposals.

Requires a Claude Code version whose `Agent` tool supports the `model` and `effort` parameters, and a plan with Fable access.

---

## Türkçe

Maestro Fable, Fable'ı bir şef gibi kullanan bir Claude Code skill'i: **Fable planlar, yönlendirir ve denetler; işi Opus ve Sonnet alt ajanları yapar.** Her iş için model ve effort zorluk puanına göre seçilir. Yönlendirme kuralları her projede sonuçlara göre gelişir, ama hiçbir kural değişikliği sen onaylamadan yazılmaz.

- **Puanlama:** Kapsam, belirsizlik, risk, akıl yürütme ve doğrulanabilirlik eksenlerinde 0-10.
- **Yönlendirme:** Kolay işler Sonnet'e, geri kalanı Opus'a; effort puana göre. Riskli işler asla Sonnet'e gitmez.
- **Denetim:** Fable diff'i okur, test/lint/typecheck çalıştırır, gerekirse düzelttirir veya üst seviyeye çıkarır.
- **Öğrenme:** Aynı desen en az 3 kez görülünce kanıtıyla kural önerir; Onayla / Reddet / Düzenle diye sorar.
- **İlk kurulum:** İstersen proje komple analiz edilir ve projeye özel kurallar önerilir.

Kurulum ve kullanım için yukarıdaki *Install* ve *Use* bölümlerine bak.

## License

MIT
