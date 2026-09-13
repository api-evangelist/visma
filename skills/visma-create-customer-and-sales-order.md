---
name: visma-create-customer-and-sales-order
description: Create a customer in Visma.net ERP and raise a sales order against it, then cancel the order if it was raised in error.
api: Visma.net ERP API
generated: '2026-09-13'
method: generated
source: openapi/visma-net-erp-service-api-openapi.json
operations:
  - Customer_GetAll
  - Customer_Post
  - Customer_GetBycustomerCd
  - SalesOrder_Post
  - SalesOrder_GetByorderNbr
  - SalesOrder_CancelSalesOrderBysaleOrderNumber
---

# Create a customer and raise a sales order (Visma.net ERP)

Base URL `https://api.finance.visma.net`. Every call needs an OAuth 2.0 bearer token from
`https://connect.visma.com/connect/token` and a tenant context header issued with the Developer
Portal application. Do not call `https://integration.visma.net` — it is retired and throttled to
500 calls/hour per application.

## 1. Check the customer does not already exist

`Customer_GetAll` — `GET /v1/customer`. Filter with `lastModifiedDateTime` plus
`lastModifiedDateTimeCondition`, and page with `pageNumber` / `pageSize`. There is no
idempotency key on this API, so this read is the only thing standing between you and a duplicate
customer record.

## 2. Create the customer

`Customer_Post` — `POST /v1/customer`. Set `erp-api-background: true` if you want the create to run
as a background job rather than blocking.

On failure, read the HTTP status: 400 for validation, 401 for a stale token, 403 for a scope the
application was not approved for, 412 only if you sent an `If-Match` header.

## 3. Read it back and keep the ETag

`Customer_GetBycustomerCd` — `GET /v1/customer/{customerCd}`. Keep the `ETag` response header (it
also appears in the body as `timeStamp`). Send it as `If-Match` on any later update so a concurrent
edit fails with 412 instead of silently overwriting someone else's change.

## 4. Raise the sales order

`SalesOrder_Post` — `POST /v1/salesorder`, with the `customerCd` from step 2. Confirm with
`SalesOrder_GetByorderNbr` — `GET /v1/salesorder/{orderNbr}`.

## 5. If it was wrong, cancel rather than delete

`SalesOrder_CancelSalesOrderBysaleOrderNumber` —
`POST /v1/salesorder/{saleOrderNumber}/action/cancelSalesOrder`. Visma documents no time window for
this; the practical limit is whether the order has been shipped or invoiced, and whether the
accounting period is still open.

## Rate limits

Read `X-RateLimit-Remaining` on every response. On 429, wait exactly `Retry-After` seconds. The
standard policy is per company + client per hour and Visma does not publish the number — treat the
headers as the only source of truth.
