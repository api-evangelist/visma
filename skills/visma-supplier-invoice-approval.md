---
name: visma-supplier-invoice-approval
description: Register a supplier and its invoice in Visma.net ERP, move the invoice through release, and void or reverse it when needed.
api: Visma.net ERP API
generated: '2026-09-13'
method: generated
source: openapi/visma-net-erp-service-api-openapi.json
operations:
  - Supplier_GetAll
  - Supplier_Post
  - Supplier_GetBysupplierCd
  - SupplierInvoice_Post
  - SupplierInvoice_GetByinvoiceNumber
  - SupplierInvoice_GetByApprovalDocumentId
  - SupplierInvoice_ReleaseInvoiceByinvoiceNumber
  - SupplierInvoice_VoidInvoiceBydocumentTypeinvoiceNumber
  - SupplierInvoice_ReverseInvoiceBydocumentTypeinvoiceNumber
  - Supplier_GetSupplierBalanceBysupplierCd
---

# Supplier invoice intake and approval (Visma.net ERP)

Base URL `https://api.finance.visma.net`.

1. **Look the supplier up first.** `Supplier_GetAll` — `GET /v1/supplier`, paged with
   `pageNumber` / `pageSize`. No idempotency key exists, so the read is your duplicate guard.
2. **Create if absent.** `Supplier_Post` — `POST /v1/supplier`, then
   `Supplier_GetBysupplierCd` — `GET /v1/supplier/{supplierCd}` and keep the `ETag`.
3. **Register the invoice.** `SupplierInvoice_Post` — `POST /v1/supplierInvoice`. For a large batch,
   send `erp-api-background: true` and let Visma run it as a background job rather than holding the
   connection open.
4. **Track approval.** `SupplierInvoice_GetByApprovalDocumentId` — `GET /v1/supplierInvoice/approval`
   lists invoices sitting in the approval flow. `SupplierInvoice_GetByinvoiceNumber` reads one.
5. **Release when approved.** `SupplierInvoice_ReleaseInvoiceByinvoiceNumber` —
   `POST /v1/supplierInvoice/{invoiceNumber}/action/release`.
6. **Undo.** Void an unpaid released invoice with
   `SupplierInvoice_VoidInvoiceBydocumentTypeinvoiceNumber` —
   `POST /v1/supplierInvoice/{documentType}/{invoiceNumber}/action/voidinvoice`; reverse a posted one
   with `SupplierInvoice_ReverseInvoiceBydocumentTypeinvoiceNumber`. No window is published for
   either; a locked accounting period will refuse both.
7. **Reconcile.** `Supplier_GetSupplierBalanceBysupplierCd` —
   `GET /v1/supplier/{supplierCd}/balance`.

## Concurrency

Send the `ETag` you read back as `If-Match` on updates. A mismatch returns **412 Precondition
Failed**, which is Visma's only built-in protection against a lost update — 97 of the 511 operations
declare it.
