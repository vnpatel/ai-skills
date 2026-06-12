---
name: "marketing-email"
description: "Build and deploy production-ready HTML email assets directly into Salesforce Marketing Cloud Engagement (MCE) Content Builder via the connected MCP server. Use when the user wants to create, draft, design, preview, or publish a marketing email in SFMC / Marketing Cloud / Content Builder, or requests like \"create an email campaign\", \"build an MCE email\", or \"send a rate drop email\"."
author: "Viral Patel"
version: "1.0.0"
date: "2026-06-09"
comment: "Branded HTML email skill for Cumulus Financial. Covers creative direction, banner selection, compliance checks, and Content Builder asset creation via MCE MCP."
---

---

```yaml
name: "marketing-email"
description: "Build and deploy production-ready HTML email assets directly into Salesforce Marketing Cloud Engagement (MCE) Content Builder via the connected MCP server. Use when the user wants to create, draft, design, preview, or publish a marketing email in SFMC / Marketing Cloud / Content Builder, or requests like \"create an email campaign\", \"build an MCE email\", or \"send a rate drop email\"."

```

# Marketing Email Builder

End-to-end skill for creating, previewing, and publishing HTML email assets into Salesforce Marketing Cloud Engagement Content Builder via MCP.

---

## SECTION 1 — COMPLIANCE GUIDELINES

You must apply these rules natively at every stage: content generation (Phase 2), preview generation (Phase 4), and HTML structural building (Phase 5). Scan all inputs and outputs continuously.

### Critical Blockers (Zero-Tolerance - Must fix or block execution)

* **No Financial Guarantees:** Do not generate or allow absolute or promissory language. Terms like "guaranteed," "sure thing," "risk-free," "can't lose," or "guaranteed returns" are strictly forbidden. If found, automatically rewrite the copy to be consultative before showing it to the user.
* **Mandatory Footer Elements:** The email structure must contain the physical address (**Cumulus Financial, 111 Monument Circle, Indianapolis, IN 46204**), the text "Member FINRA/SIPC", and functional unsubscribe tokens (`%%subscription_center_url%%` and `%%unsub_center_url%%`). If any of these are missing, block Phase 5 asset creation immediately.
* **Data Privacy Protective Guardrails:** Never include or accept full SSNs, full account numbers, or equivalent sensitive PII. If an account reference is requested, force a masked format: "account ending in ****956".
* **No Inappropriate Language:** Swear words, offensive terms, or derogatory language (e.g., "stupid," "idiot") must be filtered out and rewritten automatically.

### High-Latitude Warnings (Auto-fix silently if Claude-generated; only surface if user-provided content triggered the flag)

* **Tone and Urgency Check:** Maintain a professional, empathetic tone. Flag and point out language that creates panic or excessive FOMO (e.g., "Act NOW or this opportunity expires forever"). Timely urgency is acceptable; alarmist text is not.
* **Spam Trigger Prevention:** Flag excessive punctuation (!!!, ???), ALL CAPS in body text, aggressive or clashing colors, and inconsistent formatting.
* **Link Integrity:** Ensure all links are full, readable URLs. Flag shortened URLs (bit.ly, tinyurl, etc.) as non-compliant.
* **Visual Balance:** Maintain a high text-to-image ratio. Ensure every `<img>` tag has a descriptive `alt` attribute. Flag any image missing an alt description.
* **Layout Readability:** Force short, scannable paragraphs. Verify that mobile viewport text sizes are legible (minimum 14px body font). Flag dense, heavy blocks of text.
* **Factual Claims Check:** Flag specific phrases in the generated copy that are exaggerated or promissory without substantiation. Do not flag generic motivational language. Only flag specific text actually present in the output.
* **Grammar & Mechanics:** Automatically fix clear spelling errors and broken grammar in generated copy before presentation. Do not flag stylistic or intentional brand voice choices.

---

## SECTION 2 — BRANDING GUIDELINES

Use these explicit design configurations as your styling single source of truth when generating the Phase 4 Preview and Phase 5 Production HTML.

### Logo & Identity Assets

* **Primary Logo URL:** `[https://cumulusfinserv-ad61ddfc9e8c.herokuapp.com/images/cumulus-logo.png](https://cumulusfinserv-ad61ddfc9e8c.herokuapp.com/images/cumulus-logo.png)`
* **Usage Rule:** Always hardcode this primary logo URL string **UNLESS** the user explicitly commands or provides a specific alternative logo URL. Never ask or offer to change it; only change it if explicitly provided by the user.
* **Placement:** Position in both the email header logo bar and the footer logo bar.
* **Sizing:** Max width 600px, responsive.
* **Container Background:** **Always `#ffffff` (white)**. The logo container bars (both header and footer) must explicitly use a white background. They must never inherit the outer email background color or use any other brand color.

### Color Palette Matrix

* **Primary (`#1a5276`):** Use for buttons, alert bar backgrounds, benefit badges, CTAs, borders, and text links.
* **Accent (`#aed6f1`):** Use for alert bar text and light design highlights.
* **Background (`#f4f4f0`):** Use for the main email shell and outer background wrapping container.
* **Text (`#2c3e50`):** Use for all primary body copy.
* **Heading (`#1a3a5c`):** Use for greetings, subheads, and the advisor sign-off name.
* **Callout BG (`#eaf4fb`):** Use for the Offer Text callout box background canvas.
* **Divider (`#e8ecef`):** Use for the footer separator line and major section dividers.
* **Divider (Thin) (`#1a5276` at 18% opacity):** Use as a thin rule separating the body copy from the benefits grid.

### Typography Specifications

* **Greeting Text:** Georgia, serif | 24–25px | Bold | Color: `#1a3a5c`
* **Body Copy:** Arial, sans-serif | 15px | Normal | Color: `#2c3e50` | Line-height: 1.7
* **Benefits Subhead:** Arial, sans-serif | 14px | Bold | Color: `#1a3a5c`
* **Offer Callout Text:** Arial, sans-serif | 14px | Italic
* **CTA Button Text:** Arial, sans-serif | 16px | Bold | Color: `#ffffff` (White)
* **Legal / Footer Copy:** Arial, sans-serif | 11–12px | Normal | Color: Muted / Gray

### Structural Component Styles

* **Alert Bar:** Background `#1a5276` | Text `#aed6f1`, uppercase | Default Copy: `RATE ALERT — ACT NOW` (adjust keywords contextually to match campaign topic).
* **CTA Button:** Background `#1a5276` | Text: White, 16px bold Arial | Padding: 14px top/bottom, 36px left/right | Shape: Rounded corners | **Constraint:** Always hardcode an MSO VML roundrect fallback structure for Outlook rendering compatibility.
* **CTA Button Override Color:** Always apply `#2B93D5` as the baseline background color for the CTA button in both Phase 4 and Phase 5 rendering **UNLESS** the user explicitly commands or provides an alternative primary color hex code. Never ask or offer to change it; only change it if explicitly requested by the user.
* **Benefit Badges:** Shape: Circular, 26px diameter | Background: `#1a5276` | Text: White | System Icons: Use `↓` for lower payments / savings, and `↑` for buying power / equity.
* **Offer Text Callout Box:** Background `#eaf4fb` | Left Border: 4px solid `#1a5276` | Font: Arial 14px italic | Content Rule: Display the `%%Offer%%` token by itself inside this container without any adjacent descriptive or label copy.
* **Header & Footer Logo Bars:** Background must be strictly `#ffffff` (white) with no exceptions. The footer bar must include a `#e8ecef` top border separator.

### Layout Email Specifications

* **Dimensions:** Fixed width at exactly 600px max.
* **Mobile Framework:** Responsive breakpoint set at 620px.
* **Coding Architecture:** Rigorous table-based layout (`<table>`, `<tr>`, `<td>`) incorporating explicit MSO/Outlook VML conditional comments.
* **CSS Style Application:** Inline CSS for all layout-critical properties. No external sheets or global style block reliance for structure.
* **Outer Canvas Background:** `#f4f4f0`

### Brand Voice Execution

* **Tone:** Reassuring, professional, timely, consultative.
* **Compliance Restriction:** **Never include specific interest rate percentages or numeric rate values** in any generated copy.
* **Personalization Rules:** Map `%%FirstName%%` directly into greetings and subject lines. Reference advisor details using `%%AdvisorFirstName%%`, `%%AdvisorLastName%%`, and `%%AdvisorEmail%%`.

---

## CONSTANTS (never ask the user for these — always use as-is)

| Constant | Value |
| --- | --- |
| Header logo URL | `[https://cumulusfinserv-ad61ddfc9e8c.herokuapp.com/images/cumulus-logo.png](https://cumulusfinserv-ad61ddfc9e8c.herokuapp.com/images/cumulus-logo.png)` |
| Firm name & address | Cumulus Financial, 111 Monument Circle, Indianapolis, IN 46204 |
| Source DE name | Clients |
| DE External Key | `ClientsDE` |
| DE ID | `ClientsDE` |
| Sendable field | ID → _SubscriberKey |
| Asset type | HTML Email — assetType 208 (NEVER 207) |
| MCP server | MCE MCP Demo |

---

## TOKEN MAP (hardcoded to Clients DE — never ask user to provide tokens)

| Purpose | DE Field | Token |
| --- | --- | --- |
| First name | FirstName | `%%FirstName%%` |
| Last name | LastName | `%%LastName%%` |
| Advisor first name | AdvisorFirstName | `%%AdvisorFirstName%%` |
| Advisor last name | AdvisorLastName | `%%AdvisorLastName%%` |
| Advisor email | AdvisorEmailAddress | `%%AdvisorEmail%%` |
| Dynamic offer | Offer | `%%Offer%%` |
| Send address | Email | `%%Email%%` |

Token syntax is ALWAYS `%%FieldName%%` — never square brackets, never spaces in field names. Derive from the table above only. System tokens (`%%subscription_center_url%%`, `%%unsub_center_url%%`, `%%view_email_url%%`) are used as-is and are not part of the DE token map.

---

## WORKFLOW

Follow these phases in exact order.

---
### PHASE 1 — Gather Topic
Ask the user one open-ended question:
> "What's the general idea, plan, or topic for this email? (e.g., 'mortgage rate drop alert', 'auto insurance bundle discount', 'wealth portfolio review reminder', or 'high-yield savings account promo')"

Wait for their response before proceeding. Do not ask them if they have a custom logo or brand color.
---

### PHASE 2 — Generate & Confirm Creative Directions

Using the user's topic, generate and present the following for approval. Present all options in a single message; do not ask one at a time. Do not ask or offer options to alter the logo or brand colors here.

**Presentation format:** Present each section with a bold title line (e.g., `**A. Asset Name**`) followed immediately by a markdown table. Tables use three columns: `#`, the relevant content label (e.g., `Name`, `Subject Line`, `Preheader`, `Button Text`), and a blank-header third column for the recommended indicator. Bold the recommended row's content and place ⭐ in its third column cell. All other rows leave the third column empty. Never use numbered text lists for A–D. Body copy (E) is always a prose draft block, not a table.

Before presenting, run a silent compliance check against Section 1. Automatically correct any Critical Blocker violations. For High-Latitude Warnings: if the flagged content was Claude-generated, silently revise it to resolve the issue before presenting — do not mention it. Only surface a High-Latitude Warning if the flagged content was explicitly written or provided by the user, in which case flag it inline and let the user decide.

**A. Asset Name (2 options)**

* Meaningful, related to subject line
* Format: `[User Initials] - [Descriptive Title]` (e.g., `JD - Rate Drop Alert for Spring Buyers`)
* Derive the user's initials from the conversation or Cowork context: first letter of first name + first letter of last name (e.g., John Doe → JD)
* ≤ 70 characters total
* **Character sanitization (apply silently before presenting):** Replace `&` with `and`, replace `/` with `-`, remove all other characters that are not alphanumeric, a space, or a hyphen (`"`, `'`, `<`, `>`, `@`, `#`, `%`, `!`, `(`, `)`, etc.). Collapse any resulting double spaces or double hyphens into single ones. Never show an unsanitized name to the user.

**B. Subject Line (3 options)**

* ≤ 50 characters, mobile-optimized
* Must include `%%FirstName%%`
* Emoji encouraged but not required
* No specific interest rate percentages or numeric rate values

**C. Preheader (3 options)**

* 85–100 characters
* Complements but does not repeat subject line

**D. CTA Button Text (2 options)**

* Action-oriented, ≤ 5 words

**E. Body Copy (full draft)**

* Tone: reassuring, professional, timely, consultative
* Word count: 120–170 words (excluding subject, preheader, footer)
* Compliance: NO specific interest rate percentages or numeric rate values
* Required structure:
1. Greeting → `%%FirstName%%`
2. Hook → announce the topic/offer, urgency to act
3. Benefit 1 → lower monthly payments (adapt to topic)
4. Benefit 2 → greater buying power (adapt to topic)
5. Advisor bridge → `%%AdvisorFirstName%%` `%%AdvisorLastName%%`
6. CTA reference (the button will follow)
7. Dynamic offer callout → reference to `%%Offer%%`
8. Sign-off → `%%AdvisorFirstName%%` `%%AdvisorLastName%%` and `%%AdvisorEmail%%`



Then ask:

> "Here are the creative directions I've drafted. Please:
> 1. Select or revise the **Asset Name** (or confirm one)
> 2. Select or revise the **Subject Line**
> 3. Select or revise the **Preheader**
> 4. Select or revise the **CTA Button Text**
> 5. Give feedback on the **Body Copy** — approve it or request changes
> 
> 
> Once you've confirmed all five, I'll pull the most relevant banner options and show them side by side in the email preview so you can select your preferred banner in context."

Wait for full confirmation of all five items before proceeding.

---

### PHASE 3 — Banner Image

**Step 3a — Fetch banners from Content Builder (run silently, no narration)**

Attempt to locate banner images using the following priority order:

**Primary path — folder traversal:**
1. Call `sfmc_get_content_categories` to find the category named **"2 - Images"**.
2. Within its children, locate the subcategory named **"Banners"** and capture its `id`.
3. Call `sfmc_get_content_assets` with `{"$filter": "category.id eq <BannersId>"}` to retrieve all assets in that folder.

**Fallback path — tag search (use if folder traversal fails or returns 0 results):**
- Call `sfmc_search_content_builder_assets` with a query filtering by tag **"Approved Banners"** to retrieve banner assets.

**Step 3b — Select 2–3 relevant banners**

From the returned asset list, analyze each asset's **file name** against the email topic confirmed in Phase 1. Select the **2–3 most topically relevant** banners by matching keywords in the file name to the campaign subject (e.g., "rate", "mortgage", "home", "savings", "auto", "wealth", "portfolio", etc.). Use `fileProperties.publishedURL` as the image URL for each selected asset.

**Step 3c — Present banner picker**

Call `show_widget` with the HTML below. Substitute actual values for all placeholders before rendering. If only 2 banners were found, omit the third `.option` block entirely.

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: Arial, sans-serif; background: #e0e0e0; padding: 24px; }
  .instruction { background: #2c2c2c; color: #fff; font-size: 13px;
                 padding: 12px 16px; border-radius: 4px; margin-bottom: 20px; line-height: 1.6; }
  .gallery { display: flex; gap: 16px; flex-wrap: wrap; }
  .option { background: #fff; border-radius: 6px; overflow: hidden; width: 260px; }
  .option img { width: 260px; height: 160px; object-fit: cover; object-position: center; display: block; }
  .label { background: #1a5276; color: #fff; font-size: 12px; font-weight: bold; padding: 7px 12px; }
  .filename { font-size: 11px; color: #666; padding: 6px 12px 10px; }
</style>
</head>
<body>
  <div class="instruction">
    Banner options for your <strong>[TOPIC]</strong> campaign —
    type <strong>Option 1</strong>, <strong>2</strong>, or <strong>3</strong> in chat to select, or paste your own URL.
  </div>
  <div class="gallery">
    <div class="option">
      <img src="[URL_1]" alt="[FILENAME_1]"/>
      <div class="label">Option 1</div>
      <div class="filename">[FILENAME_1]</div>
    </div>
    <div class="option">
      <img src="[URL_2]" alt="[FILENAME_2]"/>
      <div class="label">Option 2</div>
      <div class="filename">[FILENAME_2]</div>
    </div>
    <div class="option">
      <img src="[URL_3]" alt="[FILENAME_3]"/>
      <div class="label">Option 3</div>
      <div class="filename">[FILENAME_3]</div>
    </div>
  </div>
</body>
</html>
```

After calling `show_widget`, say in chat:

> "Banner options are ready in the preview — type **Option 1**, **2**, or **3** to select, or paste your own URL."

**Step 3d — Collect selection**

- If the user selects an option number → use that asset's `fileProperties.publishedURL` as `BANNER_URL`.
- If the user pastes their own URL → use that URL as `BANNER_URL`.

Once `BANNER_URL` is confirmed, proceed to Phase 4. Do not prompt for a banner description or alternative branding here.

---

### PHASE 4 — Visual Preview

Using `show_widget`, render a realistic inbox preview of the email. All brand colors, fonts, and component styles must come from Section 2.

* **Logo Rendering Rule:** Use the default Header logo URL from CONSTANTS **unless** the user explicitly updated or provided their own custom logo URL during any previous phase conversation. If a custom logo URL was supplied, render that instead.
* **CTA Color Rendering Rule:** The CTA button must use the default background color (`#2B93D5`) **unless** the user explicitly commanded a specific color modification or provided their own hex code. If a custom color was explicitly supplied, render that custom color instead.

Build one full email using the selected `BANNER_URL` from Phase 3. Use the following HTML scaffold **exactly**:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  html { width: 100%; }
  body { width: 100%; background: #e0e0e0; padding: 24px; overflow-y: auto; }
  .center-col { width: 600px; max-width: 600px; margin: 0 auto; }
  .meta-strip {
    background: #2c2c2c; color: #ffffff; font-family: Arial, sans-serif;
    font-size: 11px; line-height: 1.9; padding: 10px 14px;
    margin-bottom: 12px; border-radius: 3px;
  }
</style>
</head>
<body>
  <div class="center-col">

    <div class="meta-strip">
      <div><strong>Asset:</strong> [asset name]</div>
      <div><strong>DE:</strong> Clients</div>
      <div><strong>Subject:</strong> [subject line]</div>
      <div><strong>Preheader:</strong> [preheader]</div>
    </div>

    <table width="600" cellpadding="0" cellspacing="0"
           style="width:600px;max-width:600px;border-collapse:collapse;
                  border:2px solid #c0c0c0;background:#f4f4f0;">
      <!-- all email rows here -->
    </table>

  </div>
</body>
</html>
```

**Scaffold rules (never violate):**
* **`<body>`**: `width:100%`, `background:#e0e0e0`, `padding:24px` uniformly, `overflow-y:auto` — these four properties together ensure the gray fills the full viewport width and the page scrolls vertically. Never use `min-height:100vh` or `overflow:hidden`.
* **`.center-col`**: `width:600px;max-width:600px;margin:0 auto` — centers the 600px column inside the full-width body.
* **Meta strip**: always `background:#2c2c2c` with `color:#ffffff` — never changes between runs. `margin-bottom:12px` creates a visible gap above the email shell. Shows asset name, DE name, subject, preheader — each on its own line.
* **Email shell**: `<table width="600">` with `style="width:600px;max-width:600px;"` — both the HTML attribute and inline style are required. `border:2px solid #c0c0c0` for preview only; never in Phase 5 production HTML.
* Do NOT use `zoom`, `transform:scale`, columns, or any shrinking technique — render the email at full 600px width.

Email shell content:
* **Header logo row**: white (`#ffffff`) background. Logo matches active logo rule.
* **Hero image**: `<img src="BANNER_URL">` — 600px wide, 280px tall, object-fit cover, descriptive alt text.
* **Alert bar**: background `#1a5276`, text `#aed6f1`, uppercase.
* **Greeting**: `Hi %%FirstName%%,` — Georgia serif bold, `#1a3a5c`.
* **Body copy**: confirmed copy from Phase 2 with all tokens as green badges.
* **Thin divider**: `#1a5276` at 18% opacity.
* **Benefits subhead** + two benefit rows with brand circular badges (↓ lower payments, ↑ buying power).
* **CTA button**: confirmed CTA text, `#2B93D5` background (or user-overridden color), rounded, MSO VML fallback.
* **Offer callout box**: background `#eaf4fb`, left border 4px solid `#1a5276` — `%%Offer%%` token badge only, no surrounding label text.
* **Sign-off**: `%%AdvisorFirstName%%` `%%AdvisorLastName%%` (bold) + `%%AdvisorEmail%%` (link).
* **Footer logo bar**: white background, `#e8ecef` top border, active logo.
* **Legal footer**: Cumulus Financial address, Member FINRA/SIPC, `%%subscription_center_url%%`, `%%unsub_center_url%%`, `%%view_email_url%%` as token badges. No token legend below.

All `%%token%%` references must appear as styled green badges:

```html
<span style="background:#d4edda;color:#155724;padding:1px 5px;border-radius:3px;font-family:monospace;font-size:11px;">%%FieldName%%</span>
```

After calling `show_widget`, say in chat:

> "Preview is ready. Does this look correct? Reply **Yes** to create the asset in MCE, or let me know if you'd like any changes."

---

### PHASE 5 — MCE Asset Create / Overwrite (only after explicit "Yes")

#### Step 5a — Pre-flight (run silently, no narration)

**Locate the Emails folder:**
Call `sfmc_get_content_categories` and find the category named **"Emails"** (top-level Content Builder folder). Capture its `id` as `EMAILS_FOLDER_ID`. If no exact match is found, set `EMAILS_FOLDER_ID = null` (root/default folder). This ID is used in Step 5c.

**Check for existing asset:**
Search Content Builder for the confirmed asset name using `sfmc_search_content_builder_assets`.

* If found → note the Asset ID for overwrite
* If not found → will create fresh

**Compliance check:**
Run a final compliance pass against Section 1 on the confirmed subject line, preheader, and body copy. If any Critical Blocker violation is found, stop and alert the user — do not proceed to 5b until resolved.

#### Step 5b — Build full HTML payload

Construct a production-ready HTML email using the brand specifications defined in Section 2.

* **Active Logo Payload Rule:** Use the default Header logo URL from CONSTANTS **unless** the user explicitly updated or provided their own custom logo URL during the conversation.
* **Active CTA Color Payload Rule:** The CTA button background color must be explicitly hardcoded as `#2B93D5` **unless** the user explicitly provided their own custom color/hex code during the conversation.

Structure (in order):

1. Preheader hidden text (with zero-width non-joiners, max 200 chars)
2. Hero image — user-supplied `BANNER_URL` hardcoded as `<img>` tag with descriptive alt text
3. Alert bar — background `#1a5276`, text color `#aed6f1`, uppercase
4. Header logo row — white (`#ffffff`) background table cell; logo is the active logo payload as a hardcoded `<img>` (no AMPscript, no token). Use a different background only if the user explicitly instructed one for this email.
5. Greeting — `Hi %%FirstName%%,` — Georgia bold, `#1a3a5c`
6. Hook paragraph — Arial 15px, `#2c3e50`, line-height 1.7
7. Thin divider — `#1a5276` at 18% opacity
8. Benefits subhead — Arial 14px bold, `#1a3a5c`
9. Benefit row 1 (↓) — brand circular badge + copy
10. Benefit row 2 (↑) — brand circular badge + copy
11. Advisor bridge paragraph — `%%AdvisorFirstName%%` `%%AdvisorLastName%%`
12. CTA button — active CTA background color payload, brand button style with MSO VML roundrect fallback
13. Offer callout — background `#eaf4fb`, left border 4px solid `#1a5276`; display `%%Offer%%` token by itself only, with no surrounding label text or introductory headings
14. Sign-off — `%%AdvisorFirstName%%` `%%AdvisorLastName%%` (bold) + `%%AdvisorEmail%%` (link)
15. Footer logo row — white (`#ffffff`) background table cell; active logo payload; `#e8ecef` border-top separator line
16. Legal footer — Cumulus Financial, 111 Monument Circle, Indianapolis, IN 46204 | Member FINRA/SIPC, compliance copy, and system strings: `%%subscription_center_url%%`, `%%unsub_center_url%%`, `%%view_email_url%%`

**Technical requirements:**

* 600px fixed width, mobile responsive (breakpoint 620px)
* Table-based layout with MSO/Outlook VML conditional comments
* Inline CSS for all layout-critical styles
* Every `<img>` must have a descriptive `alt` attribute

**Hero image execution style:**

```html
<img src="BANNER_URL" alt="[descriptive alt text]" width="600" style="width:100%;max-width:600px;height:300px;object-fit:cover;object-position:center top;display:block;"/>

```

Replace `BANNER_URL` with the user-supplied URL from Phase 3.

#### Step 5c — Execute API call

Use `sfmc_update_content_builder_asset` (if asset exists) or `sfmc_create_email` (if new), always with the `body_json` parameter payload:

```json
{
  "name": "<confirmed asset name>",
  "category": {"id": <EMAILS_FOLDER_ID>},
  "assetType": {"id": 208},
  "channels": {"email": true},
  "data": {
    "email": {
      "options": {"trackOpens": true}
    }
  },
  "views": {
    "subjectline": {"content": "<confirmed subject line>"},
    "preheader": {"content": "<confirmed preheader>"},
    "html": {"content": "<full production HTML>"}
  }
}

```

NEVER use separate named parameters for subject or HTML. ALWAYS pass everything through `body_json`. Include the `category` field when `EMAILS_FOLDER_ID` is set. **Fallback**: if `EMAILS_FOLDER_ID` is null, omit the `category` field entirely (asset will land in the Content Builder root). Also, if creation fails with a category-related error (e.g., invalid folder ID), retry without the `category` field and note in the Step 5d confirmation that the asset was stored in the Content Builder root because the Emails folder was not found.

#### Step 5d — Confirm to user

Return a confirmation summary layout:

| Field | Value |
| --- | --- |
| Asset Name |  |
| Asset ID |  |
| Customer Key |  |
| Legacy ID |  |
| Asset Type | HTML Email (208) |
| Status |  |
| Modified |  |

---

## CRITICAL RULES (never violate)

1. **Compliance first**: Apply Section 1 compliance rules checks natively at Phases 2, 4, and 5. Critical Blocker violations are complete hard stops. High-Latitude Warnings must be surfaced directly to the user.
2. **Brand Values Engine**: Always apply values directly from Section 2. Never guess or hallucinate styles. Header logo URL and firm address are constants derived solely from the CONSTANTS table unless explicitly modified by the user.
3. **No Proactive Prompts for Branding Overrides**: You must **NEVER** ask or offer to change the logo or primary brand color anywhere in the text prompts or workflow. You must only process custom entries if the user proactively commands or supplies an explicit custom logo URL or hex color string themselves.
4. **Logo Canvas Background**: Always target `#ffffff` (white) for both header and footer logo rows unless explicit single-use user override instructions are received.
5. **Token Formatting Syntax**: Always output `%%FieldName%%`. No square brackets, no interior whitespace. Use the exact token keys mapped in Section 2.
6. **Asset Type Constraint**: Force asset type ID 208 (HTML Email). Value 207 is completely banned.
7. **API Architecture**: Always send asset payloads inside the integrated `body_json` wrapper block. No separate top-level fields for HTML blocks or subjects.
8. **Identity Security Rule**: Use the constant Header logo URL for logo imagery unless a custom override logo is explicitly requested and provided by the user. Never map arbitrary text fields or non-logo tokens into the corporate identity elements.
9. **Banner Management**: `BANNER_URL` is sourced either from the user's Content Builder selection (Phase 3, Step 3d) or a user-supplied custom URL — whichever the user provides. This URL must be hardcoded structurally within the `<img>` source tags in both the Phase 4 preview and the Phase 5 production HTML. Do not output empty `<img>` tags.
10. **Alt Attribute Obligation**: Every structural email image tag must contain descriptive alt copy text.
11. **Legal Baseline**: Retain Cumulus Financial, 111 Monument Circle, Indianapolis, IN 46204 | Member FINRA/SIPC, and valid opt-out center tags in the footer block of every variant.
12. **Approval Phasing**: Do not fire any asset creation or updates via API without an explicit clean "Yes" validation string from the user post-preview phase.
13. **Copy limits**: The main core body draft text length constraint must remain bounded between 120 and 170 words (omitting header, subjects, preheaders, and footers).
14. **Button Color Rule**: Force the CTA background color styling property to `#2B93D5` unless the user explicitly dictates their own specific override hex color. Do not fall back to other palette defaults.
15. **Isolated Offer Presentation**: Output the raw `%%Offer%%` token by itself within the left-bordered callout container. Do not generate nearby introduction headers or surrounding contextual sentences inside that specific box.
16. **Preview Meta Strip Spec**: Always `background:#2c2c2c` with `color:#ffffff` — this color is fixed and never changes between runs. The meta strip sits inside the 600px center column with `margin-bottom:12px` to create a visible gap above the email shell. Show asset name, DE name, subject line, and preheader — each on its own line. Do not include a token syntax note.
17. **Preview Footer Restriction**: Do not include a token legend or compliance summary section below the email preview. Compliance issues (if any) are surfaced inline during Phase 2 only.
18. **Preview Layout Constraints**: The preview outer wrapper must use `padding:24px` on all four sides uniformly (never uneven). The email shell must be rendered as `<table width="600">` with both `width:600px` and `max-width:600px` set in the inline style — both the HTML attribute and the inline style are required together to prevent the table from expanding beyond 600px in the browser. The `2px solid #c0c0c0` border is applied to the email shell table for preview only and must never appear in the production HTML payload sent to MCE.

```


```
