# Claude Code Prompt: Verida Property Correspondence and Reconciliation Workflow

## Context

**Rock Property has sold its rent roll to Verida Property.** From the handover date, property management correspondence, statements and payments come from **Verida Property** instead of Rock Property.

All relevant Verida Property correspondence is held in **Yahoo Mail**, in the folder **`B_11_VP`** and its subfolders.

A **Rock Property Statement Run** workflow already exists in Claude Code. This new work must be built as a **continuation of that workflow**, not as a separate, parallel system.

## Objective

Build a workflow, or a small set of workflows, that will:

1. **Download and file** Verida Property correspondence from Yahoo Mail on an **ongoing, incremental** basis.
2. **Interpret and reconcile** the four Verida Property document types:
   - **Receipt of Payment**
   - **Payment Receipt**
   - **Remittance Advice**
   - **Rental Income Statement**

## Step 0: Understand the existing Rock Property Statement Run first

Before writing any code, locate and read the existing Rock Property Statement Run workflow (CLAUDE.md files, skills, slash commands, scripts, folder structure, config, ledgers, output templates and logs). If you cannot find it, **stop and ask me where it lives**. Do not guess.

Then report back, briefly:

- How the Statement Run is triggered and what it produces.
- The folder structure and **file naming conventions** it uses.
- How it parses documents, what fields it extracts and where it stores results (CSV, Excel, JSON, database).
- How properties, owners, tenants and periods are identified and keyed.
- Any existing reconciliation logic, exception reporting or state tracking.
- What can be **reused directly**, what needs **extending**, and what is genuinely new.

**Reuse its conventions, structure, naming, output formats and code wherever possible.** Rock Property history and Verida Property history must sit side by side and be readable as one continuous record per property.

## Step 1: Mailbox discovery (read only)

- Connect to Yahoo Mail via **IMAP** (`imap.mail.yahoo.com`, port 993, SSL) using a **Yahoo app password**. Read credentials from environment variables or the existing workflow's secret store. **Never hard-code, print, log or commit credentials.**
- **Read only.** Do not delete, move, flag, mark as read or otherwise alter any email unless I explicitly ask.
- **Never send any email** on my behalf.
- Enumerate `B_11_VP` and **all of its subfolders**, with message counts and date ranges.
- Sample messages from each subfolder and identify: senders, subject patterns, attachment types, and which of the four document types each contains. Note any other document types found (for example bond lodgements, invoices, lease documents, inspection reports, handover letters) and list them for me.

Report findings before building anything further.

## Step 2: Workflow 1, download and file

Requirements:

- **Incremental and idempotent.** Track state per folder (IMAP `UIDVALIDITY` plus last processed `UID`), and de-duplicate by `Message-ID` and attachment content hash. Running it twice must never create duplicates.
- **Back-fill** all existing history on the first run, then only process new mail afterwards.
- Save each email (as `.eml` or the format the existing workflow uses) together with its attachments.
- **Classify** each document by type and file it by property, period and document type, following the existing Rock Property naming convention. Where there is no existing convention, propose one using `YYYY-MM-DD` in file names so they sort correctly.
- Maintain a **master index** (CSV or the existing ledger format) with: date received, sender, subject, subfolder, document type, property, period, file path, hash and processing status.
- Anything that cannot be classified confidently goes to an **"Unclassified / Review"** list, never silently skipped.
- Log every run: what was downloaded, filed, skipped and why.

## Step 3: Workflow 2, interpret and reconcile

### Extraction

For each document type, extract structured data. At minimum:

| Document | Fields to extract |
|---|---|
| **Rental Income Statement** | Statement number, statement period, owner, property, opening balance, rent received per tenant/period, other income, itemised expenses (management fee, letting fee, admin fee, repairs, council, water, strata, insurance, etc.), GST, total disbursed, closing/held balance |
| **Remittance Advice** | Remittance date, amount, payee, bank account (last digits only), reference, statement(s) it relates to |
| **Receipt of Payment** | Receipt number, date, payer, property, amount, period covered, payment method |
| **Payment Receipt** | Receipt number, date, payer/payee, property, amount, purpose, period covered |

First confirm from real samples what **Receipt of Payment** and **Payment Receipt** actually represent and how they differ (for example, tenant rent receipts vs payments made by Verida). **Do not assume.** Show me examples and your interpretation before coding the rules.

Validate every extraction: line items must sum to stated totals, opening balance plus income less expenses must equal closing balance. Flag any document that fails.

### Reconciliation checks

- **Receipts to Statement:** every rent receipt appears on the corresponding Rental Income Statement, for the correct amount and period.
- **Statement to Remittance:** the net amount on each Rental Income Statement matches the Remittance Advice.
- **Remittance to Bank:** if the existing Statement Run has access to bank data, match each remittance to the actual deposit (amount and date tolerance).
- **Rent continuity:** for each property and tenancy, rent is received for every period with **no gaps and no overlaps**, based on the lease rent amount and frequency.
- **Fees:** management and other fees are charged at the agreed rates, with GST calculated correctly. Flag any change in rates between Rock Property and Verida Property.
- **Balance carry-forward:** each statement's opening balance equals the previous statement's closing balance.

### Handover cut-over (critical)

Reconcile the **transition period** between the final Rock Property statement and the first Verida Property statement for every property:

- No rent **missed** or **double counted** across the changeover.
- Any funds held by Rock Property at handover (float, held balances, prepaid rent) are accounted for by Verida Property.
- Bond transfers are confirmed where evidence exists.
- Management agreement terms and fee rates are compared before and after the sale.

### Outputs

- A **reconciliation workbook** (Excel, matching the existing Statement Run format if there is one) per period, with a summary sheet and detail sheets per property.
- An **exceptions report** listing every mismatch, missing document, gap or failed validation, with the amount, the documents involved and a suggested next action.
- A short **plain-English summary** of each run: what arrived, what reconciled, what did not.
- Where the existing workflow feeds a financial year ledger or tax summary, extend it so Verida Property data flows in the same way.

## Integration and operation

- Add this as a continuation of the Statement Run: either new steps within it or a companion command (for example `/verida-run`) that follows the same structure, whichever fits better with what already exists. Explain your choice.
- Update the relevant **CLAUDE.md** and skill documentation so a future session can run it without this prompt.
- Propose how to run it on an ongoing basis (manual command, scheduled Routine, or both). **Do not set up any schedule until I approve it.**
- If this is a cloud session, check that outbound access to `imap.mail.yahoo.com` is allowed and that credentials are available as environment secrets. If not, tell me exactly what to configure.

## Conventions

- **Australian formats:** AUD, dates as `DD/MM/YYYY` in reports, financial year **1 July to 30 June**, GST at 10%.
- **Privacy:** tenant names, contact details and bank details are sensitive. Keep them out of logs and commit messages, mask bank account numbers, and make sure downloaded mail, attachments and outputs are **git-ignored** unless the existing workflow deliberately commits them.
- **No em dashes** in any output, report or document.
- Keep code simple, commented where the logic is not obvious, and consistent with the existing Statement Run code.

## Working method

1. Complete **Step 0 and Step 1** and report back before building.
2. Present a short **plan**: components, file layout, data model, and what is reused vs new.
3. Build Workflow 1, run the back-fill, and show me the index and any unclassified items.
4. Build Workflow 2, run it over all history, and show me the reconciliation workbook and exceptions report.
5. Ask me only about genuine business decisions (for example tolerance thresholds, how to treat a disputed fee). Make sensible, stated defaults for everything else.
6. Commit work in logical steps with clear commit messages.

## Definition of done

- One command downloads and files all new Verida Property correspondence from `B_11_VP` and its subfolders, safely and repeatably.
- All four document types are extracted, validated and reconciled, including the **Rock Property to Verida Property handover**.
- Reconciliation workbook, exceptions report and summary are produced in the same style as the existing Statement Run.
- Documentation is updated so the workflow can be run in any future session.
