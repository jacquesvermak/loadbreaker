# Loadbreaker — a 60-second sprint from Stage 8 to Stage Zero

Tap to generate energy. The faster and more consistently you tap, the hotter
your combo meter runs and the bigger every tap pays out — up to ×12. Spend
what you earn on real Stage Zero products to add passive generation, then
race the 60-second clock down through South Africa's actual load-shedding
scale, stage by stage, until you hit **Stage 0**: the schedule where there's
nothing left to shed.

Plain HTML + CSS + JS, no build step, no external JS dependencies. One file.

## Run

```sh
python3 -m http.server 8000
```

Open <http://localhost:8000>. Or build and run the exact image the cluster
runs:

```sh
docker build -t loadbreaker:test .
docker run --rm -p 8080:8080 loadbreaker:test    # http://localhost:8080
```

## The loop

1. **Start — 60 seconds on the clock.** The grid starts at Stage 8: the worst
   of South Africa's real load-shedding scale, over twelve hours a day
   without grid power in 2023.
2. **Tap the grid.** Every tap adds energy equal to your base output times
   your current combo multiplier. Tap fast and consistently and the
   multiplier climbs toward ×12; stop, and it drains — speed is the whole
   game, not total taps.
3. **Spend while you tap.** Energy buys real Stage Zero products, each with
   its own effect on the combo system, not just raw output (see below).
4. **Watch the stage fall.** Crossing a stage threshold pops a short,
   factual note about what that stage actually means and ends the moment you
   either reach Stage 0 early or the clock hits zero.
5. **Result screen.** Stage reached, kWh generated, best combo, taps — then a
   shared leaderboard and a "get my real savings estimate" lead form.

### The stages

Thresholds are cumulative lifetime kWh generated in the round, not current
balance — spending doesn't undo stage progress.

| Stage | kWh to reach | |
| --- | --- | --- |
| 8 | 0 (start) | Over 12 hours a day without grid power was the reality in parts of SA in 2023. |
| 7 | 60 | |
| 6 | 200 | Outages start stacking on top of each other for most of the day. |
| 5 | 600 | Rolling blackouts don't ask what you were doing. |
| 4 | 1,500 | Solar generates exactly when the grid struggles most — hot, sunny afternoons. |
| 3 | 3,200 | Batteries turn daytime sun into night-time power. |
| 2 | 5,500 | Every panel is capacity the grid doesn't have to supply. |
| 1 | 8,000 | The tipping point — most homes generate more than they use on an average day. |
| 0 | 11,000 | Not a load-shedding stage at all — the schedule when there's nothing to shed. |

### The shop

| Product | Cost | Output | Effect |
| --- | --- | --- | --- |
| ☀️ Solar Panel | 4 (×1.13/unit) | +0.4 kWh/s | Baseline generation. |
| 🔋 Battery | 60 (×1.13/unit) | +2.5 kWh/s | +2.5 heat per tap — charges your combo faster. |
| 🔋 Battery Backup | 900 (×1.13/unit) | +25 kWh/s | Slows combo-heat decay ~10% per unit (floored). |
| 🏠 Fixed Backup System | 3,000 (×1.13/unit) | +90 kWh/s | +2 flat energy on every tap, on top of the combo multiplier. |

Product names and their real-world roles (portable vs. fixed, battery vs.
whole-home backup) come from Stage Zero's actual lineup — the specific costs,
outputs and combo effects are tuned for a 60-second arcade loop, not a
pricing calculator. Every product is a generic entry in one config array; the
buy/render/combo logic loops over it, so adding a fifth tier is a one-line
addition, not a refactor.

### Scoring & the leaderboard

A run is ranked by whether it reached Stage 0 at all, then:

- **Reached Stage 0:** ranked by time used to get there — faster wins.
- **Didn't reach it:** ranked by total kWh generated in the 60 seconds.

## Architecture

```
                    index.html
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   inline <style>   inline <script>   Google Fonts
   (tokens, no      (game state,      (Big Shoulders /
   framework)       STAGES/BUILDINGS   Public Sans /
                     config arrays,    JetBrains Mono)
                     vanilla DOM,
                     inline SVG scene)
```

No React, no bundler, no build step — the whole game is one HTML file with
inline CSS and JS. That's a deliberate choice for a sales-booth hand-off:
there's nothing to break between "clone" and "open in a browser," and the
Docker image is a single `COPY`.

### The shared leaderboard, two ways

This file was originally built as a Claude Artifact, where a runtime
capability (`window.claude.use("artifact")`) lets the page publish a new
version of itself with an updated leaderboard baked in as JSON, so every open
view sees the same board. **That capability doesn't exist outside the
Claude Artifacts host** — including this standalone deployment — so
`index.html` checks for it and falls back automatically:

- **On claude.ai**: submissions publish to a real, shared, cross-player board.
- **Here (and anywhere else)**: `window.claude` is `undefined`, so submissions
  and the "save my snapshot" download both fall back to `localStorage` and a
  real browser file download — fully functional, just scoped to one device
  instead of shared across players. See "Known limits" below for what a real
  shared leaderboard on this deployment would need.

## Deploying to QA

Push to `main` → GitHub Action builds the image → Harbor → the action rewrites
the image tag in `infrastructure/apps/sz-loadbreaker-game/values.yaml` →
ArgoCD syncs. Lands at **https://loadbreaker.qa.stagezero.co.za**.

Same pattern as `sz-battery-game` and `sz-zero-hour-game`: the chart keeps
its config in the default `values.yaml` rather than an `env/qa/` overlay, so
Helm — and therefore ArgoCD — reads it with no `helm.valueFiles` setting on
the Application. CI rewrites the same file ArgoCD renders, which removes any
way for the two to disagree about the image tag.

### One-time setup

| What | Where |
| --- | --- |
| `HARBOR_URL` variable | GitHub repo → Settings → Variables |
| `HARBOR_CA_CERT`, `HARBOR_USERNAME`, `HARBOR_PASSWORD` | GitHub repo → Settings → Secrets |
| ArgoCD `Application` → `path: infrastructure/apps/sz-loadbreaker-game` (no values file needed) | ArgoCD, `argocd` namespace |
| `qa-cert` TLS secret present in the `sz-loadbreaker-game` namespace | cluster |
| Harbor pull credentials for a fresh namespace — set `imagePullSecrets` in `values.yaml` if needed | cluster |

The ArgoCD `Application` itself isn't part of this repo's GitOps loop — it's
the one resource that has to be registered by hand (`kubectl apply`) against
the `argocd` namespace before anything here starts syncing, same as the two
sibling apps.

## Files

- `index.html` — the whole game: tokens, layout, energy scene, combo meter,
  shop, stage/milestone system, result screen, leaderboard, lead-capture
  flow, all inline
- `Dockerfile` · `nginx.conf` — static image, non-root nginx on 8080, `/healthz`
- `infrastructure/apps/sz-loadbreaker-game/` — Helm chart deployed by ArgoCD
- `.github/workflows/deploy.yml` — build → push to Harbor → rewrite manifest → ArgoCD sync

## Known limits of this build

- **No shared leaderboard on this deployment.** The Claude Artifacts version
  has one; this standalone build falls back to a per-device `localStorage`
  board (see above). A real cross-player board here needs a small backend —
  even a single key-value endpoint the page can `POST` a run to and `GET` the
  top 20 from would do it.
- **The lead-capture screen doesn't submit anywhere real.** "Send my details"
  shows a client-side confirmation and keeps the lead in `localStorage` only
  — wiring it to a real CRM/webhook is the natural next step before using
  this for actual lead generation.
- **"Save snapshot" is a local text file**, not an emailed or stored report.
- **Single language, no i18n.** English/Rand only.
- **No analytics.** Nothing is instrumented — worth adding before a real
  activation, to see where players drop off, which products get bought, and
  how often "Stage 0" is actually reached.
