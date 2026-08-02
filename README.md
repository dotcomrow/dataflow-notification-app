# dataflow-notification-app

Production NiFi flow app for platform notification orchestration.

Flow ownership:

1. Consume notification requests from `platform.notifications.requested.v1`.
2. Extract notification metadata.
3. Mark the request as `processing` through `platform-notification-service`.
4. Resolve rules, templates, preferences, and channel fanout.
5. Call `platform-notification-executor` for each concrete delivery attempt.
6. Record delivery attempts through `platform-notification-service`.
7. Mark the request as `sent`, `partially_sent`, or `failed`.
8. Route orchestration failures to `platform.notifications.requested.dlq.v1`.

The first implementation creates the process group, Kafka connection, core
callbacks, and a dry-run executor call. Rules/template resolution processors are
present as named placeholders so they can be replaced with concrete service calls
without changing the intake or delivery contract.

Required Vault values before first production sync:

- `secret/data/k8s-kafka-nifi-registry-bucket-id#value`
- `secret/data/kafka-admin-username#value`
- `secret/data/kafka-admin-password#value`
- `secret/data/kafka-nifi-username#value`
- `secret/data/kafka-nifi-password#value`
- `secret/data/platform-notification-service#token`

Default topics:

- request topic: `platform.notifications.requested.v1`
- DLQ topic: `platform.notifications.requested.dlq.v1`

Execution boundary:

- `platform-notification-service` owns request validation, enrichment, queue
  publication, and Directus audit writes.
- NiFi owns visible orchestration, retries, failure routing, and callbacks.
- Rules/template lookup should be implemented as explicit processors or service
  calls inside this flow.
- `platform-notification-executor` owns provider-specific delivery behavior.
