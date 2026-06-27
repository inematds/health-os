# Health OS

A complete, turnkey blueprint for a **personal AI health coach**: a Telegram agent that remembers everything in its own Supabase database, reads your wearable (WHOOP) over the official API, runs a recovery-led morning review, and grounds every call in your real bloods, genetics, body composition, and goals.

Clone it, point it at your own Supabase project and bot token, work through the checklist below, and you have a coach that reviews yesterday, reads last night's recovery, and tells you whether to push or pull today.

> ⚠️ **Not medical advice.** This is a software blueprint, not health guidance. Everything here is simply what worked for one person who consulted their doctors at every step. It is not a prescription, diagnosis, or recommendation for you. Talk to qualified clinicians before changing anything about your labs, supplements, training, or diet. The AI coach can be wrong or hallucinate, so treat anything it suggests as a question to bring to your doctor, never an instruction to follow.

![Health OS system map](./docs/health-os-schematic.png)

> Everything in this repo is generic or an example. It contains **no personal health information**. Bring your own data, bot, and Supabase project. Keep your filled-in `CLAUDE.md`, your seed values, and your `~/.env` out of version control.

---

## What you get

- **A Telegram health coach** grounded in your data, not generic advice.
- **Its own Supabase database** (full schema in `agent/supabase/migrations/`): food, workouts, weigh-ins, body comp, caffeine, supplements, vitals, labs, daily check-ins, goals, context, plus semantic message memory.
- **A WHOOP connection blueprint** end to end: OAuth app, the redirect/callback, the daily sync, the JSON-to-database mapping, and the two production gotchas (Cloudflare user-agent, refresh-token rotation).
- **A recovery-led morning review** that ties yesterday's food/caffeine/training to how you recovered, and gates today's plan on the number.
- **Photo workflows** (food -> macros, lab scan -> markers, body photo -> workout).
- **The live dashboard** (`agent/dashboard/`): the exact dark look and feel, reading straight from Supabase.
- **All the diagrams** generated to convey the system, embedded throughout.

## A day in the life

A typical day, all in Telegram. Illustrative example, not real numbers, and not advice.

- **07:00** the morning check-in fires: "Recovery 71% (green), slept 7h20m, HRV 64, RHR 52. Yesterday's protein hit target and caffeine was early and modest, that clean run is why you're green. Today is a push day, put carbs around the session. Weight this morning? Any BP?"
- **08:30** you photograph breakfast: "~38g protein, low saturated fat, logged." (writes a `food_log` row)
- **13:00** you ask "coffee now?": "You're green and it's early, fine, keep it under your ceiling and cut off by mid-afternoon."
- **19:00** "steak and a glass of wine": logged; the coach notes the alcohol may dent tonight's recovery.
- **Next morning** the loop closes: "Recovery dipped to 48% amber, the wine and late meal are the likely cause. Ease off today, lean on protein, fiber, and hydration."

Every number above is illustrative. Your coach's outputs are only as good as the profile you give it, and should always be checked with your doctor.

---

---

## Architecture

The agent reads a compact **session snapshot** at the start of every turn (weight trend, today's intake, BP, last night's recovery, the 7-day sleep pattern, goals), so it always answers from current context. Data flows in from the wearable, food photos, and manual logs into Supabase; the coach reasons over it, grounded in your labs and genetics; and it delivers the morning review, the dashboard, cited advice, and supplement schedules.

![Health OS data model](./docs/health-os-data-model.png)

The recovery-led morning review the coach runs each day:

![The recovery-led morning review](./docs/health-os-morning-review.png)

See the full map above. The repo layout:

```
health-os-private/
├── README.md                  ← you are here (overview + the test checklist)
├── docs/
│   ├── BUILD_GUIDE.md         ← step-by-step build, end to end
│   ├── health-os-schematic.png
│   ├── whoop-1-setup.png
│   └── whoop-2-data.png
└── agent/                     ← the self-contained, sanitized agent
    ├── CLAUDE.md              ← the coach's brain (template, fill in your profile)
    ├── agent.yaml.example     ← bot config + slash commands
    ├── AGENTS.md
    ├── scripts/               ← state, memory, db, WHOOP, supplements, advice, ...
    ├── supabase/migrations/   ← the full schema + example seed
    └── dashboard/             ← the live web dashboard (page + data layer + routes)
```

---

## The WHOOP connection (the headline blueprint)

### One-time setup
![WHOOP setup flow](./docs/whoop-1-setup.png)

### Data path: API JSON to coach
![WHOOP data path](./docs/whoop-2-data.png)

Full walkthrough in [`docs/BUILD_GUIDE.md`](./docs/BUILD_GUIDE.md). The short version: create a WHOOP app, register `https://<your-host>/whoop/callback`, enable `read:recovery read:sleep read:cycles offline`, authorize once, and a daily cron (`agent/scripts/whoop-sync.py`) pulls recovery + sleep into the `vitals` table. It rotates the refresh token every run and sends a browser user-agent (Cloudflare bans the default Python one).

---

## Environment variables

Every secret lives in `~/.env` (your home directory, never a project `.env`, never committed):

| Variable | Purpose |
|---|---|
| `HEALTH_BOT_TOKEN` | Telegram bot token (from @BotFather) |
| `SUPABASE_URL` | `https://<project-ref>.supabase.co` |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-side DB access (bypasses RLS) |
| `SUPABASE_ANON_KEY` | Public key (RLS means it reads nothing) |
| `SUPABASE_DB_PASSWORD` | For `supabase` CLI migrations |
| `OPENAI_API_KEY` | Embeddings for semantic memory |
| `GOOGLE_API_KEY` | Vision for food/lab photos + workout clips (Gemini) |
| `DASHBOARD_TOKEN` | Gates the web dashboard |
| `WHOOP_CLIENT_ID`, `WHOOP_CLIENT_SECRET` | Your WHOOP app credentials |
| `WHOOP_REFRESH_TOKEN` | Written by the OAuth callback, rotated every sync |

---

## Quick start

1. Provision a private Supabase project; push `agent/supabase/migrations/`.
2. Create a private `health-assets` storage bucket.
3. Fill `~/.env` (Supabase keys, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, your `HEALTH_BOT_TOKEN`, and `WHOOP_CLIENT_ID/SECRET`).
4. Create a Telegram bot via @BotFather; set `telegram_bot_token_env` in `agent.yaml`.
5. Copy `CLAUDE.md` and fill in your own profile.
6. Connect WHOOP (one-time OAuth). Schedule the sync + the morning check-in.

Full detail: [`docs/BUILD_GUIDE.md`](./docs/BUILD_GUIDE.md).

---

## Validation checklist

Work top to bottom. Check each box once you have personally verified it on your own setup.

### Supabase + schema
- [ ] Private Supabase project provisioned in an appropriate region
- [ ] `pgvector` extension enabled (the `0001_init.sql` migration does this)
- [ ] All migrations pushed; all 14 tables exist
- [ ] RLS enabled on every table, no policies (anon key reads nothing)
- [ ] Private `health-assets` storage bucket created
- [ ] Example seed replaced with your own goals + context (`0002_seed_example.sql`)
- [ ] `scripts/db.py select goals` returns your rows

### Agent + Telegram
- [ ] Bot created via @BotFather, token in `~/.env`
- [ ] Agent boots and responds to a plain message in Telegram
- [ ] Slash commands appear in the "/" menu (`/checkin`, `/today`, `/sofar`, `/newday`, `/supplements`, `/advice`)
- [ ] `CLAUDE.md` filled in with your real profile (goal, bloods, genetics, constraints)
- [ ] `state.py` snapshot prints your weight, intake, and goals

### Logging
- [ ] A food photo is read and written to `food_log` with macros + flags
- [ ] A weight message writes a `weigh_ins` row
- [ ] A workout / caffeine / supplement message writes its row
- [ ] A BP reading writes a `vitals` row
- [ ] A lab scan extracts markers into `lab_results`
- [ ] Semantic recall (`mem.py recall`) returns relevant past messages

### WHOOP
- [ ] WHOOP developer app created; scopes enabled
- [ ] Redirect URI registered exactly and saved
- [ ] One-time OAuth completed; `WHOOP_REFRESH_TOKEN` written to `~/.env`
- [ ] `whoop-sync.py` runs and writes `recovery_pct`, `hrv_ms`, `resting_hr`, `sleep_hours`
- [ ] Re-running the sync is idempotent (no duplicate rows for a day)
- [ ] The cron is scheduled and fires in the morning
- [ ] Token rotation verified (a second run still succeeds, no `invalid_grant`)
- [ ] Browser user-agent confirmed (no Cloudflare 1010 error)

### Coaching behavior
- [ ] Morning check-in fires and opens with last night's recovery
- [ ] The check-in ties yesterday's choices to the recovery number
- [ ] Today's plan is visibly gated on recovery (green = push, red = ease off)
- [ ] On-demand "should I eat this given how I slept" pulls recovery and leads with it
- [ ] `coach_summary` records the recovery + its cause (history accrues)
- [ ] The 7-day sleep/recovery trend shows in the snapshot

### Optional
- [ ] Live trends dashboard reachable and showing the Sleep & Recovery card
- [ ] Workout demo clip generation works (`exercise_clip.py`)
- [ ] Influencer advice RAG returns cited tips reconciled to your data

---

## The panel this system is built around

The coach is most useful grounded in a comprehensive baseline. This is the full set of **blood markers** and **DNA SNPs** the design tracks, the same panel the schema, the risk flags, and the goals are modeled on. One checkbox per test so you can tick them off as you order them. Names only, no values or genotypes; your own results live in your private `lab_results` rows and your filled-in `CLAUDE.md`.

### Blood markers

**Cardiometabolic**
- [ ] LDL-C
- [ ] ApoB
- [ ] ApoA-1
- [ ] Lp(a)
- [ ] hs-CRP
- [ ] HOMA-IR (fasting glucose + insulin)

**Hormones**
- [ ] Testosterone, total
- [ ] Testosterone, free
- [ ] SHBG
- [ ] Estradiol
- [ ] DHEA-S
- [ ] Pregnenolone
- [ ] Cortisol (AM)

**Thyroid**
- [ ] Reverse T3
- [ ] Thyroid panel (TSH, free T3, free T4)

**Liver**
- [ ] ALT
- [ ] AST

**Methylation**
- [ ] Homocysteine

**Vitamins + minerals**
- [ ] Vitamin D (25-OH)
- [ ] Magnesium
- [ ] Zinc
- [ ] Copper
- [ ] Selenium

**Foundation**
- [ ] Full lipid panel
- [ ] Complete blood count (CBC)
- [ ] Comprehensive metabolic panel

### DNA SNPs

**Lipids + cardiovascular**
- [ ] APOE
- [ ] LPA
- [ ] PCSK9
- [ ] CETP
- [ ] ACE
- [ ] AGT
- [ ] NOS3 (eNOS)

**Methylation / B-vitamins**
- [ ] MTHFR
- [ ] MTHFD1
- [ ] MTR
- [ ] MTRR
- [ ] CBS

**Detox (Phase II)**
- [ ] GSTM1
- [ ] GSTT1
- [ ] GSTP1

**Caffeine + neurotransmitters**
- [ ] COMT
- [ ] CYP1A2

**Vitamin D**
- [ ] VDR
- [ ] CYP2R1
- [ ] GC

**Metabolic / body weight**
- [ ] FTO
- [ ] PPARG
- [ ] TCF7L2

**Inflammation / antioxidant**
- [ ] TNF
- [ ] SOD2
- [ ] GPX1

**Other**
- [ ] HFE (iron handling)
- [ ] TAS2R38 (taste / bitter sensitivity)

Each marker or SNP maps to a risk flag, a supplement, or a target in the schema. That is how the coach gives mechanism-aware advice instead of generic tips.

---

## Why it is built this way

- **Specific beats generic.** A coach grounded in your real labs, genetics, and goals gives mechanism-aware advice; a generic bot gives platitudes. The whole design forces specificity.
- **Memory is the product.** Everything is written to structured rows, so the coach knows your whole history and can spot patterns over weeks, not just react to the last message.
- **Recovery is the spine of the day.** Opening each day with how you actually recovered, then tying it to what you did, teaches you your own levers.
- **The owner's data always wins.** Any external tip is reconciled against your own numbers.
- **Locked down by default.** Private project, service-role server-side, secrets in `~/.env`, out of git.
- **It is not a doctor.** It is a tracking and thinking tool that sends you to real clinicians for anything clinical.

---

## FAQ

**Is this medical advice?** No, see "Important: not medical advice" above. It is a software blueprint; verify everything with your own doctors.

**Do I need a WHOOP?** No. WHOOP is the worked example, but the same OAuth + daily-sync pattern fits Oura, Garmin, Fitbit, or manual logging. The coach and dashboard work with whatever lands in the `vitals` table.

**What does it cost to run?** The Supabase free tier handles one person. You pay for the LLM calls, embeddings (pennies), and Gemini vision for photos. WHOOP's API is free with a membership.

**Where does my data live?** Your own private Supabase project, locked down (service-role server-side, RLS on with no policies). Nothing leaves except the LLM calls you choose to make.

**Can it diagnose or change my medication?** No, and it is explicitly instructed not to. It flags clinical concerns toward a doctor and never touches medications.

**How accurate are the food-photo macros?** They are vision estimates, good for trend-tracking, not a substitute for weighing food. The coach says when it is guessing, and you should still run anything that matters past a professional.

---

## Important: not medical advice

This repository is a **software blueprint** for building a personal health-tracking assistant. It is **not** medical, nutritional, or fitness advice, and nothing in it is a recommendation for you.

- **It is one person's experience.** Every target, marker, supplement, and habit referenced here is an example of what worked for the author, who worked with qualified doctors at every step. Your physiology, labs, and risks are different.
- **Consult professionals, always.** Before acting on any value, panel, supplement, or plan, talk to your physician and the relevant specialists, and get your own labs interpreted by your own clinicians, every step of the way.
- **AI can hallucinate.** The coach is a large language model. It can be confidently wrong, miss context, or invent specifics. Treat every recommendation it produces as a prompt to verify with a doctor, not direction to follow. Run the AI's suggestions past real clinicians, exactly as the author did.
- **You own your health decisions.** The authors and contributors accept no liability for how you use this.

---

## Privacy

This is sensitive data. The Supabase project is private and locked down (service-role server-side, no anon policies). Keep your real `CLAUDE.md`, seed values, photos, and `~/.env` out of git. This repo is the scrubbed blueprint, not anyone's records.
