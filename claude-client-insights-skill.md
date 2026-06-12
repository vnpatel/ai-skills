---
name: "client-insights"
description: "Client intelligence skill for Cumulus FSC. Two modes: (1) INDIVIDUAL CLIENT PROFILE — use when the user asks about a specific named person (\"tell me about Rachel Adams\", \"show me John's portfolio\", \"what's [name]'s profile/assets/risk/goals\") — looks up full 360 profile: demographics, income, AUM across all account types, financial goals, attrition risk, email/web engagement, cases, and CRM opportunities. (2) SEGMENT & POPULATION ANALYTICS — use when the user asks aggregate questions about customer populations (\"how many affluent clients in California\", \"show me clients over 65\", \"what are our top segments\", \"clients with assets over $1M\") — queries Data Cloud DMOs directly without requiring a segment. Invoke this skill proactively for ANY question about a client or group of clients."
author: "Viral Patel"
version: "2.0.1"
date: "2026-06-10"
comment: "Unified client intelligence skill. v2 adds individual client profile mode (CRM + Data Cloud 360 lookup) on top of v1 population/segment analytics."
---

# Client Insights Skill

## Execution behavior — CRITICAL
**Work silently.** Do not narrate, explain, or describe what you are doing between queries. Do not say things like "Now I'll fetch...", "Let me try...", "I found X, so next I'll...". Execute all queries without commentary and output ONLY the final formatted card. The user sees tool call indicators automatically — do not add text narration on top of them.

**When queries fail or return 0 results, work around it silently.** Never surface technical failure reasons in output. Specifically:
- If a cross-DMO semi-join returns 0 results, skip that filter and report the dimensions separately (e.g. population count from Individual DMO + attrition stats from Attrition DMO)
- If a DMO is inaccessible or a field is missing, omit that section silently
- Never write phrases like "semi-join doesn't resolve", "Data Cloud configuration", "org limitation", "this relationship doesn't work", or any explanation of why data is unavailable
- The `⚠ Data notes` footer must only list field coverage gaps (e.g. "income data ~25% populated") — never mention query failures, infrastructure issues, or workarounds applied

---

## MODE 1 — Individual Client Profile

### When to invoke
Trigger this mode when the user asks about a **specific named person**:
- "Tell me about [Name]"
- "What's [Name]'s portfolio / profile / risk / goals"
- "Show me [Name]'s accounts / assets / cases"
- Any question where a full name (or partial name) is the subject

---

### Step 1 — CRM Name Search

Use SOSL to find the contact:
```sql
FIND {"[Name]"} IN NAME FIELDS
RETURNING Contact(
  Id, FirstName, MiddleName, LastName, Email, Phone, MobilePhone,
  Title, Department, Birthdate, Account.Name, Account.Type,
  MailingStreet, MailingCity, MailingState, MailingPostalCode,
  OwnerId
)
```

**If 0 results:** Try a broader search (first name only, or last name only). If still nothing, report not found.

**If multiple results:** Show a disambiguation table and STOP — wait for the user to confirm which contact before proceeding:

| # | Name | Title | Account | Email |
|---|---|---|---|---|
| 1 | ... | ... | ... | ... |

**If 1 result:** Proceed directly to Step 2.

---

### Step 2 — Resolve Contact EID (Bridge Query)

**CRITICAL:** Financial DMOs and other Data Cloud DMOs join on `Contact_EID__c` (a custom EID like `CI246-151010-GR`), NOT the CRM Contact Id. You must resolve this EID before running Step 3 queries.

The bridge: `ssot__Individual__dlm.Id` = CRM Contact Id. Query Individual by `Id` to fetch `Contact_EID__c` and the full core profile in one shot:

```sql
SELECT Contact_EID__c, ssot__FirstName__c, ssot__LastName__c, Age__c,
       ssot__BirthDate__c, Address_City__c, Address_State_Prov__c,
       Address_Postal_Code__c, ssot__YearlyIncome__c, ssot__GenderId__c,
       Life_Event__c, Product_Interest__c, Product_Category_Interest__c
FROM ssot__Individual__dlm
WHERE Id = '[CRM Contact Id]'
LIMIT 1
```

**If 0 results:** Show CRM data only and note "No Data Cloud profile found for this contact."

**Once you have `Contact_EID__c`** (e.g. `CI246-151010-GR`): use this value for ALL Step 3 queries. Do NOT use the CRM Contact Id in any financial or engagement DMO query.

---

### Step 3A — Lead Card Queries (ALWAYS run on first invocation)

Run these 6 queries only. Do NOT run Step 3B queries on first invocation.

#### 3A-a — Attrition + pre-computed summary
```sql
SELECT YearlyIncome__c, Attrited_prediction__c,
       Goa_Has_Goals__c, Goa_Number_of_Goals__c,
       Tra_Number_of_Accounts__c, Ten_OPEN_DATE_formula__c,
       Predictor_1_Name__c, Predictor_1_Value__c,
       Predictor_2_Name__c, Predictor_2_Value__c,
       Predictor_3_Name__c, Predictor_3_Value__c
FROM Attrition_Prediction_With_Predictors__dlm
WHERE Contact_EID__c = '[Contact_EID__c]'
LIMIT 1
```

#### 3A-b — Investment total (aggregate only)
```sql
SELECT SUM(balance__c)
FROM Account_Investment__dlm
WHERE customer_id__c = '[Contact_EID__c]'
```

#### 3A-c — Deposit total (aggregate only)
```sql
SELECT SUM(balance__c)
FROM Account_Deposit__dlm
WHERE customer_id__c = '[Contact_EID__c]'
```

#### 3A-d — Loan total (aggregate only)
```sql
SELECT SUM(balance__c)
FROM Account_Loan__dlm
WHERE customer_id__c = '[Contact_EID__c]'
```

#### 3A-e — Open cases (most recent 3)
```sql
SELECT ssot__Subject__c, ssot__CaseStatusId__c, ssot__Origin__c, ssot__CreatedDate__c
FROM ssot__Case__dlm
WHERE Contact_EID__c = '[Contact_EID__c]'
AND ssot__IsClosed__c = 'false'
ORDER BY ssot__CreatedDate__c DESC NULLS LAST
LIMIT 3
```

#### 3A-f — Open opportunities (CRM — uses CRM Contact Id from Step 1)
```sql
SELECT Id, Name, StageName, Amount, CloseDate
FROM Opportunity
WHERE Id IN (
  SELECT OpportunityId FROM OpportunityContactRole WHERE ContactId = '[CRM Contact Id from Step 1]'
)
AND IsClosed = false
ORDER BY CloseDate ASC NULLS LAST
LIMIT 5
```

---

### Step 3B — Detail Queries (ONLY run when user explicitly asks for more)

Trigger phrases: "show me more", "full profile", "show accounts", "show goals", "show engagement", "show cases", "show opportunities", or any specific follow-up question about account details.

#### 3B-a — Investment account rows
```sql
SELECT account_id__c, name__c, balance__c, status__c, type__c,
       risk_tolerance__c, tax_status__c, goals__c, open_date__c
FROM Account_Investment__dlm
WHERE customer_id__c = '[Contact_EID__c]'
ORDER BY balance__c DESC NULLS LAST
```

#### 3B-b — Deposit account rows
```sql
SELECT account_id__c, name__c, balance__c, status__c, type__c,
       interest_rate__c, open_date__c
FROM Account_Deposit__dlm
WHERE customer_id__c = '[Contact_EID__c]'
```

#### 3B-c — Loan rows
```sql
SELECT account_id__c, name__c, balance__c, status__c, type__c,
       interest_rate__c, minimum_payment__c, payment__c, term__c, due_date__c
FROM Account_Loan__dlm
WHERE customer_id__c = '[Contact_EID__c]'
```

#### 3B-d — Credit cards
```sql
SELECT account_id__c, name__c, balance__c, status__c, type__c,
       available_credit__c, monthly_purchase_volume__c,
       points_balance__c, interest_rate__c, payment__c, due_date__c
FROM Account_Credit_Card__dlm
WHERE customer_id__c = '[Contact_EID__c]'
```

#### 3B-e — Insurance policies
```sql
SELECT account_id__c, name__c, status__c, type__c,
       coverage_amount__c, premium_amount__c, renewal_date__c, open_date__c
FROM Account_Policy__dlm
WHERE customer_id__c = '[Contact_EID__c]'
```

#### 3B-f — Financial goals
```sql
SELECT goal_id__c, name__c, type__c, status__c,
       targetvalue__c, currentvalue__c, GOAL_DELTA_VALUE__c, date__c
FROM Financial_Goal__dlm
WHERE customer_id__c = '[Contact_EID__c]'
ORDER BY date__c DESC NULLS LAST
```

#### 3B-g — Email engagement (last 5)
```sql
SELECT engagement_date__c, engagement_type__c, email_name__c,
       email_subject_line__c, channel__c
FROM Email_Engagement__dlm
WHERE customer_id__c = '[Contact_EID__c]'
ORDER BY engagement_date__c DESC NULLS LAST
LIMIT 5
```

#### 3B-h — Web engagement (last 5 sessions)
```sql
SELECT session_date__c, category_viewed__c, device_category__c,
       device_os_name__c, browser__c, session_duration__c, page_views__c
FROM Web_Engagement__dlm
WHERE customer_id__c = '[Contact_EID__c]'
ORDER BY session_date__c DESC NULLS LAST
LIMIT 5
```

#### 3B-i — All cases
```sql
SELECT ssot__Subject__c, ssot__CaseStatusId__c, ssot__CaseTypeId__c,
       ssot__IsClosed__c, ssot__CreatedDate__c, ssot__ClosedDateTime__c,
       ssot__Origin__c
FROM ssot__Case__dlm
WHERE Contact_EID__c = '[Contact_EID__c]'
ORDER BY ssot__CreatedDate__c DESC NULLS LAST
LIMIT 10
```

#### 3B-j — All opportunities (CRM)
```sql
SELECT Id, Name, StageName, Amount, CloseDate, Type, LeadSource,
       Probability, IsClosed, IsWon
FROM Opportunity
WHERE Id IN (
  SELECT OpportunityId FROM OpportunityContactRole WHERE ContactId = '[CRM Contact Id from Step 1]'
)
ORDER BY CloseDate DESC NULLS LAST
LIMIT 10
```

---

### Step 4 — Output

#### First invocation — Lead Card only

Compute AUM = sum of investment + deposit balances. Liabilities = sum of loan balances. **Omit any line silently where data is null.**

```
## [Full Name] — Client Snapshot
Age [X] · Income $[X] · [City, ST]
AUM $[X] ([Wealth Tier]) · Liabilities $[X]
Attrition [X]% — [Low / Medium / High Risk] · Top driver: [Predictor_1_Name]
Product interest: [Product_Interest__c] · [Product_Category_Interest__c]

[Only if open opps exist:] Open opportunities ([N]): [Opp Name] — [Stage] (closes [CloseDate]) …
[Only if open cases exist:] Open cases ([N]): [Subject] — [Status] …

**Key signals:** [2–3 sentence synthesis of the most actionable signals from the data above —
e.g. tenure, attrition risk, open issues, product interest alignment with open opps.]
```
*Ask for accounts, goals, engagement, cases, or opportunities.*

**Key signals rules:**
- Always include — synthesize what matters most for a client conversation
- Draw from: tenure (`Ten_OPEN_DATE_formula__c`), AUM tier, attrition risk + top driver, open cases, open opps, product interest
- Write in plain business language — no field names, no technical jargon
- 2–3 sentences max; lead with the most actionable signal

**Other line rules:**
- Line 1 (age/income/location): income from Attrition DMO (`YearlyIncome__c`) as primary, Individual DMO as fallback; omit if both null
- Line 2 (AUM): wealth tier — Starter ≤$100K, Growth $100K–$500K, Affluent >$500K; omit liabilities if no loans
- Line 3 (attrition): score 0–100, show as percentage; omit if no attrition record
- Line 4 (product interest): omit if null
- Opp/case lines: show count + up to 3 items inline; omit entirely if none open

#### On follow-up (user asks for more detail)

Run the relevant Step 3B queries and render the appropriate detail tables (investment accounts, deposits, loans, credit cards, insurance, goals, email engagement, web engagement, full cases, full opportunities). Only run and render what was asked for — not all sections at once unless "full profile" was requested.

**Edge cases:**
- No Data Cloud profile → show name/CRM title/account + note "No Data Cloud profile found"
- No financial accounts → show identity lines + note "No financial accounts found in Data Cloud"

---

## MODE 2 — Segment & Population Analytics

### When to invoke
Trigger this mode when the user asks **aggregate questions** about customer populations:
- "How many affluent clients in California"
- "Show me clients who are over 65 with high attrition risk"
- "What % of customers have investment accounts over $1M"
- "What are our top segments"
- Any question of the form "how many [type of customer]...", "show me clients who...", "what % of customers..."

### Routing logic
- **Segment mentioned by name** → Query `Analytics_Market_Segment__c` in CRM first, then enrich with DMO queries
- **No segment mentioned** → Go directly to DMO queries anchored on `ssot__Individual__dlm`

---

## Confirmed DMO Field Reference (DO NOT inspect schemas at runtime — use these directly)

### `ssot__Individual__dlm` — Core person profile
- `Contact_EID__c` — primary join key to all financial DMOs
- `ssot__FirstName__c`, `ssot__LastName__c`, `ssot__PersonName__c`
- `Age__c` — numeric age
- `ssot__BirthDate__c` — birth date
- `Address_State_Prov__c` — state/province
- `Address_City__c` — city
- `Address_Postal_Code__c` — zip code
- `ssot__YearlyIncome__c` — yearly income (only ~25% populated; use Attrition DMO income as fallback)
- `ssot__GenderId__c` — gender
- `Life_Event__c` — life events
- `Product_Interest__c` — product interest
- `Product_Category_Interest__c` — product category interest

### `Attrition_Prediction_With_Predictors__dlm` — Pre-computed customer summary (use this for efficiency)
Join: `Contact_EID__c` = `ssot__Individual__dlm.Contact_EID__c`
- `Contact_EID__c` — join key
- `YearlyIncome__c` — yearly income (more reliably populated than Individual)
- `Attrited_prediction__c` — attrition score (0–100 scale; > 60 = high risk, avg ~18, min 0.01, max 99.58)
- `Goa_Has_Goals__c` — has financial goals (string 'true'/'false')
- `Goa_Number_of_Goals__c` — count of goals
- `Tra_Number_of_Accounts__c` — total number of accounts across all types
- `Ten_OPEN_DATE_formula__c` — customer tenure (days)
- `Predictor_1_Name__c`, `Predictor_1_Value__c` — top attrition predictor
- `Predictor_2_Name__c`, `Predictor_2_Value__c`
- `Predictor_3_Name__c`, `Predictor_3_Value__c`

### `Account_Investment__dlm` — Investment accounts
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
- `customer_id__c`, `account_id__c`
- `name__c` — account name
- `balance__c` — current balance (double)
- `status__c` — account status
- `type__c` — investment type
- `risk_tolerance__c` — risk tolerance
- `tax_status__c` — tax status (taxable, IRA, Roth, etc.)
- `goals__c` — linked goals
- `open_date__c` — account open date

### `Account_Deposit__dlm` — Checking/savings/deposit accounts
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
- `customer_id__c`, `account_id__c`
- `name__c`, `balance__c`, `status__c`, `type__c`
- `interest_rate__c`
- `open_date__c`

### `Account_Credit_Card__dlm` — Credit card accounts
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
- `customer_id__c`, `account_id__c`
- `name__c`, `balance__c`, `status__c`, `type__c`
- `available_credit__c`
- `monthly_purchase_volume__c`
- `points_balance__c`
- `interest_rate__c`
- `payment__c`
- `due_date__c`

### `Account_Loan__dlm` — Loans and mortgages
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
- `customer_id__c`, `account_id__c`
- `name__c`, `balance__c`, `status__c`, `type__c`
- `interest_rate__c`
- `minimum_payment__c`
- `payment__c`
- `term__c`
- `due_date__c`

### `Account_Policy__dlm` — Insurance policies
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
- `customer_id__c`, `account_id__c`
- `name__c`, `status__c`, `type__c`
- `coverage_amount__c`
- `premium_amount__c`
- `renewal_date__c`
- `open_date__c`

### `Financial_Transaction__dlm` — All financial transactions
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
Also join: `account_id__c` = any Account DMO `account_id__c`
- `customer_id__c`, `account_id__c`, `transaction_id__c`
- `amount__c` — transaction amount
- `date__c` — transaction date
- `type__c` — transaction type (payment, deposit, debit, etc.)
- `category__c` — category
- `description__c`
- `balance__c` — running balance after transaction

### `Financial_Goal__dlm` — Financial goals
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
- `customer_id__c`, `goal_id__c`
- `name__c` — goal name
- `type__c` — goal type (Buying a Home, Saving for Retirement, etc.)
- `status__c` — goal status
- `targetvalue__c` — target amount
- `currentvalue__c` — current progress
- `GOAL_DELTA_VALUE__c` — gap to goal
- `Goal_Icon__c`
- `date__c`

### `Email_Engagement__dlm` — Email engagement
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
Note: Dates may be years old — always ORDER BY engagement_date__c DESC, ignore staleness
- `customer_id__c`
- `engagement_date__c` — most recent engagement date
- `engagement_type__c` — open, click, etc.
- `email_name__c` — email/campaign name
- `email_subject_line__c` — subject line
- `channel__c`

### `Web_Engagement__dlm` — Web engagement
Join: `customer_id__c` = `ssot__Individual__dlm.Contact_EID__c`
Note: Dates may be years old — always ORDER BY session_date__c DESC, ignore staleness
- `customer_id__c`
- `session_date__c` — session date
- `category_viewed__c` — content category viewed
- `device_category__c` — device type
- `device_os_name__c` — OS name
- `browser__c`
- `session_duration__c` — duration in seconds
- `page_views__c` — pages viewed

### `ssot__Case__dlm` — Service cases
Join: `Contact_EID__c` = `ssot__Individual__dlm.Contact_EID__c`
- `Contact_EID__c`
- `ssot__CaseStatusId__c` — status
- `ssot__CaseTypeId__c` — type
- `ssot__Subject__c` — subject
- `ssot__IsClosed__c` — 'true'/'false'
- `ssot__CreatedDate__c`
- `ssot__ClosedDateTime__c`
- `ssot__Origin__c` — channel of origin

---

## Hardcoded Wealth Tiers (pre-computed from actual data — do NOT recompute at runtime)

### Investment Balance Tiers (primary wealth proxy)
Based on `Account_Investment__dlm.balance__c` across 1,402,941 accounts:
- **Starter**: balance__c ≤ $100,000 → 330,207 accounts (23.5%)
- **Growth**: balance__c > $100,000 AND ≤ $500,000 → 525,974 accounts (37.5%)
- **Affluent**: balance__c > $500,000 → 546,760 accounts (39.0%)
- Data stats: Min $2,501 | Avg $492,190 | Max $2,499,992
- Total Individuals: 4,051,584

### Income Tiers (from `Attrition_Prediction_With_Predictors__dlm.YearlyIncome__c`)
Based on ~1M records with populated income:
- **Mass Market**: YearlyIncome__c < $100,000
- **Mass Affluent**: YearlyIncome__c ≥ $100,000 AND < $300,000
- **Affluent**: YearlyIncome__c ≥ $300,000
- Data stats: Min $55,000 | Avg $176,114 | Max $724,998

### Attrition Risk Tiers
Based on `Attrition_Prediction_With_Predictors__dlm.Attrited_prediction__c` (0–100 scale):
- **Low Risk**: < 30
- **Medium Risk**: 30–60
- **High Risk**: > 60
- Data stats: Min 0.01 | Avg 18.48 | Max 99.58 | ~117,703 high-risk clients total

---

## DMO Selection Logic for Population Queries (pick minimum needed for the question)

| Question type | Required DMOs |
|---|---|
| Geographic / age / name only | `ssot__Individual__dlm` only |
| Income / wealth tier | `ssot__Individual__dlm` + `Attrition_Prediction_With_Predictors__dlm` |
| Investment assets / balances | + `Account_Investment__dlm` |
| Deposit / savings balances | + `Account_Deposit__dlm` |
| Total assets (all accounts) | + Investment + Deposit + Loan (as liability) |
| Credit card behavior | + `Account_Credit_Card__dlm` |
| Loan / mortgage holders | + `Account_Loan__dlm` |
| Insurance / policy holders | + `Account_Policy__dlm` |
| Financial goals | + `Financial_Goal__dlm` |
| Transaction behavior | + `Financial_Transaction__dlm` |
| Email engagement | + `Email_Engagement__dlm` |
| Web engagement | + `Web_Engagement__dlm` |
| Service / cases | + `ssot__Case__dlm` |
| Attrition risk | + `Attrition_Prediction_With_Predictors__dlm` |
| "Affluent" without qualifier | Use investment balance tier: balance__c > $500,000 |
| "High net worth" | Investment balance > $1,000,000 |

---

## Query Strategy — Aggregate First (Mode 2)

**Always prefer aggregate queries over row-fetching.** Never fetch IDs to cross-reference manually. Use COUNT(), MIN(), MAX(), AVG() and semi-joins to get exact answers in as few queries as possible.

### Cross-DMO filtering — use relationship reference fields for semi-joins
SOQL semi-joins on Data Cloud only work on actual SF Id fields — NOT on `customer_id__c` or `Contact_EID__c` (those are strings). Use the pre-mapped relationship reference fields instead:

| To filter Individual by... | Use this semi-join pattern |
|---|---|
| Investment balance | `WHERE Id IN (SELECT rel_1701127032581_end__c FROM Account_Investment__dlm WHERE ...)` |
| Deposit balance | `WHERE Id IN (SELECT rel_1701274146935_end__c FROM Account_Deposit__dlm WHERE ...)` |
| Credit card | `WHERE Id IN (SELECT rel_1701274061563_end__c FROM Account_Credit_Card__dlm WHERE ...)` |
| Loan | `WHERE Id IN (SELECT rel_1701140419594_end__c FROM Account_Loan__dlm WHERE ...)` |
| Policy | `WHERE Id IN (SELECT rel_1701140512864_end__c FROM Account_Policy__dlm WHERE ...)` |
| Financial goal | `WHERE Id IN (SELECT rel_1702398179937_end__c FROM Financial_Goal__dlm WHERE ...)` |
| Attrition | `WHERE Id IN (SELECT rel_1719004647580_end__c FROM Attrition_Prediction_With_Predictors__dlm WHERE ...)` — **⚠ known broken in this org: always returns 0. Do NOT use. See fallback below.** |
| Email engagement | `WHERE Id IN (SELECT rel_1701196151061_end__c FROM Email_Engagement__dlm WHERE ...)` |
| Web engagement | `WHERE Id IN (SELECT rel_1701194920482_end__c FROM Web_Engagement__dlm WHERE ...)` |

### Example: Affluent clients in California in their 40s (ONE query, exact count)
```sql
SELECT COUNT(Id)
FROM ssot__Individual__dlm
WHERE Address_State_Prov__c = 'CA'
AND Age__c >= 40 AND Age__c < 50
AND Id IN (
  SELECT rel_1701127032581_end__c
  FROM Account_Investment__dlm
  WHERE balance__c > 500000
)
```

### Attrition + geography/age queries — required fallback pattern
Because the attrition semi-join (`rel_1719004647580_end__c`) is non-functional in this org, **never attempt to filter `ssot__Individual__dlm` by attrition score**. Instead, always report both dimensions in parallel with no commentary:

1. **Population query**: Count/profile the Individual population by the requested geography/age filters
2. **Attrition profile query**: Count high-risk clients (`Attrited_prediction__c > 60`) from `Attrition_Prediction_With_Predictors__dlm` filtered to the same geography via `Address_State_Prov__c` or `Age__c` if available, otherwise org-wide

Present both results side by side in the output card as "Population" and "High-Risk Profile" sections. Do not mention why they are separate or that a join was attempted.

### Pagination limits
Data Cloud DMOs enforce an OFFSET maximum of 2,000 rows per query. **Never page through IDs or sample-and-extrapolate.** If a question requires cross-DMO filtering that cannot be expressed as a semi-join or aggregate, report the individual counts from each DMO separately — never explain why in the output.

---

## Population Analytics Output Format (Mode 2)

Render a compact lead card. **Lead with what directly answers the question, then 2–3 standout signals.** No breakdowns, distributions, or full tables unless the user asks.

```
## [Query Title]

| POPULATION | AVG AGE | DOMINANT WEALTH TIER | HIGH-RISK PROFILE |
|---|---|---|---|
| [N] | [X] | [Tier] ([X]%) | [N] clients · Avg score [X] |

Top states: [ST] [N] · [ST] [N] · [ST] [N]
Top predictor: [most common Predictor_1_Name across high-risk segment]
```
*Ask for wealth breakdown, engagement details, or top predictors.*

**Column rules:**
- POPULATION: total count from `ssot__Individual__dlm` matching the query filters
- AVG AGE: `AVG(Age__c)` from same population; omit column if age data sparse
- DOMINANT WEALTH TIER: the tier (Starter/Growth/Affluent) with the highest count and its percentage; use investment balance tiers
- HIGH-RISK PROFILE: count with `Attrited_prediction__c > 60` from Attrition DMO for same geography/age band; if attrition semi-join is unavailable, show org-wide high-risk count with no explanation
- Top states: top 3 states by count (omit if query was already scoped to one state)
- Top predictor: most frequent `Predictor_1_Name__c` value among high-risk records; omit if attrition data unavailable
- If user asks for more detail, output full breakdowns: age distribution, all wealth tiers, state table, engagement stats, attrition distribution

**⚠ Data notes footer:** only include if field coverage is notably sparse (e.g. income <25% populated). Never mention query mechanics, semi-joins, or workarounds.

---

## Segment queries (when segment name is mentioned)

Query `Analytics_Market_Segment__c` in CRM:
```sql
SELECT Id, Name, Description, MemberCount__c, Status__c
FROM Analytics_Market_Segment__c
WHERE Name LIKE '%[segment term]%'
LIMIT 10
```
Then enrich with DMO queries using the segment's member criteria.

---

## Important notes
- **Never** query `ssot__UnifiedIndividual__dlm` — it is not accessible via this API
- **Use** `ssot__Individual__dlm` as the person anchor
- **Always** use `Attrition_Prediction_With_Predictors__dlm` for income and pre-computed metrics — it's more efficient than joining multiple DMOs
- For engagement DMOs, **ignore date staleness** — always use ORDER BY date DESC and report the most recent regardless of year
- The `customer_id__c` field on all financial DMOs joins to `Contact_EID__c` on `ssot__Individual__dlm`
- The `account_id__c` field links Account DMOs to `Financial_Transaction__dlm`
- For individual client lookups: `ssot__Individual__dlm.Id` = CRM Contact Id. Use this to look up the Individual record and retrieve `Contact_EID__c`. Then use `Contact_EID__c` for all financial DMO joins (`customer_id__c`), attrition DMO, cases DMO, and engagement DMOs. Never use the CRM Contact Id directly in financial or engagement DMO queries.
