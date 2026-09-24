# Claude Code Prompt: Verida Property Correspondence and Reconciliation Workflow

## Context

**Rock Property has sold its rent roll to Verida Property.** From the handover date, property management correspondence, statements and payments come from **Verida Property** instead of Rock Property.

All relevant Verida Property correspondence is held in **Yahoo Mail**, in the folder **`B_11_VP`** and its subfolders.

A **Rock Property Statement Run** workflow already exists in Claude Code. Build this new work as a **continuation of that workflow**, not as a separate, parallel system.

## Objective

Build a workflow, or a small set of workflows, that will:

1. **Download and file** Verida Property correspondence from Yahoo Mail on an **ongoing, incremental** basis.
2. **Interpret and reconcile** the four Verida Property document types:
   - **Receipt of Payment**
   - **Payment Receipt**
   - **Remittance Advice**
   - **Rental Income Statement**

## Step 0: Understand the existing Rock Property Statement Run

Before writing any code, find and read the existing Rock Property Statement Run workflow: its CLAUDE.md files, skills, slash commands, scripts, folder structure, config, ledgers, output templates and logs. If you cannot find it, **stop and ask me where it lives**. Do not guess.

Then report back briefly on:

- How the Statement Run is triggered and what it produces.
- Its folder structure and **file naming conventions**.
- How it parses documents, which fields it extracts, and where it stores results (CSV, Excel, JSON or a database).
- How it identifies and keys properties, owners, tenants and periods.
- Any existing reconciliation logic, exception reporting or state tracking.
- What can be **reused directly**, what needs **extending**, and what is genuinely new.

**Reuse its conventions, structure, naming, output formats and code wherever possible.** Rock Property history and Verida Property history must sit side by side and read as one continuous record for each property.

## Step 1: Mailbox discovery (read only)

- If the existing workflow already connects to Yahoo Mail, **reuse that connection**. Otherwise, connect via **IMAP** (`imap.mail.yahoo.com`, port 993, SSL) using a **Yahoo app password**. Read credentials from environment variables or the existing workflow's secret store. **Never hard-code, print, log or commit credentials.**
- **Read only.** Do not delete, move, flag or mark as read any email, or change it in any other way, unless I explicitly ask. Fetch messages with `BODY.PEEK` so they are not marked as read.
- **Never send any email** on my behalf.
- List `B_11_VP` and **all of its subfolders**, with message counts and date ranges.
- Sample messages from each subfolder and identify senders, subject patterns, attachment types, and which of the four document types each message contains. Note any other document types you find (for example bond lodgements, invoices, lease documents, inspection reports or handover letters) and list them for me.

Report your findings before building anything further.

## Step 2: Download and file (Workflow 1)

Requirements:

- **Incremental and idempotent.** Track state per folder (IMAP `UIDVALIDITY` plus the last processed `UID`), and remove duplicates by `Message-ID` and attachment content hash. Running it twice must never create duplicates.
- **Back-fill** all existing history on the first run, then process only new mail after that.
- Save each email (as `.eml`, or the format the existing workflow uses) together with its attachments.
- **Classify** each document by type, and file it by property, period and document type, following the existing Rock Property naming convention. Where no convention exists, propose one that puts `YYYY-MM-DD` in file names so they sort correctly.
- Maintain a **master index** (CSV or the existing ledger format) with: date received, sender, subject, subfolder, document type, property, period, file path, hash and processing status.
- Anything you cannot classify confidently goes on an **"Unclassified / Review"** list. Never skip anything silently.
- Log every run: what was downloaded, filed and skipped, and why.

## Step 3: Interpret and reconcile (Workflow 2)

### Extraction

Extract structured data from each document type. At minimum:

| Document | Fields to extract |
|---|---|
| **Rental Income Statement** | Statement number, statement period, owner, property, opening balance, rent received per tenant and period, other income, itemised expenses (management fee, letting fee, admin fee, repairs, council rates, water, strata, insurance, etc.), GST, total disbursed, closing or held balance |
| **Remittance Advice** | Remittance date, amount, payee, bank account (last digits only), reference, the statement(s) it relates to |
| **Receipt of Payment** | Receipt number, date, payer, property, amount, period covered, payment method |
| **Payment Receipt** | Receipt number, date, payer or payee, property, amount, purpose, period covered |

First, use real samples to confirm what **Receipt of Payment** and **Payment Receipt** actually represent and how they differ (for example, tenant rent receipts compared with payments Verida made). **Do not assume.** Show me examples and your interpretation before you code the rules.

Validate every extraction: line items must add up to the stated totals, and the opening balance plus income, less expenses and payments to the owner, must equal the closing balance. Flag any document that fails.

### Reconciliation checks

- **Receipts to Statement:** every rent receipt appears on the matching Rental Income Statement, for the correct amount and period.
- **Statement to Remittance:** the net amount on each Rental Income Statement matches the Remittance Advice.
- **Remittance to Bank:** if the existing Statement Run has access to bank data, match each remittance to the actual deposit, allowing a tolerance on amount and date.
- **Rent continuity:** for each property and tenancy, rent is received for every period, with **no gaps and no overlaps**, based on the lease's rent amount and payment frequency, where lease details are available.
- **Fees:** management and other fees are charged at the agreed rates, with GST calculated correctly. Flag any rate change between Rock Property and Verida Property.
- **Balance carry-forward:** each statement's opening balance equals the previous statement's closing balance.

### Handover cut-over (critical)

For every property, reconcile the **transition period** between the final Rock Property statement and the first Verida Property statement:

- No rent is **missed** or **double counted** across the changeover.
- Verida Property accounts for any funds Rock Property held at handover (float, held balances, prepaid rent).
- Bond transfers are confirmed where there is evidence.
- Management agreement terms and fee rates are compared before and after the sale.

### Outputs

- A **reconciliation workbook** for each period (Excel, in the existing Statement Run format if there is one), with a summary sheet and a detail sheet for each property.
- An **exceptions report** listing every mismatch, missing document, gap and failed validation, with the amount, the documents involved and a suggested next action.
- A short **plain-English summary** of each run: what arrived, what reconciled and what did not.
- If the existing workflow feeds a financial year ledger or tax summary, extend it so Verida Property data flows in the same way.

## Integration and operation

- Add this work as a continuation of the Statement Run. That can be new steps inside it, or a companion command (for example `/verida-run`) that follows the same structure, whichever fits better with what already exists. Explain your choice.
- Update the relevant **CLAUDE.md** and skill documentation so a future session can run the workflow without this prompt.
- Propose how to run it on an ongoing basis (a manual command, a scheduled Routine, or both). **Do not set up any schedule until I approve it.**
- If this is a cloud session, check that outbound access to `imap.mail.yahoo.com` is allowed and that credentials are available as environment secrets. If they are not, tell me exactly what to configure.

## Conventions

- **Australian formats:** AUD, dates as `DD/MM/YYYY` in reports, a financial year of **1 July to 30 June**, and GST at 10%.
- **Privacy:** tenant names, contact details and bank details are sensitive. Keep them out of logs and commit messages, and mask bank account numbers. Make sure downloaded mail, attachments and outputs are **git-ignored**, unless the existing workflow commits them on purpose.
- **No em dashes** in any output, report or document.
- Keep the code simple, add comments where the logic is not obvious, and stay consistent with the existing Statement Run code.

## Working method

1. Complete **Step 0 and Step 1**, then report back before building.
2. Present a short **plan**: components, file layout, data model, and what is reused and what is new.
3. Build Workflow 1, run the back-fill, and show me the index and any unclassified items.
4. Build Workflow 2, run it over all history, and show me the reconciliation workbook and exceptions report.
5. Ask me only about genuine business decisions (for example tolerance thresholds, or how to treat a disputed fee). For everything else, choose sensible defaults and state them.
6. Commit work in logical steps with clear commit messages.

## Definition of done

- One command downloads and files all new Verida Property correspondence from `B_11_VP` and its subfolders, safely and repeatably.
- All four document types are extracted, validated and reconciled, including the **handover from Rock Property to Verida Property**.
- The reconciliation workbook, exceptions report and summary are produced in the same style as the existing Statement Run.
- The documentation is updated so the workflow can be run in any future session.
