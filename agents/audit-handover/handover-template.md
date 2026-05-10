# Auditor handover — {{entity_name}}, {{period}}

**Prepared by:** Ntropii Audit Handover (Claude MA)
**Period:** {{period}}
**Entity:** {{entity_name}} ({{entity_slug}})
**Currency:** {{currency}}
**Prepared:** {{prepared_at_iso}}

## 1. Executive summary

One short paragraph. State the period, the NAV / TB outcome at a high level, total movements, and anything unusual that the auditor should look at first.

## 2. Trial balance summary

| Field | Value |
|---|---|
| Opening balance | {{tb.opening_total}} |
| Closing balance | {{tb.closing_total}} |
| Σ debits | {{tb.total_debits}} |
| Σ credits | {{tb.total_credits}} |
| Account count | {{tb.account_count}} |

## 3. Journal posting log

One row per posted journal line.

| Account | Description | Debit | Credit | Source |
|---|---|---|---|---|
| {{account_code}} | {{description}} | {{debit}} | {{credit}} | {{source_doc}} |

(Repeat for each line in `period.journal_proposal.lines`.)

## 4. Variances vs prior period

One short paragraph. Reference the largest variances by account or category. If no prior baseline is available (first period for this entity), say so explicitly.

## 5. Reconciliations status

| Check | Status | Detail |
|---|---|---|
| Journal balanced | {{check.journal_balanced.status}} | {{check.journal_balanced.detail}} |
| Closing TB balanced | {{check.closing_tb_balanced.status}} | {{check.closing_tb_balanced.detail}} |
| Bank reconciles | {{check.bank_reconciles.status}} | {{check.bank_reconciles.detail}} |
| Bank vs rent-roll | {{check.bank_vs_rentroll_reconciles.status}} | {{check.bank_vs_rentroll_reconciles.detail}} |

If a check did not run for this period, write "Not run" in its row.

## 6. Exceptions / open items

Bullet list of items that need follow-up:
- Unresolved arrears (rent-roll vs bank gap)
- HITL warnings overridden by the preparer
- Documents missing or rejected at extraction
- Any quality check that returned `warn` or `fail`

If none, write: "None flagged."

## 7. Supporting documents

| Filename | Source | Document ref |
|---|---|---|
| {{filename}} | {{source}} | {{document_ref}} |

(Repeat for each entry in `period.documents`.)

## 8. Sign-off

| Role | Name | Date |
|---|---|---|
| Preparer | __________________________ | ____________ |
| Reviewer | __________________________ | ____________ |
