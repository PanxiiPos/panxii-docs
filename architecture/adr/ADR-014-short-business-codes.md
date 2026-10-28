# ADR-014: Short product and invoice business codes

- Status: Accepted
- Date: 2026-10-28

## Context

Users confirmed that UUID-length product and invoice codes are too long and
selected a 12-character suffix. Codes must work offline across devices without
changing UUID identity, durable operation payloads, or historical invoices.

## Decision

- New automatic SKU: `MH-<12 characters>`; new invoice number:
  `INV-<12 characters>`. User-entered SKUs and category codes are unchanged.
- Hash lowercase UUID hex without hyphens as UTF-8 using SHA-256. Encode the
  first 60 bits, most significant bit first, using
  `0123456789ABCDEFGHJKMNPQRSTVWXYZ`. This avoids ambiguous I/L/O/U and does not
  expose UUID version bits or depend on byte ordering differences across SDKs.
- Shared vector: UUID `9d2a0f21-4ff3-4b37-8fe1-c8fa51d983c2` gives suffix
  `A5NZDMY7GYVM`. C# and Dart test identical vectors.
- Keep full UUIDs as primary keys, idempotency keys and offline references.
  Keep the editor client UUID stable across failed save attempts.
- Mobile generates sale numbers when creating new drafts. Backend derives
  the same short number from the invoice UUID, and uses the same algorithm
  for new reversal/correction invoices. Existing stored invoices are not renamed.
- Legacy drafts with `INV-<full UUID hex>` retain that submitted number when
  first synced; operation replay continues returning the already-stored invoice.
  Do not recompute or modify existing operation payload hashes.
- This is a new-write format change, not a schema/data migration. Deploy the
  updated API/worker before the updated mobile app so short draft numbers stay
  unchanged on sync. Existing clients can continue submitting legacy numbers.

## Alternatives considered

- A tenant-wide sequential counter needs online allocation or device ranges.
- Cutting UUID prefixes retains fewer random bits and leaks version structure.
- Full UUID encoding remains too long for the selected user experience.

## Consequences

The suffix space is 60 bits, not a mathematical uniqueness guarantee. Existing
tenant-scoped unique SKU/invoice indexes and explicit conflict errors remain
authoritative; do not silently overwrite, change a persisted code or retry with
a different operation payload. A sample collision test is a regression check,
not proof that collisions cannot happen.

## Related documents

- [Business rules](../../business_rule.md)
- [API contract](../../api/api_contract.md)
- [Mobile screen mapping](../../frontend/mobile_screen_flow_api_mapping.md)
