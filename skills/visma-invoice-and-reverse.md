---
name: visma-invoice-and-reverse
description: Issue, release and print a customer invoice in Visma.net ERP, and reverse it correctly when it has to be undone.
api: Visma.net ERP API
generated: '2026-09-13'
method: generated
source: openapi/visma-net-erp-service-api-openapi.json
operations:
  - CustomerInvoice_Create
  - CustomerInvoice_GetByinvoiceNumber
  - CustomerInvoice_ReleaseInvoiceByinvoiceNumber
  - CustomerInvoice_PrintInvoiceByrefNbr
  - CustomerInvoice_SendToAutoInvoiceByinvoiceNumber
  - CustomerInvoice_CorrectInvoiceByinvoiceNumber
  - CustomerInvoice_ReverseInvoiceByinvoiceNumber
  - CustomerInvoice_ReverseInvoiceAndApplyToNoteByinvoiceNumber
  - CustomerInvoice_DeleteByinvoiceNumber
---

# Issue and reverse a customer invoice (Visma.net ERP)

Base URL `https://api.finance.visma.net`. Bearer token from Visma Connect.

## 1. Create the invoice

`CustomerInvoice_Create` — `POST /v1/customerinvoice`. At this point the document is unreleased and
still fully reversible by deletion.

## 2. Release it

`CustomerInvoice_ReleaseInvoiceByinvoiceNumber` —
`POST /v1/customerinvoice/{invoiceNumber}/action/release`. **This is the irreversible step.** Once
released, the invoice has hit the ledger and `CustomerInvoice_DeleteByinvoiceNumber` is no longer the
right instrument.

## 3. Deliver it

- `CustomerInvoice_PrintInvoiceByrefNbr` — `GET /v1/customerinvoice/{refNbr}/print`
- `CustomerInvoice_SendToAutoInvoiceByinvoiceNumber` —
  `POST /v1/customerinvoice/{invoiceNumber}/action/sendToAutoInvoice` for electronic delivery.

## Undoing it — pick the right verb

| Situation | Operation |
| --- | --- |
| Not yet released | `CustomerInvoice_DeleteByinvoiceNumber` — `DELETE /v1/customerinvoice/{invoiceNumber}` |
| Released, needs amending | `CustomerInvoice_CorrectInvoiceByinvoiceNumber` — `POST /v1/customerinvoice/{invoiceNumber}/action/correct` |
| Released, must be undone | `CustomerInvoice_ReverseInvoiceByinvoiceNumber` — `POST /v1/customerinvoice/{invoiceNumber}/action/reverse` |
| Released, undo and settle against a note | `CustomerInvoice_ReverseInvoiceAndApplyToNoteByinvoiceNumber` — `POST /v1/customerinvoice/{invoiceNumber}/action/reverseandapplytonote` |

Visma publishes no deadline for any of these. The real constraint is the accounting period: once the
period or fiscal year is locked, the reversal is refused. In the eAccounting API the equivalent
refusals are error codes 4026 `VoucherDateIsInLockedPeriod` and 4038
`VoucherDateIsInLockedYearException`; the ERP API returns a 4xx without a documented code.

## Retry safety

There is no `Idempotency-Key` on this API. If a `POST /v1/customerinvoice` times out, do **not**
blindly retry — call `CustomerInvoice_GetAll` filtered on `createdDateTime` first and check whether
the invoice landed.
