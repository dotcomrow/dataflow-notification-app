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

The complete production flow is stored in Git as a NiFi Registry snapshot.
Argo bootstraps that exact snapshot into NiFi Registry and a `NifiDataflow`
resource pins version `1` with `syncMode: always`. NiFiKop continuously restores
the Git-managed version if the live process group drifts.

The NiFi GUI is intentionally read-only for the human OIDC group. Flow changes
must be made by updating the snapshot and version in Git; the
certificate-authenticated NiFi operator is the only flow writer.

Required Vault values before first production sync:

- `secret/data/k8s-kafka-nifi-registry-bucket-id#value`
- `secret/data/kafka-admin-username#value`
- `secret/data/kafka-admin-password#value`
- `secret/data/kafka-nifi-username#value`
- `secret/data/kafka-nifi-password#value`
- `secret/data/platform-notification-service#token` (injected into the NiFi
  runtime by the `k8s-kafka` manifests; it is never stored in the flow snapshot)

Default topics:

- request topic: `platform.notifications.requested.v1`
- DLQ topic: `platform.notifications.requested.dlq.v1`

Kafka access:

- `platform-notification-kafka-topics` creates the request and DLQ topics and
  grants the NiFi Kafka principal read/write/describe/create access to the
  `platform.notifications.` topic prefix, read/describe access to the consumer
  group, and cluster describe access.

Execution boundary:

- `platform-notification-service` owns request validation, enrichment, queue
  publication, and Directus audit writes.
- NiFi owns visible orchestration, retries, failure routing, and callbacks.
- Rules/template lookup should be implemented as explicit processors or service
  calls inside this flow.
- `platform-notification-executor` owns provider-specific delivery behavior.
