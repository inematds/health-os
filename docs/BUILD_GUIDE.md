# Health OS — Build Guide

End to end, how to stand up your own personal health coach. Authored to be turnkey: follow it top to bottom and you will have a working coach with a live WHOOP connection. No personal data anywhere; fill in your own as you go.

> ⚠️ **Not medical advice.** This is a software blueprint, not health guidance. Everything here is simply what worked for one person who consulted their doctors at every step. It is not a prescription, diagnosis, or recommendation for you. Talk to qualified clinicians before changing anything about your labs, supplements, training, or diet. The AI coach can be wrong or hallucinate, so treat anything it suggests as a question to bring to your doctor, never an instruction to follow.

---

## 0. The design in one paragraph

A Telegram agent is the interface and brain. A private Supabase project is the durable memory. The agent reads a compact **snapshot** at the start of every turn, writes **structured rows** as you chat, and grounds advice in **your** labs, genetics, body composition, and goals. A daily job pulls your **wearable** into the database. A morning job runs a **recovery-led review**. Everything is mechanism-aware and specific, never generic.

---

## 1. Supabase (the memory)

1. Create a **private** project, region-appropriate for your data.
2. Generate a strong DB password; save it to `~/.env` as `SUPABASE_DB_PASSWORD`.
3. Link and push the schema:
   ```bash
   supabase link --project-ref <PROJECT_REF>
   supabase db push          # applies agent/supabase/migrations/
   ```
   `0001_init.sql` enables `pgvector`, creates all 14 tables, and locks them with RLS (enabled, no policies). The coach uses the **service-role** key server-side, which bypasses RLS; a leaked anon key reads nothing.
4. Create the private storage bucket for photos:
   ```bash
   # via the Storage API (more reliable than SQL on a fresh project)
   # bucket name: health-assets, public: false
   ```
5. Replace the placeholders in `0002_seed_example.sql` with your real goals + context and push.
6. Write the keys into `~/.env` (home dir, NOT a project `.env`):
   ```
   SUPABASE_URL=https://<PROJECT_REF>.supabase.co
   SUPABASE_ANON_KEY=<anon key>
   SUPABASE_SERVICE_ROLE_KEY=<service_role key>
   SUPABASE_DB_PASSWORD=<password>
   SUPABASE_PROJECT_REF=<PROJECT_REF>
   ```

### The tables
`messages` (every exchange, embedded for recall), `assets` (photos/scans), `weigh_ins`, `body_measurements`, `food_log`, `workouts`, `supplements_log`, `caffeine_log`, `vitals` (BP + wearable metrics), `lab_results`, `daily_checkins`, `goals`, `context` (where you are + how to adapt), `influencer_tips` (optional RAG). `db.py` is a zero-dependency REST helper over the service-role key; `mem.py` handles embedded semantic memory.

---

## 2. The agent

1. Create a Telegram bot via **@BotFather**, copy the token to `~/.env` as `HEALTH_BOT_TOKEN`.
2. Set `telegram_bot_token_env: HEALTH_BOT_TOKEN` in `agent.yaml` (copy from `agent.yaml.example`).
3. Copy `CLAUDE.md` and **fill in your profile**: the one goal, your bloods (with targets), the genetic variants that change your advice, body comp, and constraints. This is the single biggest lever on quality.
4. Boot the agent and send a message. It should read `state.py`, answer in context, and write a `messages` row.

The slash commands (`/checkin`, `/today`, `/sofar`, `/newday`, `/supplements`, `/advice`) are defined in `agent.yaml`; they register in the Telegram "/" menu on agent startup. After editing commands, **restart the agent** so it re-registers the menu.

---

## 3. WHOOP (or any OAuth wearable)

The pattern generalizes to Oura, Garmin, Fitbit, etc.: an OAuth app, a redirect/callback your service hosts, a token store, and a daily sync.

### 3.1 The app
- developer-dashboard.whoop.com -> create an app.
- Redirect URI: `https://<your-host>/whoop/callback` (must be **https**; plain `http://localhost` is rejected).
- Scopes: `read:recovery read:sleep read:cycles offline` (the `offline` scope is what grants a refresh token).
- Copy Client ID + Secret to `~/.env` (`WHOOP_CLIENT_ID`, `WHOOP_CLIENT_SECRET`).

### 3.2 The OAuth flow
- A token-gated route (`/whoop/connect`) on your service 302s to WHOOP's authorize URL with the scopes and a CSRF `state`.
- WHOOP redirects back to `/whoop/callback` with a code; your handler validates the `state`, exchanges the code at `https://api.prod.whoop.com/oauth/oauth2/token`, and writes `WHOOP_REFRESH_TOKEN` to `~/.env`.
- Authorize **once** in a browser logged into WHOOP.

### 3.3 The daily sync (`scripts/whoop-sync.py`)
- Refreshes the access token. **WHOOP rotates the refresh token on every refresh and invalidates the old one**, so the script persists the new token back to `~/.env` immediately. Skip this and the next run fails with `invalid_grant`.
- Pulls the latest SCORED `GET /v2/recovery` and the matching `GET /v2/activity/sleep/{id}` from `https://api.prod.whoop.com/developer`.
- Maps fields and writes one `vitals` row per metric per local day (idempotent, delete-then-insert):

  | WHOOP field | vitals metric |
  |---|---|
  | `score.recovery_score` | `recovery_pct` |
  | `score.hrv_rmssd_milli` | `hrv_ms` |
  | `score.resting_heart_rate` | `resting_hr` |
  | `(light + slow_wave + rem) / 3.6e6` | `sleep_hours` |

- **Cloudflare gotcha:** WHOOP's API is behind Cloudflare, which bans the default `Python-urllib` user-agent (HTTP 1010). `whoop_common.py` sets a browser user-agent. Keep it.

### 3.4 Schedule it
Run `whoop-sync.py` a few times each morning (e.g. 7/10/13 local) so it catches your recovery whenever WHOOP scores the night. The writes are idempotent, so repeats are harmless.

---

## 4. The morning review (the scheduled job)

A job fires each morning and runs the check-in. The arc, in order:
1. Open with last night's recovery (already in the snapshot, never asked for).
2. Close the loop: tie yesterday's food/caffeine/training to why recovery landed where it did.
3. Supplements: assume taken, flag only exceptions.
4. Trend: weight, protein, caffeine, BP, and the 7-day sleep/recovery pattern.
5. Today's plan, gated on recovery.
6. Ask only what the wearable cannot tell you (weight, BP, subjective energy).

Write `daily_checkins` with the recovery + its one-line cause in `coach_summary` so the longitudinal picture builds (semantic recall searches these).

---

## 5. Photo workflows

- **Food photo** -> estimate items, calories, protein; set risk flags from the user's profile; immediate feedback; store the photo + a `food_log` row.
- **Lab scan** -> extract markers into `lab_results`; compare to baseline; flag changes; celebrate target hits.
- **Body photo** -> optionally prescribe a workout; track body composition over time.

Vision uses your `GOOGLE_API_KEY` (Gemini) or equivalent.

---

## 6. Optional: live dashboard + workout clips

- A self-contained web dashboard can read the same Supabase data and chart weight, nutrition, BP, and the Sleep & Recovery card. Expose it over a tunnel and gate it with a token.
- `scripts/exercise_clip.py` generates looping exercise demo clips (define the move, render poses, image-to-video, loop, store).

---

## 7. Principles that make it good

- **Specific beats generic.** The coach is only as good as the profile in `CLAUDE.md`.
- **Write structured rows as you go.** The point of the system is that it knows everything over time.
- **Recovery is the spine of the day.** Open with it, explain it, plan around it.
- **The owner's own data wins.** Reconcile every tip against their labs and genetics.
- **Lock the data down.** Private project, service-role server-side, no anon policies, secrets in `~/.env` out of git.
