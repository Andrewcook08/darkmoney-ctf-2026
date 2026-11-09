# DarkMoney CTF — Claude Workflow Guide

## Overview
This is an authorized CTF (CMD+CTRL Cyber Range, Schwab 2026 Advanced) on the DarkMoney vulnerable banking app. All vulnerability testing is in-scope.

**Target:** https://50-18-145-16-darkmoneyprod.vulnerablesites.net  
**Tracker:** https://crwd-ctrl.net/d9e8f33f-db5e-405c-8236-499d2f0df794  
**Ranges dashboard:** https://cmdnctrl.net/ranges  
**Account:** Alice Morgan, user ID 1, clientId DM-CLIENT-A9F131  
**Auth:** HttpOnly session cookie — user must log in manually; Claude uses `{credentials:'include'}` in fetch calls

---

## Standard Workflow

### For Each Challenge:
1. **Read the challenge description** on crwd-ctrl.net (click challenge name in sidebar)
2. **Identify the vulnerability type** from the hint (IDOR, XSS, SQLi, SSRF, etc.)
3. **Probe the DarkMoney API** via JavaScript fetch in the browser console (tab 2 = DarkMoney)
4. **Document findings** in STATUS.md immediately
5. **Submit to crwd-ctrl** — most challenges auto-detect when you hit the right endpoint; some require SUBMIT button + file upload
6. **Check for newly unlocked challenges** after each solve
7. **Update STATUS.md** with the solution details — including points, method, and exact payload/request used
8. **Commit and push** to GitHub after every solve so the team stays in sync:
   ```bash
   git add STATUS.md && git commit -m "solve: <challenge name> (+Xpts)" && git push
   ```

### How to Open Challenge Form on crwd-ctrl:
The SUBMIT button triggers a React state change. Use this to open the form programmatically:
```javascript
// Find and open the challenge form
const submitBtn = [...document.querySelectorAll('button')].find(b => b.textContent.trim() === 'Submit');
const fiberKey = Object.keys(submitBtn).find(k => k.startsWith('__reactFiber'));
let fiber = submitBtn[fiberKey];
let count = 0;
while (fiber && count < 30) {
  if (fiber.stateNode?.state?.isChallengeFormOpen !== undefined) {
    fiber.stateNode.setState({isChallengeFormOpen: true});
    break;
  }
  fiber = fiber.return;
  count++;
}
```

### How to Upload a File to crwd-ctrl:
1. Open the challenge form (above)
2. Use `read_page` with `filter: "all"` on the crwd-ctrl tab to find the `input[type=file]` ref
3. Use `mcp__claude-in-chrome__file_upload` with the file ref and path
4. Click the Send button

### How to Close Dialogs on crwd-ctrl:
```javascript
const closeBtn = [...document.querySelectorAll('button')].find(b => b.textContent.trim() === 'Close');
closeBtn?.click();
```

---

## Key Endpoints & Credentials

### Leaked at /secrets:
- `JWT_SIGNING_KEY`: `super-secret-signing-key-only-for-darkmoney-admins`
- `DB_PASSWORD`: `CorrectHorseBatteryStaple_final_final`
- `AWS_ACCESS_KEY_ID`: `AKIAFAKE1234567890`
- `ENCRYPTION_SALT`: `NaCl_just_a_pinch`

### Known Users:
- User 1: Alice Morgan, username `amorgan` (our account)
- User 2: Priya Kapoor, username `pkapoor`
- Users 3–8: exist (varying endpoints return 200 vs 400 for different IDs)

### API Endpoints Discovered:
| Endpoint | Method | Notes |
|----------|--------|-------|
| `/robots.txt` | GET | Leaks hidden paths |
| `/secrets` | GET | Leaks all app credentials |
| `/api/users/self` | GET | Own user object |
| `/api/users/{id}/accounts` | GET | IDOR — works for users 1–8 |
| `/api/users/{id}/loans` | GET | IDOR — works for users 1–8 |
| `/api/users/{id}/transactions/download` | GET | IDOR PDF — works for users 1–5 |
| `/api/transactions/statement/print` | GET | HTML statement — XSS vector |
| `/api/transactions?page=0&size=10` | GET | Paginated tx list |
| `/api/bill-pay/payments?page=0&size=50` | GET | Own bill payments |
| `/api/bill-pay/payments/{id}` | GET | 403 = other user's, 404 = nonexistent |
| `/api/oracle/positions` | GET | Own oracle bets |
| `/api/oracle/positions/{id}` | GET | IDOR — sequential int IDs |
| `/api/auth/login` | POST | `{username, password}` → `{success, blocked}` |
| `/api/linked-accounts` | GET | Own linked accounts |
| `/api/partner-banks` | GET | Lists 4 partner banks |

### Partner Bank Codes:
- `DM-ATL-0042` — Bank of Atlantis
- `DM-GRO-0117` — Groß Notional Bank
- `DM-TGK-0298` — Tallgrass Konza Bank
- `DM-FUE-0531` — Font & Underwood Exchange

---

## Per-Challenge Requirements

Each challenge must have ALL of the following recorded in STATUS.md before marking complete:
- **Points awarded** (from crwd-ctrl score delta)
- **Vulnerability type** (IDOR, XSS, SQLi, Business Logic, etc.)
- **Exact HTTP request** — method, endpoint, headers/body used
- **Proof of success** — response body or crwd-ctrl confirmation
- **Submission method** — auto-detect / SUBMIT button + file upload / flag text

### Completed Challenge Solutions

| Challenge | Type | Key Request | Submission |
|-----------|------|-------------|------------|
| Whispers in the Source | Info Disclosure | View page source of DarkMoney | Auto |
| The Open Ledger | IDOR | `GET /api/users/{2-8}/accounts` | Auto |
| Not Your Loan | IDOR | `GET /api/users/{2-8}/loans` | Auto |
| Things You Were Asked Not to See | Info Disclosure | `GET /robots.txt` → `GET /secrets` | Auto |
| Paper Trail | IDOR + File Upload | `GET /api/users/2/transactions/download` → save PDF → upload to crwd-ctrl | File upload |
| Locked, But Not Really | Broken Auth | `POST /api/auth/login` wrong pw 3× for `pkapoor` → `blocked:true` | Auto |
| Knock and Listen | Enumeration | `GET /api/bill-pay/payments/53` → 403 vs 404 differential | Auto |
| The Forgotten Rite | IDOR | `GET /api/oracle/positions/1` (IDs 1–6 = other users) | Auto |
| A Tribute in Reverse | Business Logic | `POST /api/oracle/markets/1005/orders` `{"side":"YES","shares":-100}` | Auto |
| Name Your Price | Business Logic | `POST /api/oracle/markets/1005/orders` `{"side":"YES","shares":100,"entryPriceCents":1}` | Auto |
| Borrowed Confidence | Business Logic | `POST /api/loans` `{"loanType":"PERSONAL","loanAmount":25000,"creditScore":850}` | Auto |

---

## STATUS.md
All challenge solutions and status are tracked in `STATUS.md`. Always update it after each solve.

---

## Browser Setup
- Tab 1 (crwd-ctrl): `https://crwd-ctrl.net/d9e8f33f-db5e-405c-8236-499d2f0df794`
- Tab 2 (DarkMoney): `https://50-18-145-16-darkmoneyprod.vulnerablesites.net`
- Run all API probes via `mcp__claude-in-chrome__javascript_tool` on the DarkMoney tab

## Local LLM (available for analysis):
```bash
curl http://localhost:11434/api/chat \
  -d '{"model":"qwen38-27b-unc-q4:latest","messages":[{"role":"user","content":"PROMPT"}],"stream":false,"think":true,"options":{"temperature":0.7,"num_ctx":32768},"keep_alive":-1}'
```
