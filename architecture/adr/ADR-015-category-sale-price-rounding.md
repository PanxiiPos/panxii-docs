# ADR-015: Category-scoped sale-price rounding

- Status: Accepted
- Date: 2026-11-04

## Context

The owner approved nearest-thousand VND sale prices for the selected steel
category only, including future creates, edits and price adjustments. A remainder
of 500 rounds up; prices below 500 have a minimum sale price of 1000. Listed
prices, conversion quantities and historical invoices must not change.

## Decision

Persist an opt-in `categories.RoundSalePriceToThousands` boolean, default false.
Identify the policy by the tenant-owned category ID, never by its mutable name,
SKU or brand. For opted-in categories calculate directly from the exact product:

`salePrice = max(1000, floor((listedPrice * calculationQuantity + 500) / 1000) * 1000)`

All other categories retain the existing four-decimal, midpoint-away-from-zero
calculation. Server create/update/bulk paths are authoritative; mobile preview,
saved commands and offline cache use matching fixed-point arithmetic. The server
accepts either the old exact-product submission or the new rounded submission
for an opted-in category, then stores the policy result; arbitrary mismatches
remain invalid. Category updates that omit the new flag preserve its value.

Enabling the flag does not silently rewrite existing catalog rows. Reprice the
selected existing rows through audited, versioned catalog commands with their
unchanged listed prices and conversion quantities. Do not recalculate invoice
snapshots or apply this rule to invoice totals/discounts.

## Alternatives considered

- Display-only rounding: rejected because the charged/stored price must match.
- Global rounding or matching a translated category name: rejected because
  accessories and other tenants must retain their current behavior.
- Replacing listed prices or weights: rejected because it loses source inputs.

## Consequences

Backend migration precedes mobile rollout. Old category payloads default to
disabled on mobile; refresh category caches after enabling the rule. A rollback
to older pricing code must not accept catalog writes while this flag is enabled.
Tests cover 19400/19500 boundaries, very small values, overflow, unchanged
accessories, queued legacy submissions and default-variant/product consistency.

Invoice sync still validates submitted unit prices against the current catalog.
Changing catalog prices can therefore reject an older offline sale; it does not
recalculate an already-synced invoice. Before enabling/repricing or moving a
family between policies, sync pending sales and resolve drafts on every selling
device. Server queue drainage alone does not establish that device-side work is
clear. Deploying the additive column with all flags false is separate from
authorizing a live repricing operation.

## Related documents

- [Business rules](../../business_rule.md)
- [API contract](../../api/api_contract.md)
- [Database design](../../database/database_design.md)
