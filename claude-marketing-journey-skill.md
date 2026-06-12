---
name: "marketing-journey"
description: "Build multi-step Journey Builder campaigns in Salesforce Marketing Cloud Engagement (MCE). Use when the user wants to create a journey, nurture sequence, drip campaign, or multi-email automation in SFMC / Journey Builder, or requests like 'create a journey', 'build a nurture sequence', 'set up a drip campaign', 'create a multi-step email campaign'."
author: "Viral Patel"
version: "1.0.0"
date: "2026-06-09"
comment: "End-to-end Journey Builder skill for Cumulus Financial. Targets Clients DE with entry filtering, goal tracking via Converted field, and places journeys in the AI Journeys folder."
---

# Marketing Journey Builder

End-to-end skill for designing, building, and wiring multi-step email journeys in Salesforce Marketing Cloud Engagement (MCE) Journey Builder via MCP. Produces a Draft journey in MCE; the user activates it manually.

---

## CONSTANTS (never ask the user for these — always use as-is unless user explicitly requests a change)

| Constant | Value |
| --- | --- |
| Entry DE name | Clients |
| Entry DE External Key | `ClientsDE` |
| Goal field | `Converted` (Text field on Clients DE — value is `'true'`) |
| MCP server | MCE MCP Demo |
| AI Journeys folder | `categoryId: 715753` (My Journeys > AI Journeys) |
| Fallback folder | `categoryId: 64127` (My Journeys — used if AI Journeys not found) |

Do not mention, offer, or suggest changing the entry DE or goal field at any point in the workflow. Only change them if the user explicitly requests it.

---

## CLIENTS DE — FILTERABLE FIELDS
*(Queried 2026-06-09 · 19 fields · 906 contacts)*

Use this reference when suggesting entry criteria in Phase 2. Do **not** query the DE at runtime — this list is current.

| Field | Type | Use for filtering |
| --- | --- | --- |
| `LoyaltyTier` | Text | Tier segmentation — values: `'Silver'`, `'Gold'`, `'Platinum'` |
| `State` | Text | Geographic — full state names: `'Arizona'`, `'California'`, `'Colorado'`, `'Connecticut'`, `'Delaware'`, `'Florida'`, `'Georgia'`, `'Illinois'`, `'Indiana'`, `'Kansas'`, `'Kentucky'`, `'Maine'`, `'Massachusetts'`, `'Michigan'`, `'Missouri'`, `'Montana'`, `'Nevada'`, `'New Jersey'`, `'New York'`, `'North Carolina'`, `'Ohio'`, `'Oklahoma'`, `'Oregon'`, `'Pennsylvania'`, `'Tennessee'`, `'Texas'`, `'Washington'`, `'Wisconsin'` |
| `ProductInterest` | Text | Product affinity — values: `'Home Insurance'`, `'Life Insurance'`, `'Auto Insurance'`, `'Retirement Planning'`, `'Wealth Planning'`, `'Education Planning'`, `'Credit Cards'`, `'Auto Loans'`, `'Mortgage Loans'`, `'Checking'`, `'Savings'` |
| `CreditTier` | Text | Credit segment — values: `'Poor'`, `'Fair'`, `'Good'`, `'Very Good'`, `'Exceptional'` |
| `Sentiment` | Text | Case sentiment — `'Positive'`, `'Negative'`, `'Neutral'` |
| `Income` | Number | Annual income in USD — filter with `>= [value]` |
| `NetWorth` | Number | Net worth in USD — filter with `>= [value]` |
| `Offer` | Text | Specific offer code pre-loaded into the DE |
| `CaseCreatedDate` | Date | Date most recent case was created |
| `CaseClosedDate` | Date | Date most recent case was closed |

*Excluded (identity / advisor / goal — not useful as entry filters): `SubscriberKey`, `Email`, `FirstName`, `LastName`, `AdvisorFirstName`, `AdvisorLastName`, `AdvisorEmail`, `AdvisorPhone`, `Converted`.*

Use journey topic to pick relevant suggestion fields:
- Financial product journeys (mortgage, HYSA, loans) → `ProductInterest`, `LoyaltyTier`, `Income`, `NetWorth`
- Geographic campaigns → `State`
- Re-engagement / win-back → `Sentiment = 'Negative'`
- Event invitations → `LoyaltyTier`, `State`
- Offer-specific → `Offer`
- Credit-focused → `CreditTier`

**Operator mapping** (for converting user filter to MCE XML `Operator` attribute):

| User syntax | MCE XML Operator |
| --- | --- |
| `=` | `Equal` |
| `!=` | `NotEqual` |
| `>=` | `GreaterThanOrEqual` |
| `>` | `GreaterThan` |
| `<=` | `LessThanOrEqual` |
| `<` | `LessThan` |
| `is null` | `IsNull` |
| `is not null` | `IsNotNull` |

---

## WORKFLOW

Follow these phases in exact order.

---

### PHASE 1 — Journey Brief

Derive `USER_INITIALS` from the conversation or Cowork context (first letter of first name + first letter of last name, e.g. Viral Patel → VP). Do not ask the user for their initials.

Ask the user the following in a single message:

> "Let's build your journey. Tell me:
>
> 1. **Topic / Goal** — What is this journey about? (e.g., 'mortgage rate alert nurture', 'HYSA promo', 'event invitation series')
> 2. **Campaign length** — Overall timeframe? (e.g., '2 weeks', '30 days')
> 3. **Re-entry** — Can a contact enter this journey more than once?
>    - **Never** — once entered, never again
>    - **After exiting** — can re-enter only after fully completing the previous run
>    - **Anytime** — multiple simultaneous entries allowed
> 4. **Success goal** — What's the target for this journey? Give a percentage and an action.
>    (e.g., '10% register', '5% sign up', '20% apply', '15% open an account')"

Wait for the user's response. Parse question 4 as:
- `GOAL_PERCENTAGE` — the number only (e.g., `10`)
- `GOAL_LABEL` — the action word or short phrase, title-cased (e.g., `Register`, `Sign Up`, `Apply`)

---

### PHASE 2 — Journey Architecture

Using the user's input, propose a complete journey structure. Present everything in one message.

**Step 2a — Rationale (2–3 sentences)**
Briefly explain the proposed structure: why this number of emails, the split logic, and the pacing.

**Step 2b — Journey Summary Table**

```
Journey: [USER_INITIALS] -- [Journey Topic Title] ([N]-day campaign)
Entry:   Clients DE · Re-entry: [Never / After exiting / Anytime]
Goal:    [GOAL_PERCENTAGE]% [GOAL_LABEL] (converted = true)
Segment: [ENTRY_FILTER or "All Clients DE contacts (906)"]

 Step  Type                   Purpose                            Wait After
 ─────────────────────────────────────────────────────────────────────────
  1    📧 Email               [purpose of email 1]               [N] days
  2    🔀 Engagement Split    Opened Email 1?
        ├─ Yes → Step 3       [purpose — warm follow-up]         [N] days
        └─ No  → Step 4       [purpose — re-engagement]          [N] days
  3    📧 Email               [purpose]                          —
  4    📧 Email               [purpose]                          —
       🏁 Goal Exit           converted = true → exit journey
```

Adapt the table to the actual proposed structure. Omit splits or add more emails as needed. Always include the goal exit line.

**Step 2c — Suggest entry criteria**

Recommend 2–3 concrete segment options from the CLIENTS DE FILTERABLE FIELDS reference. Frame them as choices before the user confirms the full structure:

> "**Who should enter this journey?**
>
> - **Option A** — `[Field] = '[value]'` — [one-sentence rationale]
> - **Option B** — `[Field] = '[value]'` — [one-sentence rationale]
> - **No filter** — all 906 contacts in Clients DE
>
> Which would you like to use?"

When the user confirms a filter, decompose it into three stored components:
- `ENTRY_FILTER_FIELD` — the DE field name exactly as listed in the CLIENTS DE table (e.g., `LoyaltyTier`)
- `ENTRY_FILTER_OPERATOR_XML` — the MCE XML operator from the operator mapping table (e.g., `Equal`)
- `ENTRY_FILTER_VALUE` — the raw value without quotes (e.g., `Gold`)
- `ENTRY_FILTER` — the human-readable description for display (e.g., `LoyaltyTier = 'Gold'`)

If the user chooses **No filter**, set `ENTRY_FILTER_FIELD`, `ENTRY_FILTER_OPERATOR_XML`, and `ENTRY_FILTER_VALUE` to `""`, and `ENTRY_FILTER` to `All Clients DE contacts (906)`.

The filter is passed directly to the MCE Event Definition API in Phase 4c and is enforced at journey entry.

**Step 2d — Ask for confirmation**

> "Does this journey structure work for you? You can:
> - Adjust the number of emails or their purpose
> - Change wait durations
> - Add or remove splits
> - Rename the journey
> - Change the entry filter
>
> Reply with changes or **Confirm** to start building the emails."

Wait for confirmation. Re-propose if the user requests changes. Once confirmed, store:
- `JOURNEY_NAME`: `[USER_INITIALS] -- [Journey Topic Title]` (sanitized, ≤ 60 chars — see Critical Rule 4)
- `JOURNEY_STRUCTURE`: the confirmed step-by-step flow
- `TOTAL_EMAILS`: total number of email activities
- `ENTRY_MODE`: `OnceAndDone` / `SingleEntryAcrossAllVersions` / `MultipleEntries`
- `GOAL_LABEL`: user's action label (e.g., `Register`)
- `GOAL_PERCENTAGE`: numeric target (e.g., `10`)
- `ENTRY_FILTER`, `ENTRY_FILTER_FIELD`, `ENTRY_FILTER_OPERATOR_XML`, `ENTRY_FILTER_VALUE`: from Step 2c

**Architecture rules:**
- Propose 2–5 emails scaled to the campaign length
- Default: include an engagement split after the first email unless the user explicitly requests a linear journey
- Default wait times: 3–5 days for nurture; 1–2 days for urgent/time-sensitive
- Always include the goal definition — field is always `Converted`, label and percentage match user's input
- A WAIT activity must always precede an engagement split — contacts need time to open/click before the split evaluates

---

### PHASE 3 — Email Creation

Create each email in the journey sequence one at a time, in order. Do not move to Phase 4 until all emails are built and their asset IDs are captured.

**For each email [N] of [TOTAL_EMAILS]:**

**Step 3a — Announce the current email**

Output this header block before doing anything else for this email:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📧  Email [N] of [TOTAL_EMAILS]  ·  [JOURNEY_NAME]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Purpose:  [this email's role in the sequence]
Context:  [1 sentence — what came before and/or which audience path this is]
```

Immediately follow with a progress line:

```
Progress: [✅ Email 1]  [🔄 Email 2]  [⬜ Email 3]  [⬜ Assembly]
```

Use ✅ for completed emails, 🔄 for the one currently being built, ⬜ for upcoming steps. Always include Assembly as the final step.

**Step 3b — Invoke the marketing-email skill in journey mode**

Invoke the `marketing-email` skill. Since the journey brief has already established the topic and direction, treat Phase 1 of marketing-email as complete — do not ask "What's the topic?" again. Instead, open Phase 2 of the marketing-email workflow pre-briefed with the following context:

- Journey name and overall goal
- This email's specific purpose within the sequence
- Its position (Email N of TOTAL_EMAILS)
- What the previous email covered (for emails 2+), so the copy does not repeat the same hook or offer
- Any relevant split context (e.g., "this is the re-engagement path — the contact did NOT open Email 1")

Proceed through marketing-email Phases 2 → 3 → 4 → 5 as normal. The user selects creative directions, a banner, approves the preview, and the asset is created in MCE.

**Email naming convention in a journey:**
Use `[USER_INITIALS] - [Journey Topic] - Email [N] - [Purpose]` — for example:
`VP - Mortgage Nurture - Email 1 - Initial Alert`
`VP - Mortgage Nurture - Email 2 - Warm Follow-Up`
Apply character sanitization before creating the asset: replace `&` with `and`, replace `/` with `-`, strip all other non-alphanumeric/space/hyphen characters, collapse double spaces and hyphens. ≤ 70 characters.

**Step 3c — Acknowledge completion and transition**

After marketing-email Phase 5d completes, record:
- `EMAIL_[N]_ASSET_NAME` — confirmed asset name
- `EMAIL_[N]_ASSET_ID` — Content Builder asset ID from the Phase 5d confirmation table
- `EMAIL_[N]_LEGACY_ID` — `legacyData.legacyId` from the same table (required for EMAILV2 journey activity)
- `EMAIL_[N]_SUBJECT` — confirmed subject line

Then output a completion acknowledgment in chat:

```
✅  Email [N] of [TOTAL_EMAILS] saved — "[EMAIL_[N]_ASSET_NAME]"
```

If there are more emails to build, immediately follow with the Step 3a header block for Email [N+1].

If this was the final email, output:

```
✅  All [TOTAL_EMAILS] emails are ready in Content Builder.

Progress: [✅ Email 1]  [✅ Email 2]  [✅ Email 3]  [🔄 Assembly]

Now assembling the journey in MCE — running automatically, no input needed.
```

---

### PHASE 4 — Journey Assembly

Run all steps silently. If any step fails, surface the error and wait for user guidance before continuing.

**Step 4a — Fetch org-specific IDs**

Call `sfmc_get_journeys` with `extras: "activities"` to retrieve an existing published journey that contains a configured EMAILV2 activity. From its email activity's `configurationArguments` and `metaData`, extract and store:
- `SEND_CLASSIFICATION_ID`
- `SENDER_PROFILE_ID`
- `DELIVERY_PROFILE_ID`
- `PUBLICATION_LIST_ID`

These are reused for all email activities in this journey.

**Step 4b — Fetch Clients DE GUID**

Call `sfmc_get_data_extensions` with `{"$search": "Clients"}`. Locate the Clients DE and capture its GUID as `CLIENTS_DE_GUID`.

**Step 4c — Create Event Definition**

First, determine whether a filter applies:

**If `ENTRY_FILTER_FIELD` is set (user chose a segment filter):**

Generate `ENTRY_FILTER_GUID` — a unique UUID using the same timestamp-based approach as `GOAL_FILTER_GUID`. Ensure they differ.

Build `ENTRY_FILTER_XML` from this template (single line, no whitespace between tags):
```
<FilterDefinition><ConditionSet Operator="AND" ConditionSetName="Individual Filter Grouping"><Condition Key="Clients.[ENTRY_FILTER_FIELD]" Operator="[ENTRY_FILTER_OPERATOR_XML]" UiMetaData="{}"><Value><![CDATA[[ENTRY_FILTER_VALUE]]]></Value></Condition></ConditionSet></FilterDefinition>
```

Call `sfmc_create_event_definition` with `body_json`:
```json
{
  "type": "EmailAudience",
  "name": "[JOURNEY_NAME] Entry",
  "eventDefinitionKey": "clients-entry-[TIMESTAMP]",
  "mode": "Production",
  "dataExtensionId": "[CLIENTS_DE_GUID]",
  "filterDefinitionId": "[ENTRY_FILTER_GUID]",
  "filterDefinitionTemplate": "[ENTRY_FILTER_XML — JSON-escaped]",
  "arguments": {
    "serializedObjectType": 3,
    "criteria": "[ENTRY_FILTER_XML — JSON-escaped]",
    "useHighWatermark": false,
    "dataExtensionId": "[CLIENTS_DE_GUID]"
  }
}
```

**If no filter (`ENTRY_FILTER_FIELD` is empty):**

Call `sfmc_create_event_definition` with `body_json`:
```json
{
  "type": "EmailAudience",
  "name": "[JOURNEY_NAME] Entry",
  "eventDefinitionKey": "clients-entry-[TIMESTAMP]",
  "mode": "Production",
  "dataExtensionId": "[CLIENTS_DE_GUID]",
  "filterDefinitionId": "00000000-0000-0000-0000-000000000000",
  "filterDefinitionTemplate": "",
  "arguments": {
    "serializedObjectType": 3,
    "criteria": "",
    "useHighWatermark": false,
    "dataExtensionId": "[CLIENTS_DE_GUID]"
  }
}
```

Capture `EVENT_DEFINITION_KEY` and `EVENT_DEFINITION_ID` from the response.

**Step 4d — Generate trigger JSON**

Call `sfmc_data_extension_trigger` with:
- `event_definition_key`: `EVENT_DEFINITION_KEY`
- `event_definition_id`: `EVENT_DEFINITION_ID`
- `data_extension_id`: `CLIENTS_DE_GUID`
- `schedule_type`: `"runOnce"`

Store the returned trigger JSON as `TRIGGER_JSON`.

**Step 4e — Generate activity JSONs**

For each email activity, call `sfmc_email_activity` with:
- `email_asset_id`: `EMAIL_[N]_LEGACY_ID`
- `email_subject`: `EMAIL_[N]_SUBJECT`
- `send_classification_id`, `sender_profile_id`, `delivery_profile_id`, `publication_list_id`: from Step 4a
- `activity_key`: `EMAILV2-[N]` (e.g., `EMAILV2-1`, `EMAILV2-2`)

For each wait activity, call `sfmc_wait_activity` with the confirmed duration and unit from the journey structure. Use `activity_key`: `WAIT-[N]`.

For each engagement split, call `sfmc_engagement_decision_activity` with:
- `ref_activity_key`: the immediately preceding email activity's key
- `ref_activity_name`: `EMAIL_[N]_ASSET_NAME`
- `stats_type_id`: `1` (Opened) — default; use `2` (Clicked any link) only if user specified click-based
- `yes_path_next`: activity key for the "engaged" path
- `no_path_next`: activity key for the "not engaged" path
- `activity_key`: `ENGAGEMENTDECISION-[N]`

Wire all activity `outcomes` so that each activity's `next` points to the correct downstream activity key, exactly matching the confirmed journey structure from Phase 2.

**Step 4f — Assemble and create the journey**

Before building the body, generate:
- `GOAL_FILTER_GUID` — a unique identifier for the goal filter. Compose it in UUID format: take the journey creation timestamp (Unix milliseconds), convert to hex, and distribute into `xxxxxxxx-xxxx-4xxx-8xxx-xxxxxxxxxxxx` format. Example: timestamp `1749500000000` → hex `197C4A0F180` → `0000197c-4a0f-4180-8000-000000000001`. Use any deterministic method — just ensure it is unique per journey and matches exactly in both `filterResult` and `filterDefinitionId`.

- `GOAL_CRITERIA_XML` — build this exact string, substituting `EVENT_DEFINITION_KEY`:
  ```
  <FilterDefinition><ConditionSet Operator="AND" ConditionSetName="Individual Filter Grouping"><Condition IsEphemeralAttribute="true" Key="Event.DEAudience-[EVENT_DEFINITION_KEY].Converted" Operator="Equal" UiMetaData="{}"><Value><![CDATA[true]]></Value></Condition></ConditionSet></FilterDefinition>
  ```
  When embedding in JSON, escape all double-quotes as `\"` and represent the string on a single line.

Call `sfmc_create_journey_builder_journey` with:
- `journey_name`: `JOURNEY_NAME`
- `entry_type`: `data_extension`
- `entry_mode`: `ENTRY_MODE`
- `channels`: `["email"]`
- `use_existing_event_definition`: `true`
- `use_existing_data_extension`: `true`
- `use_existing_assets`: `true`
- `defaults_email`: `{{Event.[EVENT_DEFINITION_KEY]."Email"}}`

Include in `body_json`:
1. All activity JSONs from Step 4e in the `activities` array
2. `TRIGGER_JSON` in the `triggers` array
3. `"categoryId": 715753` (AI Journeys folder). **Fallback**: if the journey creation returns an error referencing an invalid category or folder (e.g., HTTP 400/404 with category-related message), immediately retry with `"categoryId": 64127` (My Journeys). In the Phase 5 summary, note whether the journey landed in AI Journeys or My Journeys.
4. A goal object using this exact structure:

```json
"goals": [{
  "key": "GOAL",
  "name": "[GOAL_LABEL]",
  "description": "[GOAL_PERCENTAGE]% of contacts [GOAL_LABEL]",
  "type": "Event",
  "outcomes": [],
  "arguments": {
    "startActivityKey": "{{Context.StartActivityKey}}",
    "dequeueReason": "{{Context.DequeueReason}}",
    "lastExecutedActivityKey": "{{Context.LastExecutedActivityKey}}",
    "filterResult": "{{Contact.FilterId.[GOAL_FILTER_GUID]}}"
  },
  "configurationArguments": {
    "schemaVersionId": 32,
    "criteria": "[GOAL_CRITERIA_XML — JSON-escaped single line]",
    "filterDefinitionId": "[GOAL_FILTER_GUID]"
  },
  "metaData": {
    "isExitCriteria": false,
    "conversionUnit": "percentage",
    "conversionValue": "[GOAL_PERCENTAGE as string, e.g. \"10\"]",
    "chainType": "none",
    "configurationRequired": false,
    "iconUrl": "",
    "title": ""
  }
}]
```

Use a unique journey `key` with a timestamp suffix to prevent "Cannot Save" conflicts.

---

### PHASE 5 — Confirmation

After the journey is created successfully, output:

**Journey Summary**

| Field | Value |
| --- | --- |
| Journey Name | [JOURNEY_NAME] |
| Journey ID | [from API response] |
| Journey Key | [from API response] |
| Version | 1 |
| Status | Draft |
| Entry | Clients DE |
| Segment | [ENTRY_FILTER] |
| Re-entry | [ENTRY_MODE human-readable] |
| Goal | [GOAL_PERCENTAGE]% [GOAL_LABEL] (converted = true) |
| Emails | [list EMAIL_[N]_ASSET_NAME for each N] |

Then display the final ASCII flow (same format as Phase 2, with confirmed structure and actual email names filled in).

Then say:

> "Your journey is saved as a **Draft** in Journey Builder.
>
> To activate: open it in MCE Journey Builder and click **Activate**. No emails will send until you do.
>
> [If ENTRY_FILTER is not "All Clients DE contacts":]
> 💡 **Entry filter active**: Configured to admit only contacts where `[ENTRY_FILTER]`. This filter is enforced by the Event Definition — contacts not matching will not enter the journey.
>
> 💡 **Tip**: Preview a test contact through the journey in MCE before activating to verify the flow."

Do NOT offer to publish. Do NOT call any publish or activation API.

---

## CRITICAL RULES

1. **Entry DE is hardwired to Clients DE.** Never offer, suggest, or prompt to change it. Only change if the user explicitly requests a different DE.
2. **Goal field is always `Converted`.** The actual MCE filter criterion is always `Converted = 'true'` (text comparison). Display label and percentage come entirely from the user's Phase 1 answer.
3. **Never publish or activate the journey.** Always leave it as Draft. Always instruct the user to activate manually in MCE.
4. **Journey naming**: `[USER_INITIALS] -- [Journey Topic Title]` — double dash with a space on each side. Never single dash, never no spaces. ≤ 60 characters total. Apply character sanitization silently: replace `&` with `and`, replace `/` with `-`, remove all characters that are not alphanumeric, a space, or a hyphen, then collapse any double spaces or double hyphens into single ones. Never present an unsanitized name.
5. **Email naming in journey context**: `[USER_INITIALS] - [Journey Topic] - Email [N] - [Purpose]` — hyphens throughout, no parentheses. Keep purpose short (2–4 words). ≤ 70 characters total. Apply the same character sanitization rules as journey naming.
6. **Capture all email IDs before Phase 4.** The journey assembly requires `EMAIL_[N]_LEGACY_ID` from every email. Do not begin Phase 4 without all IDs confirmed.
7. **Engagement splits require a preceding WAIT.** Never place an engagement split immediately after an email with no wait between them.
8. **Re-entry mode mapping**: Never → `OnceAndDone`, After exiting → `SingleEntryAcrossAllVersions`, Anytime → `MultipleEntries`.
9. **Journey context for emails.** When invoking `marketing-email` in Phase 3, always pass the full journey context — topic, sequence position, what the previous email covered, and split path context. This keeps copy coherent across all emails in the sequence and prevents repeated hooks or offers.
10. **Always show the Step 3a header and progress line before starting each email.** Never silently begin an email. The user must always see which email is being built and the overall progress tracker before any Phase 2 creative options appear.
11. **One journey per session.** Complete Phases 1–5 fully before starting another journey.
12. **Do not narrate Phase 4 steps.** Run them silently. Only surface output if a step fails.
13. **Journey summary table is mandatory.** Always show the full ASCII flow in Phase 2 before building any emails. Never begin Phase 3 without explicit user confirmation of the structure.
14. **Entry criteria are wired into the Event Definition.** Always pass the filter XML and `filterDefinitionId` in the `sfmc_create_event_definition` call in Phase 4c. Single-condition filters only — if the user requests a multi-condition filter, ask them to pick one primary condition for the API and note that additional conditions can be added in MCE after creation.
15. **Goal GUID must be unique per journey.** Never reuse `GOAL_FILTER_GUID` from a prior journey. Always regenerate from the current timestamp.
