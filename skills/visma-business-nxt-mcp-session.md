---
name: visma-business-nxt-mcp-session
description: Open a Business NXT MCP session, load the tenant schema, query safely, and make a guarded write.
api: Visma Business NXT MCP
generated: '2026-09-13'
method: generated
source: https://docs.vismasoftware.no/businessnxtapi/ai/mcp/
operations:
  - businessnxt-init
  - businessnxt-init_company
  - businessnxt-get_table_definitions
  - businessnxt-get_field_values
  - businessnxt-execute_graphql_query
  - businessnxt-create_entity
  - businessnxt-update_entity
---

# Working a Business NXT MCP session

Endpoint `https://mcp.business.visma.net/`, streamable HTTP, stateless. Every request carries a
bearer token from Visma Connect; the token is issued to a *person*, so the server can never exceed
what that user could do in Business NXT directly.

## Order of operations — this sequence is mandatory

1. `businessnxt-init` — returns the companies the user can reach, each with `companyNo` and
   `tenantId`. Nothing else works before this.
2. `businessnxt-init_company` with that pair — loads the tenant's table schemas (including
   data-model-extension tables), the GraphQL query rules, org-unit dimension labels, account-number
   series, voucher series and open accounting periods. Queries, skills and writes all depend on it.
3. `businessnxt-get_table_definitions` for the tables you are about to touch. Columns carry
   `readOnly`, `primaryKey`, `canInsert` and `canUpdate` flags and `joinup` / `joindown` relations.
4. `businessnxt-get_field_values` for **every** column flagged with an `enumKind`. Coded integer
   fields such as `order.orderType` and `order.orderStatus1` are meaningless without it.

## Reading

`businessnxt-execute_graphql_query` runs any read-only GraphQL query the signed-in user is entitled
to. Mutations are rejected at the AST level, so do not try to sneak a write through here. Results
default to compact toon encoding — pass `response_format="json"` when exact values matter.

## Writing

Writes go through `businessnxt-create_entity` and `businessnxt-update_entity` only, and only against
17 named tables: `order`, `orderLine`, `associate`, `appointment`, `product`, `productCategory`,
`priceAndDiscountMatrix`, `deliveryAlternative`, `structure`, `text`, `batch`, `voucher`, `budget`,
`budgetLine`, `capitalAsset`, `crossReference` (create only) and `stockBalance` (update only).
Everything else is read-only through the MCP regardless of the user's Business NXT permissions.

- **Rehearse first.** `businessnxt-create_entity` supports `dryRun`, which validates and returns
  server-computed values — including currency context on `*InCurrency` amount columns — without
  persisting. Use it.
- **Updates need the old row.** `businessnxt-update_entity` requires the primary keys *and*
  `oldValues`. A stale snapshot blocks the write rather than clobbering a concurrent change.
- **Errors fail the batch.** Both default to `FAIL_TABLE`, so one bad field aborts the whole batch
  instead of quietly substituting a default. Leave it that way.

## The permission you will hit first

Even with a valid token, the GraphQL API rejects MCP requests unless the company has opted in via
the **AI permission** setting, which carries separate Read and Write toggles (Write requires Read)
and must allow the operation in *both* the System information table and the Company table. Only a
System supervisor can change it. If every call fails with an access error while ordinary API calls
succeed, this is why.
