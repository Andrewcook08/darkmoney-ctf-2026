# Cyber Range 2026 — DarkMoney Tracking

## Event
- Platform: CMD+CTRL Cyber Ranges (cmdnctrl.net)
- Event: Schwab 2026 Advanced
- Range: DarkMoney (Financial) — "Moving money is easy. Securing it isn't."
- Status: waiting on admin to start event (after 9 AM Nov 9, 2026)

## Ground Rules (from FAQ)
- Dummy users exist but we don't know their passwords — must find a vuln (e.g. SQLi) to log in, or register our own account.
- cmdnctrl.net login creds do NOT work on the range itself.
- If range misbehaves: try "database reset" first (score/completed challenges preserved), then full "reset" (5-10 min) if needed.
- OFF LIMITS: Credits page sites, external blog/social accounts tied to the game (look, don't attack/login), the scoreboard itself.
- Decoded messages often point to a URL path to visit for points, not a flag field.
- Scoring requires real impact — errors/weirdness alone don't score.
- SQLi scores: login bypass (1x), per-injection-point DB modification, per-injection-point data exfil (where results are shown back).
- XSS scores: must pop a JS alert with ZERO user interaction to auto-score.

## Task Log
| # | Task (from hacker general) | Status | Notes/Points |
|---|---|---|---|
|   |   |   |   |

## Findings Log
| Vuln type | Location/endpoint | Payload used | Result | Scored? |
|---|---|---|---|---|
|   |   |   |   |   |