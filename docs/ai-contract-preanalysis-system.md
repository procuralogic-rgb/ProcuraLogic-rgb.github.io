# ProcuraLogic -- AI-Assisted Contract Pre-Analysis System

**Last Updated:** April 6, 2026
**Make.com Org:** 7111421 | **Team:** 2083504
**Notion Clients DB:** `33402dc7-99de-80b3-b311-f2fe63325922`
**Plan Requirement:** Core ($10.59/mo) -- requires 4th scenario slot (upgrade may be needed)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Notion Schema Additions](#2-notion-schema-additions)
3. [Google Drive Folder Structure](#3-google-drive-folder-structure)
4. [Claude API Prompt Template](#4-claude-api-prompt-template)
5. [Scenario D -- AI Contract Pre-Analysis](#5-scenario-d--ai-contract-pre-analysis)
6. [Scenario E -- Delivery and Follow-Up Automation](#6-scenario-e--delivery-and-follow-up-automation)
7. [Cost Estimates](#7-cost-estimates)
8. [Step-by-Step Setup Guide](#8-step-by-step-setup-guide)
9. [Testing Checklist](#9-testing-checklist)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. System Overview

### Current Workflow (Manual)

```
Client uploads contract
       |
Owner manually reads entire contract (1-3 hours)
       |
Owner writes review from scratch
       |
Delivers via email
```

### New Workflow (AI-Assisted)

```
Client uploads contract (Google Form / Email / Direct Upload)
       |
Notion status set to "In Review"
       |
  [Scenario D triggers]
       |
  Google Drive: Download contract file
       |
  PDF-to-Text extraction (via Make HTTP or Google Docs conversion)
       |
  Claude API: Analyze contract text
       |
  Notion: Write AI draft to "AI Draft Review" field
       |
  Notion: Update status to "AI Draft Ready" (new sub-status)
       |
  Gmail: Notify owner that draft is ready
       |
  Owner reviews/edits AI output (15-30 min instead of 1-3 hrs)
       |
  Owner changes status to "Delivered"
       |
  [Scenario E triggers]
       |
  Gmail: Send final review to client
       |
  Wait 3 days --> Auto-move to "Follow-Up"
       |
  Brevo: Send follow-up email
```

### Estimated Time Savings

| Metric | Before | After |
|---|---|---|
| Review time per contract | 1-3 hours | 15-30 minutes |
| Turnaround to client | 24-48 hours | 2-6 hours |
| Contracts per day capacity | 2-3 | 8-12 |

---

## 2. Notion Schema Additions

You need to add the following properties to the existing **Clients** database (`33402dc7-99de-80b3-b311-f2fe63325922`). The existing schema already has: Name, Email, Phone, Business Name, Service Type, Status, Lead Source, Contract Type, Contract Upload, Drive Folder, Review Document, Engagement Value, Stripe Payment ID, Calendly Event, Session Date, Last Nurture Email, Nurture Start Date, Notes.

### New Properties to Add

| Property Name | Type | Purpose |
|---|---|---|
| **AI Draft Review** | Rich Text | Stores the full AI-generated draft analysis. This is the main output field. |
| **AI Analysis Status** | Select | Tracks AI processing state. Options below. |
| **AI Analyzed Date** | Date | Timestamp when AI analysis completed |
| **Contract Text** | Rich Text | Extracted plain text from the contract PDF (for reference/re-runs) |
| **Delivered Date** | Date | When the final review was sent to client |
| **Follow-Up Due** | Formula | `dateAdd(prop("Delivered Date"), 3, "days")` -- calculates 3-day follow-up |

### AI Analysis Status Options

| Option | Color | Meaning |
|---|---|---|
| Pending | gray | Contract uploaded, waiting for analysis |
| Processing | blue | AI analysis is currently running |
| Draft Ready | orange | AI draft complete, awaiting owner review |
| Owner Reviewing | purple | Owner is editing the draft |
| Approved | green | Owner approved, ready for delivery |
| Failed | red | AI analysis failed, needs manual review |

### How to Add These in Notion

1. Open the **Clients** database
2. Click **+** in the header row (or Properties > + Add a property)
3. For each property above:
   - Name it exactly as shown
   - Set the type as indicated
   - For Select types, add the options with the colors listed
4. For the Formula type (Follow-Up Due), paste: `dateAdd(prop("Delivered Date"), 3, "days")`

### Modified Status Pipeline

Your existing Status field remains unchanged. The new **AI Analysis Status** field runs in parallel:

```
Status: In Review  +  AI Analysis Status: Pending
                           |
Status: In Review  +  AI Analysis Status: Processing
                           |
Status: In Review  +  AI Analysis Status: Draft Ready
                           |
Status: In Review  +  AI Analysis Status: Owner Reviewing
                           |
Status: In Review  +  AI Analysis Status: Approved
                           |
Status: Delivered  (owner changes this manually after review)
```

---

## 3. Google Drive Folder Structure

Use the existing Google connection (ID `8160539`, account `procuralogic@gmail.com`).

### Folder Hierarchy

```
My Drive/
  ProcuraLogic/
    Client Contracts/
      [Client Name - Date]/
        Original/
          contract-original.pdf
        Extracted/
          contract-text.txt
        Reviews/
          ai-draft-review.md
          final-review.pdf
    Templates/
      review-template.md
      email-templates/
```

### Naming Convention

Each client folder: `{LastName}_{FirstName}_{YYYY-MM-DD}`
Example: `Doe_Jane_2026-04-06`

### Make.com Folder ID

After creating the `Client Contracts` parent folder in Google Drive, note its folder ID from the URL:
`https://drive.google.com/drive/folders/{FOLDER_ID}`

You will use this as the parent folder when creating client subfolders in Make.com.

---

## 4. Claude API Prompt Template

This is the core of the system. The prompt is engineered to produce output that matches ProcuraLogic's existing delivery style: plain-language, practical, risk-focused, with clear recommendations.

### System Prompt (Stored in Make.com as a variable)

```
You are a senior contract analyst working for ProcuraLogic, a contract review and advisory service based in Texas. ProcuraLogic is NOT a law firm. You provide contract review and advisory services for educational and informational purposes only.

Your role is to produce a thorough, plain-language contract analysis that helps freelancers, founders, content creators, and small business owners understand what they are signing before they sign it.

IMPORTANT GUIDELINES:
- Write in clear, professional, accessible language. Avoid legal jargon unless you are explaining it.
- Be direct and specific. Do not hedge excessively. State what the clause does and why it matters.
- Focus on practical business impact, not theoretical legal arguments.
- Always note that this is a contract review for informational purposes, not legal advice.
- Flag genuinely problematic clauses clearly. Do not soften real risks.
- When something is standard and reasonable, say so. Do not manufacture concerns.
- Use the client's name and business context when provided.
- Write as if you are explaining this to a smart business owner who is not a lawyer.
- Structure your output in clean Markdown format.
```

### User Prompt Template

This is the per-contract prompt. Variables in `{{double braces}}` are mapped from Notion/Make.com fields.

```
Analyze the following contract and produce a comprehensive review for our client.

CLIENT CONTEXT:
- Client Name: {{client_name}}
- Business Name: {{business_name}}
- Service Purchased: {{service_type}}
- Contract Type: {{contract_type}}

CONTRACT TEXT:
---
{{contract_text}}
---

Produce the analysis using EXACTLY the following structure and section headers. Each section is required.

---

# Contract Review -- {{client_name}}

**Prepared by:** ProcuraLogic
**Date:** {{current_date}}
**Contract Type:** {{contract_type}}
**Service:** {{service_type}}

> **Disclaimer:** This review is provided by ProcuraLogic for informational and educational purposes only. ProcuraLogic is not a law firm and does not provide legal advice. For legal advice specific to your situation, consult a licensed attorney.

---

## 1. Contract Overview

Provide a brief summary covering:
- **Type of Agreement:** (e.g., independent contractor agreement, vendor services agreement, NDA, employment contract, etc.)
- **Parties:** Who is involved (use actual names from the contract)
- **Effective Date:** When does this take effect
- **Term/Duration:** How long does this last, including renewal terms
- **Total Value / Compensation:** Payment amount, structure, and schedule
- **Governing Law:** Which state/jurisdiction governs this contract
- **Purpose:** In 2-3 sentences, what is this contract fundamentally about

## 2. Key Terms Breakdown

For each significant clause or section in the contract, provide:
- **Clause name/title** (as it appears in the contract or a descriptive label)
- **What it says** (plain-language summary)
- **What it means for you** (practical business impact for the client)

Cover at minimum: scope of work, payment terms, termination, intellectual property, confidentiality, non-compete/non-solicitation, indemnification, liability limitations, dispute resolution, and any other notable clauses.

## 3. Risk Assessment

### Red Flags
List any clauses that pose significant risk. For each:
- Name the clause and quote the relevant language
- Explain WHY it is a problem in plain terms
- Rate severity: HIGH / MEDIUM / LOW
- Suggest specific alternative language or negotiation approach

### One-Sided Terms
Identify any provisions that disproportionately favor one party. Explain the imbalance.

### Hidden Obligations
Flag any commitments that might not be immediately obvious -- auto-renewals, minimum purchase requirements, exclusivity windows, post-termination restrictions, etc.

## 4. Financial Terms Analysis

- **Total cost/compensation** with full breakdown
- **Payment schedule and terms** (net-30, milestones, etc.)
- **Late payment penalties or interest**
- **Expense reimbursement provisions**
- **Price escalation or adjustment clauses**
- **Financial exposure** (what is the maximum you could owe or lose under this contract)

## 5. Recommendations and Action Items

Provide a numbered list of specific, actionable recommendations. For each:
1. State what to do (e.g., "Request removal of the non-compete clause in Section 7.2")
2. Explain why
3. Suggest specific replacement language where appropriate
4. Indicate priority: MUST address before signing / SHOULD negotiate / NICE to have

### Summary Verdict

In 2-3 sentences, give your overall assessment: Is this contract reasonable? What are the top 1-3 things the client absolutely should address before signing?

---
```

### Claude API Configuration

| Parameter | Value | Rationale |
|---|---|---|
| Model | `claude-sonnet-4-20250514` | Best balance of quality and cost for contract analysis |
| Max Tokens | 8000 | Typical review is 3,000-6,000 tokens; 8,000 allows for complex contracts |
| Temperature | 0.3 | Low temperature for factual, consistent analysis |
| Top P | 0.9 | Slightly constrained for reliability |

> **Model Choice Note:** Claude Sonnet 4 is recommended for production use. For especially complex or high-value contracts ($299 Strategy Sessions), you can switch to `claude-opus-4-20250514` for deeper analysis at higher cost (~5x). For $99 reviews, Sonnet 4 provides excellent quality at a fraction of the cost.

---

## 5. Scenario D -- AI Contract Pre-Analysis

**Scenario Name:** `ProcuraLogic AI Contract Analysis`
**Trigger:** Notion Watch Database Items (watches for Status = "In Review")
**Estimated Operations per Run:** 6-8

### Architecture Diagram

```
[1. Notion: Watch Database Items]
  (filter: Status changed to "In Review")
       |
[2. Notion: Update Database Item]
  (set AI Analysis Status = "Processing")
       |
[3. Google Drive: Download a File]
  (download contract from Drive Folder URL or Contract Upload)
       |
[4. HTTP: Make a Request]  (PDF to text extraction)
  (POST to pdf.co API or use Google Docs conversion)
       |
[5. HTTP: Make a Request]  (Claude API call)
  (POST https://api.anthropic.com/v1/messages)
       |
   [Router]
    /       \
Route 1      Route 2
(Success)    (Error)
    |            |
[6a. Notion:  [6b. Notion:
 Update Item]  Update Item]
 - AI Draft    - AI Analysis
   Review =      Status = "Failed"
   {{response}} - Notes = error msg
 - AI Analysis
   Status =
   "Draft Ready"
 - AI Analyzed
   Date = now
    |
[7. Gmail: Send Email]
  (notify owner: "AI draft ready for {client_name}")
```

### Module-by-Module Configuration

#### Module 1: Notion -- Watch Database Items

| Setting | Value |
|---|---|
| Connection | My Notion Public connection (ID 8184144) |
| Database | Clients (33402dc7-99de-80b3-b311-f2fe63325922) |
| Watch | Updated Database Items |
| Filter | Status = "In Review" |
| Limit | 1 (process one at a time to manage API costs) |

> **Polling interval:** Set to 15 minutes. This is the same interval as the existing Integration Notion scenario and stays within Core plan limits.

#### Module 2: Notion -- Update a Database Item

Purpose: Mark the record as "Processing" so the owner sees activity.

| Field | Value |
|---|---|
| Page ID | `{{1.id}}` (from the watched item) |
| AI Analysis Status | `Processing` |

#### Module 3: Google Drive -- Download a File

This module gets the contract file. There are two possible sources, handled by a router or conditional logic:

**Option A -- Contract is in Google Drive (Drive Folder field populated):**

| Setting | Value |
|---|---|
| Connection | My Google connection (ID 8160539) |
| Enter a File ID | Select "Map" |
| File ID | Extract from `{{1.Drive Folder}}` URL or use the Contract Upload field |

**Option B -- Contract is a Notion file attachment (Contract Upload field):**

If the contract was uploaded directly to Notion as a file attachment, you need to:
1. Use an HTTP module to download from the Notion file URL
2. The URL is temporary (expires in 1 hour) so process immediately

> **Recommended approach:** Standardize on Google Drive. When a client submits a contract, always save it to Google Drive first, then reference the Drive link in Notion. This avoids Notion's temporary URL issue.

#### Module 4: HTTP -- PDF to Text Extraction

You need to convert the PDF to plain text before sending to Claude. Two options:

**Option A: Using pdf.co API (Recommended -- simpler)**

| Setting | Value |
|---|---|
| URL | `https://api.pdf.co/v1/pdf/convert/to/text` |
| Method | POST |
| Headers | `x-api-key: {{your_pdfco_api_key}}` |
| Body Type | Multipart/form-data |
| Fields | `file` = downloaded file from Module 3 |

> **pdf.co pricing:** Free tier includes 100 credits/month. Each page costs 1 credit. A typical contract (5-20 pages) uses 5-20 credits. This is sufficient for 5-20 contracts/month on the free plan. Paid plans start at $9.99/month for 500 credits.

**Option B: Using Google Docs conversion (Free)**

1. **Google Drive: Upload a File** -- Upload the PDF to a temporary folder
2. Set "Convert to Google Docs format" = Yes
3. **Google Docs: Get Content of a Document** -- Read the text content
4. **Google Drive: Delete a File** -- Clean up the temporary file

This is free but uses 3 extra operations per run.

#### Module 5: HTTP -- Claude API Call

This is the core AI analysis module.

| Setting | Value |
|---|---|
| URL | `https://api.anthropic.com/v1/messages` |
| Method | POST |
| Headers | See below |
| Body Type | Raw |
| Content Type | JSON (application/json) |
| Request Body | See below |

**Headers:**

| Header | Value |
|---|---|
| `x-api-key` | `{{anthropic_api_key}}` (store as a Make.com variable -- never hardcode) |
| `anthropic-version` | `2023-06-01` |
| `content-type` | `application/json` |

**Request Body (JSON):**

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 8000,
  "temperature": 0.3,
  "system": "You are a senior contract analyst working for ProcuraLogic, a contract review and advisory service based in Texas. ProcuraLogic is NOT a law firm. You provide contract review and advisory services for educational and informational purposes only.\n\nYour role is to produce a thorough, plain-language contract analysis that helps freelancers, founders, content creators, and small business owners understand what they are signing before they sign it.\n\nIMPORTANT GUIDELINES:\n- Write in clear, professional, accessible language. Avoid legal jargon unless you are explaining it.\n- Be direct and specific. Do not hedge excessively. State what the clause does and why it matters.\n- Focus on practical business impact, not theoretical legal arguments.\n- Always note that this is a contract review for informational purposes, not legal advice.\n- Flag genuinely problematic clauses clearly. Do not soften real risks.\n- When something is standard and reasonable, say so. Do not manufacture concerns.\n- Use the client's name and business context when provided.\n- Write as if you are explaining this to a smart business owner who is not a lawyer.\n- Structure your output in clean Markdown format.",
  "messages": [
    {
      "role": "user",
      "content": "Analyze the following contract and produce a comprehensive review for our client.\n\nCLIENT CONTEXT:\n- Client Name: {{1.Name}}\n- Business Name: {{1.Business Name}}\n- Service Purchased: {{1.Service Type}}\n- Contract Type: {{1.Contract Type}}\n\nCONTRACT TEXT:\n---\n{{4.text}}\n---\n\nProduce the analysis using EXACTLY the following structure and section headers. Each section is required.\n\n---\n\n# Contract Review -- {{1.Name}}\n\n**Prepared by:** ProcuraLogic\n**Date:** {{formatDate(now; \"MMMM D, YYYY\")}}\n**Contract Type:** {{1.Contract Type}}\n**Service:** {{1.Service Type}}\n\n> **Disclaimer:** This review is provided by ProcuraLogic for informational and educational purposes only. ProcuraLogic is not a law firm and does not provide legal advice. For legal advice specific to your situation, consult a licensed attorney.\n\n---\n\n## 1. Contract Overview\n\nProvide a brief summary covering:\n- **Type of Agreement**\n- **Parties**\n- **Effective Date**\n- **Term/Duration**\n- **Total Value / Compensation**\n- **Governing Law**\n- **Purpose** (2-3 sentences)\n\n## 2. Key Terms Breakdown\n\nFor each significant clause, provide:\n- **Clause name/title**\n- **What it says** (plain-language summary)\n- **What it means for you** (practical business impact)\n\nCover at minimum: scope of work, payment terms, termination, intellectual property, confidentiality, non-compete/non-solicitation, indemnification, liability limitations, dispute resolution.\n\n## 3. Risk Assessment\n\n### Red Flags\nFor each: name the clause, quote relevant language, explain why it is a problem, rate severity (HIGH / MEDIUM / LOW), suggest alternative language.\n\n### One-Sided Terms\nIdentify provisions favoring one party disproportionately.\n\n### Hidden Obligations\nFlag non-obvious commitments: auto-renewals, minimums, exclusivity, post-termination restrictions.\n\n## 4. Financial Terms Analysis\n\n- Total cost/compensation breakdown\n- Payment schedule and terms\n- Late payment penalties\n- Expense reimbursement\n- Price escalation clauses\n- Maximum financial exposure\n\n## 5. Recommendations and Action Items\n\nNumbered list. For each:\n1. What to do\n2. Why\n3. Suggested replacement language if applicable\n4. Priority: MUST address / SHOULD negotiate / NICE to have\n\n### Summary Verdict\n2-3 sentences: overall assessment and top 1-3 things to address before signing."
    }
  ]
}
```

> **Make.com Tip:** In the JSON body, use Make.com's mapping panel to insert `{{1.Name}}`, `{{1.Business Name}}`, etc. from Module 1's output. For the contract text from Module 4, map `{{4.text}}` (or whatever the output field is named from your PDF extraction module). Use the `replace()` function to escape any double quotes or newlines in the contract text: `{{replace(replace(4.text; "\""; "\\\""); newline; "\\n")}}`

**Parsing the Response:**

The Claude API returns JSON. The analysis text is in:
```
{{5.body.content[0].text}}
```

You will also want to check `{{5.body.stop_reason}}` -- it should be `end_turn`. If it is `max_tokens`, the analysis was truncated and you may need to increase `max_tokens` or split the contract.

#### Module 6a: Notion -- Update a Database Item (Success Route)

| Field | Value |
|---|---|
| Page ID | `{{1.id}}` |
| AI Draft Review | `{{5.body.content[0].text}}` |
| AI Analysis Status | `Draft Ready` |
| AI Analyzed Date | `{{now}}` |

> **Character Limit Note:** Notion rich text properties have a 2,000 character limit per block. If the AI review exceeds this, you have two options:
> 1. **Use Notion page content instead:** Add the review as page content (child blocks) rather than a property. Use the Notion "Append Block Children" API call.
> 2. **Store in Google Drive:** Save the review as a file in Google Drive and link it in the Review Document field.
>
> **Recommended:** Use a Make.com HTTP module to call the Notion API directly to append the review as page content blocks. This gives you the full review inside the Notion page. See "Appending Long Text to Notion" in the Troubleshooting section.

#### Module 6b: Notion -- Update a Database Item (Error Route)

| Field | Value |
|---|---|
| Page ID | `{{1.id}}` |
| AI Analysis Status | `Failed` |
| Notes | `AI analysis failed: {{5.statusCode}} - {{5.body.error.message}}. Manual review required.` |

#### Module 7: Gmail -- Send an Email

| Setting | Value |
|---|---|
| Connection | My Gmail connection (ID 8184235) |
| To | `procuralogic@gmail.com` |
| Subject | `AI Draft Ready: {{1.Name}} - {{1.Contract Type}}` |
| Content | See below |

**Email Content:**

```html
<h2>AI Contract Analysis Complete</h2>
<p>A new AI draft review is ready for your review.</p>
<table style="border-collapse:collapse;margin:16px 0;">
  <tr><td style="padding:8px 16px;font-weight:bold;background:#f0fdf4;">Client</td>
      <td style="padding:8px 16px;">{{1.Name}}</td></tr>
  <tr><td style="padding:8px 16px;font-weight:bold;background:#f0fdf4;">Business</td>
      <td style="padding:8px 16px;">{{1.Business Name}}</td></tr>
  <tr><td style="padding:8px 16px;font-weight:bold;background:#f0fdf4;">Service</td>
      <td style="padding:8px 16px;">{{1.Service Type}}</td></tr>
  <tr><td style="padding:8px 16px;font-weight:bold;background:#f0fdf4;">Contract Type</td>
      <td style="padding:8px 16px;">{{1.Contract Type}}</td></tr>
</table>
<p><strong>Next Step:</strong> Open the Notion record, review the AI draft in the "AI Draft Review" field, edit as needed, then change Status to "Delivered" when ready to send.</p>
<p><a href="https://notion.so/{{1.id}}" style="display:inline-block;padding:12px 24px;background-color:#064e3b;color:#fff;text-decoration:none;border-radius:6px;">Open in Notion</a></p>
```

### Error Handling

Add error handlers to critical modules:

| Module | Error Handler | Action |
|---|---|---|
| Module 3 (Drive Download) | Resume | Log error to Notes field, set AI Analysis Status = "Failed" |
| Module 4 (PDF Extract) | Resume | Log error, set status to Failed |
| Module 5 (Claude API) | Resume | Log error with status code, set status to Failed |

For the Claude API module specifically, add a **retry** directive: retry once after 30 seconds. Claude API occasionally returns 529 (overloaded) errors that succeed on retry.

---

## 6. Scenario E -- Delivery and Follow-Up Automation

**Scenario Name:** `ProcuraLogic Delivery & Follow-Up`
**Trigger:** Notion Watch Database Items (watches for Status = "Delivered")
**Estimated Operations per Run:** 4-5

### Architecture Diagram

```
[1. Notion: Watch Database Items]
  (filter: Status changed to "Delivered")
       |
[2. Notion: Get a Page]
  (fetch full page content including the final review)
       |
[3. Gmail: Send an Email]
  (send final review to client)
       |
[4. Notion: Update Database Item]
  (set Delivered Date = now)
       |
[5. Sleep: 3 days]  (Use Make.com Scheduling instead -- see note below)
       |
[6. Notion: Update Database Item]
  (set Status = "Follow-Up")
       |
[7. Brevo: Send an Email]
  (send follow-up email using template)
```

> **Important: The 3-day delay.** Make.com's Sleep module can pause execution, but a 3-day sleep ties up an execution slot and counts against your operations. A better approach is to split this into TWO scenarios:
>
> **Scenario E1 -- Delivery (instant):** Triggers on Status = "Delivered", sends the review email, sets Delivered Date.
>
> **Scenario E2 -- Follow-Up (scheduled):** Runs once daily on a schedule. Queries Notion for all records where Status = "Delivered" AND Delivered Date is 3+ days ago AND Status is not yet "Follow-Up". For each match, updates to "Follow-Up" and sends the follow-up email.
>
> This is more efficient and does not waste execution slots on sleeping.

### Scenario E1 -- Delivery (Instant Trigger)

#### Module 1: Notion -- Watch Database Items

| Setting | Value |
|---|---|
| Connection | My Notion Public connection (ID 8184144) |
| Database | Clients |
| Watch | Updated Database Items |
| Filter | Status = "Delivered" |

#### Module 2: Notion -- Get a Page

| Setting | Value |
|---|---|
| Page ID | `{{1.id}}` |

This fetches the full page content, including the edited review that the owner finalized.

#### Module 3: Gmail -- Send an Email

| Setting | Value |
|---|---|
| Connection | My Gmail connection (ID 8184235) |
| To | `{{1.Email}}` |
| Subject | `Your Contract Review is Ready -- ProcuraLogic` |
| Content | See below |

**Email Content:**

```html
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background-color:#f4f4f5;">
  <tr><td align="center" style="padding:40px 16px;">
    <table role="presentation" width="600" cellpadding="0" cellspacing="0" style="background-color:#ffffff;border-radius:8px;overflow:hidden;max-width:600px;width:100%;">

      <!-- Header -->
      <tr><td style="background-color:#064e3b;padding:32px 40px;text-align:center;">
        <h1 style="margin:0;font-size:22px;font-weight:600;color:#ffffff;letter-spacing:0.5px;">ProcuraLogic</h1>
        <p style="margin:4px 0 0;font-size:12px;color:#a7f3d0;letter-spacing:0.3px;">Contract Clarity & Review</p>
      </td></tr>

      <!-- Body -->
      <tr><td style="padding:40px;">
        <p style="margin:0 0 20px;font-size:16px;line-height:1.6;color:#1e293b;">Hi {{1.Name}},</p>

        <p style="margin:0 0 20px;font-size:16px;line-height:1.6;color:#1e293b;">Your contract review is complete and ready for you. Please find the full analysis below.</p>

        <div style="background-color:#f8fafc;border:1px solid #e2e8f0;border-radius:8px;padding:24px;margin:0 0 24px;">
          {{2.pageContent}}
        </div>

        <p style="margin:0 0 20px;font-size:16px;line-height:1.6;color:#1e293b;">If you have any questions about the findings, or if you would like to discuss them in more detail, feel free to book a follow-up consultation:</p>

        <table role="presentation" cellpadding="0" cellspacing="0" style="margin:0 auto 28px;">
          <tr><td style="background-color:#064e3b;border-radius:6px;">
            <a href="https://calendly.com/procuralogic/clarity-review" style="display:inline-block;padding:14px 32px;font-size:15px;font-weight:600;color:#ffffff;text-decoration:none;">Book a Follow-Up Session</a>
          </td></tr>
        </table>

        <p style="margin:16px 0 4px;font-size:13px;color:#94a3b8;text-align:center;">This review is provided for informational and educational purposes only. ProcuraLogic is not a law firm.</p>
      </td></tr>

      <!-- Footer -->
      <tr><td style="background-color:#f8fafc;padding:24px 40px;border-top:1px solid #e2e8f0;">
        <p style="margin:0;font-size:13px;color:#64748b;text-align:center;">
          <a href="https://procura-logic.com" style="color:#064e3b;text-decoration:none;">procura-logic.com</a>
        </p>
      </td></tr>

    </table>
  </td></tr>
</table>
```

> **Note on email content:** The `{{2.pageContent}}` variable above is a placeholder. In practice, you will want to either:
> 1. Read the AI Draft Review property directly from Module 1's output and format it as HTML
> 2. Attach the review as a PDF (generate via Google Docs > Export as PDF)
>
> For the cleanest client experience, generate a PDF from Google Docs and attach it to the email. Use Google Docs "Create a Document" with the review content, then "Download a Document" as PDF, then attach to Gmail.

#### Module 4: Notion -- Update Database Item

| Field | Value |
|---|---|
| Page ID | `{{1.id}}` |
| Delivered Date | `{{now}}` |

### Scenario E2 -- Follow-Up (Daily Schedule)

**Trigger:** Schedule -- runs once per day at 9:00 AM CT

#### Module 1: Notion -- Search Objects

| Setting | Value |
|---|---|
| Connection | My Notion Public connection (ID 8184144) |
| Database | Clients |
| Filter | Status = "Delivered" AND Delivered Date is before {{addDays(now; -3)}} |

This finds all clients who were delivered 3+ days ago but have not moved to Follow-Up yet.

#### Module 2: Iterator

Iterates over each result from Module 1 (in case multiple clients hit the 3-day mark on the same day).

#### Module 3: Notion -- Update Database Item

| Field | Value |
|---|---|
| Page ID | `{{2.id}}` |
| Status | `Follow-Up` |

#### Module 4: Brevo -- Send an Email

| Setting | Value |
|---|---|
| Connection | ProcuraLogic Brevo (ID 8229992) |
| To | `{{2.Email}}` |
| From Email | `procuralogic@gmail.com` |
| From Name | `ProcuraLogic` |
| Template ID | Use a new Brevo template (Template #8, see below) |
| Template Variables | `client_name` = `{{2.Name}}`, `service_type` = `{{2.Service Type}}` |

**Follow-Up Email Template (create as Brevo Template #8):**

Subject: `Checking in -- how is the contract looking?`

```html
<p>Hi {{params.client_name}},</p>

<p>We wanted to follow up on the contract review we sent a few days ago. We hope you found the analysis helpful.</p>

<p>A few quick questions:</p>
<ul>
  <li>Were you able to address the key recommendations?</li>
  <li>Did the other party push back on any of the changes?</li>
  <li>Do you need help with the negotiation conversation?</li>
</ul>

<p>If you would like to talk through anything, we are here to help. You can book a quick session anytime:</p>

<p><a href="https://calendly.com/procuralogic/clarity-review" style="display:inline-block;padding:12px 24px;background-color:#064e3b;color:#fff;text-decoration:none;border-radius:6px;">Book a Follow-Up Session</a></p>

<p>And if everything is settled and you have signed with confidence, that is great news. We would love to hear about it.</p>

<p>Best,<br>The ProcuraLogic Team</p>
```

---

## 7. Cost Estimates

### Claude API Costs (Per Contract Review)

Based on Claude Sonnet 4 pricing as of April 2026:

| Metric | Estimate | Notes |
|---|---|---|
| Input tokens (system + prompt + contract) | ~4,000-12,000 tokens | Depends on contract length. A 10-page contract is ~4,000 words = ~5,500 tokens |
| Output tokens (analysis) | ~3,000-6,000 tokens | Typical thorough review |
| Input cost per review | $0.012 - $0.036 | At $3/million input tokens |
| Output cost per review | $0.045 - $0.090 | At $15/million output tokens |
| **Total per review** | **$0.06 - $0.13** | Sonnet 4 |

For Claude Opus 4 (premium reviews):

| Metric | Estimate |
|---|---|
| Input cost per review | $0.060 - $0.180 |
| Output cost per review | $0.225 - $0.450 |
| **Total per review** | **$0.29 - $0.63** |

### Monthly Cost Projections

| Volume | Sonnet 4 (API) | Opus 4 (API) | Make.com | PDF Extract | Total |
|---|---|---|---|---|---|
| 10 reviews/month | $0.60 - $1.30 | $2.90 - $6.30 | $10.59 (Core) | Free (pdf.co free tier) | ~$12-18 |
| 25 reviews/month | $1.50 - $3.25 | $7.25 - $15.75 | $10.59 | Free | ~$13-29 |
| 50 reviews/month | $3.00 - $6.50 | $14.50 - $31.50 | $10.59 | $9.99 (pdf.co Starter) | ~$24-48 |

### Revenue vs. Cost Analysis

| Scenario | Revenue | AI Cost | Margin |
|---|---|---|---|
| 10 reviews at $99 | $990 | ~$15 | 98.5% |
| 25 reviews at $99 | $2,475 | ~$20 | 99.2% |
| 10 mixed ($99/$125/$299) | ~$1,500 | ~$15 | 99.0% |

> **Bottom line:** AI pre-analysis costs are negligible relative to service pricing. Even at 50 reviews per month using the most expensive model, total infrastructure costs stay under $50/month against $5,000+ in revenue.

### Make.com Operations Budget

Each AI analysis run uses approximately 6-8 operations. Each delivery run uses 4-5 operations. The Core plan includes 10,000 operations/month.

| Volume | Analysis Ops | Delivery Ops | Total | % of 10K limit |
|---|---|---|---|---|
| 10 reviews/month | 80 | 50 | 130 | 1.3% |
| 25 reviews/month | 200 | 125 | 325 | 3.3% |
| 50 reviews/month | 400 | 250 | 650 | 6.5% |

Operations from AI scenarios will not be a constraint. Your existing scenarios (Integration Notion running every 15 min) already consume the majority of your operations budget.

---

## 8. Step-by-Step Setup Guide

### Phase 1: Prerequisites (30 minutes)

#### Step 1.1 -- Get a Claude API Key

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Create an account or sign in with `procuralogic@gmail.com`
3. Go to **API Keys** > **Create Key**
4. Name: `Make.com Contract Analysis`
5. Copy the key immediately and store in your password manager
6. Add billing: Go to **Billing** > add a payment method
7. Set a usage limit: **Settings > Limits** > set monthly limit to $25 (safety cap)

#### Step 1.2 -- Get a PDF.co API Key (if using Option A for text extraction)

1. Go to [pdf.co](https://pdf.co) and sign up
2. Dashboard > API Key > copy the key
3. Free tier: 100 credits/month (sufficient for ~10-20 contracts)

#### Step 1.3 -- Set Up Google Drive Folder Structure

1. Open Google Drive with `procuralogic@gmail.com`
2. Create folder: `ProcuraLogic`
3. Inside it, create: `Client Contracts`
4. Inside it, create: `Templates`
5. Note the folder ID of `Client Contracts` from the URL
6. Inside `Templates`, upload a sample review document for reference

#### Step 1.4 -- Add Notion Properties

1. Open the Clients database in Notion
2. Add these properties (see Section 2 for details):
   - **AI Draft Review** (Rich Text)
   - **AI Analysis Status** (Select) -- add all 6 options
   - **AI Analyzed Date** (Date)
   - **Contract Text** (Rich Text)
   - **Delivered Date** (Date)
   - **Follow-Up Due** (Formula): `dateAdd(prop("Delivered Date"), 3, "days")`
3. Add a new Notion view: **AI Review Queue** (Board view grouped by AI Analysis Status)

### Phase 2: Store API Keys in Make.com (10 minutes)

#### Step 2.1 -- Create Make.com Variables

1. In Make.com, go to **Scenarios** (you will store these as custom variables within the scenario)
2. Alternatively, use a **Data Store** to hold configuration:
   - Go to **Data Stores** > **Create a Data Store**
   - Name: `ProcuraLogic Config`
   - Add fields: `anthropic_api_key` (text), `pdfco_api_key` (text), `drive_contracts_folder_id` (text)
   - Add one record with your actual keys

> **Security Note:** Make.com encrypts data store values. However, for maximum security, you can also use Make.com's built-in **Custom Variables** feature or the "Variables" module within each scenario.

### Phase 3: Build Scenario D -- AI Contract Analysis (45 minutes)

#### Step 3.1 -- Create the Scenario

1. Go to Make.com > **Scenarios** > **Create a new scenario**
2. Name: `ProcuraLogic AI Contract Analysis`
3. Add the first module: **Notion > Watch Database Items**

#### Step 3.2 -- Configure Module 1 (Notion Watch)

1. Connection: `My Notion Public connection`
2. Database: Select `Clients`
3. Filter: Set up a filter where `Status` equals `In Review`
4. Click **OK**
5. Set the scheduling to run every **15 minutes**

#### Step 3.3 -- Configure Module 2 (Notion Update -- Processing)

1. Add module: **Notion > Update a Database Item**
2. Connection: `My Notion Public connection`
3. Page ID: Map `{{1.id}}`
4. Set `AI Analysis Status` = `Processing`

#### Step 3.4 -- Configure Module 3 (Google Drive Download)

1. Add module: **Google Drive > Download a File**
2. Connection: `My Google connection` (ID 8160539)
3. Select from drive: **By ID**
4. File ID: Map from `{{1.Contract Upload}}` or extract from `{{1.Drive Folder}}`

> **Extracting File ID from a Drive URL:** If the Drive Folder field contains a URL like `https://drive.google.com/file/d/ABC123/view`, use Make.com's `replace()` function:
> ```
> {{replace(replace(1.`Drive Folder`; "/view"; ""); "https://drive.google.com/file/d/"; "")}}
> ```

#### Step 3.5 -- Configure Module 4 (PDF to Text)

**Using pdf.co:**

1. Add module: **HTTP > Make a Request**
2. URL: `https://api.pdf.co/v1/pdf/convert/to/text`
3. Method: `POST`
4. Headers: Add `x-api-key` with your pdf.co API key
5. Body Type: `application/json`
6. Body: `{"url": "{{3.downloadUrl}}"}`

> **Note:** If the file is downloaded as binary data (not a URL), you will need to upload it to a temporary location first or use the multipart form approach. Check what Module 3 outputs.

**Using Google Docs (free alternative):**

1. Add: **Google Drive > Upload a File**
   - Folder: Your temp folder ID
   - File: Map from Module 3 output
   - Convert to Google Docs: **Yes**
2. Add: **Google Docs > Get Content of a Document**
   - Document ID: `{{previous.id}}`
3. Add: **Google Drive > Delete a File**
   - File ID: `{{upload_module.id}}`

#### Step 3.6 -- Configure Module 5 (Claude API Call)

1. Add module: **HTTP > Make a Request**
2. URL: `https://api.anthropic.com/v1/messages`
3. Method: `POST`
4. Headers:
   - `x-api-key`: Your Anthropic API key
   - `anthropic-version`: `2023-06-01`
   - `content-type`: `application/json`
5. Body Type: `Raw`
6. Content Type: `JSON (application/json)`
7. Body: Paste the full JSON body from Section 5, Module 5, replacing the placeholder variables with Make.com mapped values from previous modules
8. Parse response: **Yes**

#### Step 3.7 -- Add Router for Success/Error

1. After Module 5, add a **Router**
2. **Route 1 (Success):** Filter condition: `{{5.statusCode}}` equals `200`
3. **Route 2 (Error):** Filter condition: `{{5.statusCode}}` does not equal `200`

#### Step 3.8 -- Configure Route 1 (Success)

1. Add: **Notion > Update a Database Item**
   - Page ID: `{{1.id}}`
   - AI Draft Review: `{{5.body.content[0].text}}`
   - AI Analysis Status: `Draft Ready`
   - AI Analyzed Date: `{{now}}`

2. Add: **Gmail > Send an Email**
   - Connection: My Gmail connection
   - To: `procuralogic@gmail.com`
   - Subject: `AI Draft Ready: {{1.Name}} - {{1.Contract Type}}`
   - Body: Use the HTML from Section 5, Module 7

#### Step 3.9 -- Configure Route 2 (Error)

1. Add: **Notion > Update a Database Item**
   - Page ID: `{{1.id}}`
   - AI Analysis Status: `Failed`
   - Notes: `AI analysis failed ({{5.statusCode}}): {{5.body.error.message}}. Manual review required.`

#### Step 3.10 -- Save and Test

1. Click **Save** (floppy disk icon)
2. Click **Run Once**
3. Create a test client in Notion with a sample contract uploaded
4. Set that client's status to "In Review"
5. Watch the scenario execute in Make.com
6. Verify the AI Draft Review field is populated in Notion
7. Verify you received the notification email

### Phase 4: Build Scenario E1 -- Delivery (30 minutes)

Follow the same pattern as Scenario D:

1. Create new scenario: `ProcuraLogic Delivery`
2. Module 1: Notion Watch (filter: Status = "Delivered")
3. Module 2: Notion Get Page (get full content)
4. Module 3: Gmail Send Email (to client with review)
5. Module 4: Notion Update (set Delivered Date = now)

### Phase 5: Build Scenario E2 -- Follow-Up (20 minutes)

1. Create new scenario: `ProcuraLogic Follow-Up`
2. Trigger: **Schedule** -- once daily at 9:00 AM
3. Module 1: **Notion > Search Objects** -- find records where Status = "Delivered" and Delivered Date is 3+ days ago
4. Module 2: **Iterator** over results
5. Module 3: **Notion > Update Database Item** -- set Status = "Follow-Up"
6. Module 4: **Brevo > Send an Email** -- Template #8 (follow-up)

### Phase 6: Plan Considerations

Your Make.com Core plan allows **3 active scenarios** by default. You currently have:

| # | Scenario | Status |
|---|---|---|
| 1 | Integration Notion (4614828) | Active |
| 2 | ProcuraLogic Calendly to Notion (4654104) | Active |
| 3 | ProcuraLogic Nurture Sequence (4654103) | Inactive (errors) |
| 4 | ProcuraLogic Inbound Pipeline (4654772) | Inactive (invalid) |

**To fit the AI scenarios, you have options:**

**Option A -- Consolidate (Recommended):** Merge Delivery (E1) and Follow-Up (E2) into the existing Integration Notion scenario (4614828), which already watches the Notion database and routes based on status. Add new routes for "Delivered" and "Follow-Up" triggers. This keeps you within 3 active scenarios while adding the AI Analysis scenario as #3 (replacing the broken Inbound Pipeline).

**Option B -- Upgrade:** The Core plan can be upgraded to allow more active scenarios. Check current pricing at make.com/pricing.

**Recommended Active Scenario Allocation:**

| Slot | Scenario | Purpose |
|---|---|---|
| 1 | Integration Notion (modified) | All status-based routing: emails, delivery, follow-up |
| 2 | ProcuraLogic Calendly to Notion | Calendly webhook handling |
| 3 | ProcuraLogic AI Contract Analysis (NEW) | AI pre-analysis pipeline |

The Inbound Pipeline and Nurture Sequence scenarios (both currently broken/inactive) should be rebuilt as routes within the Integration Notion scenario or fixed and swapped in when ready.

---

## 9. Testing Checklist

### Scenario D (AI Analysis) Tests

- [ ] Create a test client in Notion with all required fields filled
- [ ] Upload a sample contract PDF to Google Drive
- [ ] Link the Drive file in the client's Drive Folder or Contract Upload field
- [ ] Change client Status to "In Review"
- [ ] Verify AI Analysis Status changes to "Processing" within 15 minutes
- [ ] Verify AI Analysis Status changes to "Draft Ready"
- [ ] Verify AI Draft Review field contains a complete, well-structured analysis
- [ ] Verify notification email arrives at procuralogic@gmail.com
- [ ] Test with a Word document (.docx) to confirm PDF extraction handles it
- [ ] Test with a very long contract (20+ pages) to check token limits
- [ ] Test error handling: temporarily use a bad API key and verify "Failed" status is set
- [ ] Verify the Notion page link in the notification email works

### Scenario E1 (Delivery) Tests

- [ ] Edit the AI draft review in Notion (simulate owner editing)
- [ ] Change client Status to "Delivered"
- [ ] Verify client receives the review email
- [ ] Verify Delivered Date is set in Notion
- [ ] Check email formatting in multiple email clients (Gmail, Outlook, Apple Mail)

### Scenario E2 (Follow-Up) Tests

- [ ] Manually set a test client's Delivered Date to 4 days ago
- [ ] Run Scenario E2 manually
- [ ] Verify Status changes to "Follow-Up"
- [ ] Verify follow-up email is sent via Brevo
- [ ] Verify clients who were delivered less than 3 days ago are NOT affected

### End-to-End Test

- [ ] Create a new client through the full pipeline: intake > payment > contract upload > In Review > AI analysis > owner edit > Delivered > follow-up
- [ ] Measure total time from "In Review" to "AI Draft Ready"
- [ ] Review AI output quality on 3 different contract types (employment, vendor, freelance)

---

## 10. Troubleshooting

### Common Issues

**"AI Draft Review field is empty after successful run"**
- Check the Claude API response body in Make.com's execution log
- Verify you are mapping `{{5.body.content[0].text}}` correctly
- The response structure is: `body > content > array[0] > text`

**"Claude API returns 401 Unauthorized"**
- Verify your API key is correct in the HTTP module headers
- Check that the key has not been revoked at console.anthropic.com
- Ensure the header name is exactly `x-api-key` (lowercase)

**"Claude API returns 400 Bad Request"**
- Usually means the JSON body is malformed
- Check for unescaped quotes or newlines in the contract text
- Use Make.com's `replace()` function to sanitize: `{{replace(replace(4.text; char(34); char(92) + char(34)); newline; char(92) + "n")}}`

**"Contract text is too long for Claude"**
- Claude Sonnet 4 supports 200K tokens input. A contract would need to be 500+ pages to exceed this.
- If you hit `max_tokens` on the OUTPUT side (response truncated), increase `max_tokens` from 8000 to 12000.

**"Notion rich text field truncated"**
- Notion's API limits rich text properties to 2,000 characters per text block.
- **Solution:** Instead of writing to a property, append the review as page content using the Notion API:

```
POST https://api.notion.com/v1/blocks/{{page_id}}/children
Headers: Authorization: Bearer {{notion_token}}, Notion-Version: 2022-06-28
Body:
{
  "children": [
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": {
        "rich_text": [{"text": {"content": "AI Draft Review"}}]
      }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": {
        "rich_text": [{"text": {"content": "FIRST_2000_CHARS_HERE"}}]
      }
    }
  ]
}
```

Split the review into 2,000-character chunks, each as a separate paragraph block. Use Make.com's `substring()` function in a repeater to split the text.

**"PDF extraction returns garbled text"**
- Some scanned PDFs contain images, not text. pdf.co has OCR capabilities -- use the endpoint `https://api.pdf.co/v1/pdf/convert/to/text` with `{"inline": true, "ocr": true}` in the body.
- For consistently better results, ask clients to upload native PDFs (not scanned images) when possible.

### Appending Long Text to Notion (Detailed)

Since the AI review will typically be 3,000-6,000 words (well over the 2,000-character property limit), the recommended approach is:

1. After the Claude API call, use a **Make.com Text Parser > Split** module to break the response into 2,000-character chunks
2. Use a **Repeater** module to iterate over the chunks
3. For each chunk, use an **HTTP > Make a Request** module to call the Notion API's "Append Block Children" endpoint
4. The first iteration should include a heading block ("AI Draft Review")
5. Subsequent iterations append paragraph blocks

Alternatively, write the full review to a Google Doc and link that document in Notion's "Review Document" file property.

---

## Quick Reference: Connection IDs

| Connection | ID | App | Used In |
|---|---|---|---|
| My Gmail connection | 8184235 | Gmail | Scenarios A, D, E1 |
| My Notion Public connection | 8184144 | Notion | All scenarios |
| My Google connection | 8160539 | Google (Drive) | Scenario D |
| ProcuraLogic Brevo | 8229992 | Brevo | Scenarios B, E2 |
| *New: Anthropic API* | N/A (HTTP module) | HTTP | Scenario D |
| *New: pdf.co API* | N/A (HTTP module) | HTTP | Scenario D |

## Quick Reference: Notion Database Fields (After Additions)

**Existing Fields:** Name, Email, Phone, Business Name, Service Type, Status, Lead Source, Contract Type, Contract Upload, Drive Folder, Review Document, Engagement Value, Stripe Payment ID, Calendly Event, Session Date, Last Nurture Email, Nurture Start Date, Notes, Created Date

**New Fields:** AI Draft Review, AI Analysis Status, AI Analyzed Date, Contract Text, Delivered Date, Follow-Up Due (formula)
