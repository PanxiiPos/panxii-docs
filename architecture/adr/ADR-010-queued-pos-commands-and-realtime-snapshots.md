# ADR-010: Queued POS commands and realtime entity snapshots

- Status: Accepted
- Date: 2026-10-14

## Context

The previous sync contract returned after the API wrote an invoice and emitted a
SignalR notification that only instructed clients to pull. The POS workflow now
requires create commands to be processed through RabbitMQ and connected tenant
clients to show the committed entity at the top of their list without refreshing.

## Decision

- `POST /products`, `POST /customers`, and `POST /invoices/sync` publish a
  persistent, versioned command with publisher confirms and return `202 Queued`.
- Tenant, actor, device, and correlation identifiers are taken from the
  authenticated server context; clients cannot select another tenant or actor.
- The worker processes commands at least once and deduplicates by
  `(TenantId, OperationId, PayloadHash)`.
- In one MySQL transaction, the worker stores the entity, completed operation,
  durable `ServerChange` sequence, and realtime outbox event.
- The outbox publisher sends the full committed entity snapshot and sequence to
  the `panxii.realtime` topic exchange only after the transaction commits.
- Each API instance owns an ephemeral RabbitMQ subscription and broadcasts the
  event to `tenant:{TenantId}`. SignalR group membership is derived from the
  validated JWT. Rejections are sent only to the initiating user.
- Clients insert a new snapshot at the top of the relevant list and deduplicate
  by `serverSequence`/`messageId`. `GET /sync/pull?afterSequence=` recovers
  changes missed while disconnected.

## Alternatives considered

- Keep synchronous HTTP writes and use RabbitMQ only for notifications. This
  does not satisfy queued processing or isolate API latency from writes.
- Broadcast only an entity ID and require every connected client to fetch it.
  This adds a request per change and does not satisfy the no-refresh workflow.
- Use a process-local SignalR broadcast. This misses clients connected to other
  API instances.

## Consequences

- The three create endpoints return `202` before database persistence; clients
  must wait for `DataCreated` or `CommandRejected` to resolve the operation.
- This changes the previous `201` response semantics and requires a compatible
  mobile release before rollout.
- RabbitMQ command and retry queues are durable; poison commands go to DLQ.
- SignalR is an optimization for connected clients. MySQL `ServerChange` is the
  durable recovery path; messages may be delivered more than once.
- The backend repository has no Flutter source, so mobile subscription/list
  insertion must be implemented and tested in the mobile repository.

## Related documents

- [data_sync_flow_rabbitmq.md](../../sync/data_sync_flow_rabbitmq.md)
- [infrastructure_setup.md](../../infrastructure/infrastructure_setup.md)
- [api_contract.md](../../api/api_contract.md)
- [openapi.yaml](../../api/openapi.yaml)
- [api_compatibility_matrix.md](../../api/api_compatibility_matrix.md)