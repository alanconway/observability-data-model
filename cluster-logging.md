# Cluster Logging and OpenTelemetry

This is the protocol and semantic conventions documentation for Red Hat OpenShift Logging's OTEL support starting with Logging v6.1 which is considered **Tech-Preview**. This document should be considered as a work in progress and is subject to change until OTEL support graduates to **General Acceptance**.

| Specification                                                        | Version |
|----------------------------------------------------------------------|---------|
| [OpenTelemetry](https://opentelemetry.io/docs/specs/otel/)           | 1.48.0  |
| [Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/) | 1.36.0  |

## Forwarding Protocol

Red Hat OpenShift Logging provides a log collection and forwarding solution that is capable of writing logs to OpenTelemetry endpoints using OTLP. [OTLP](https://opentelemetry.io/docs/specs/otlp/) is the *protocol* for encoding, transporting, and delivering telemetry data.  This document defines the semantic conventions associated with the logs collected from the various sources of an OpenShift cluster.

**Note:** Logs are forwarded using OTLP/HTTP as defined by the OpenTelemetry Observability Framework.  It uses Protobuf payloads encoded in JSON format.

## LokiStack Storage

The LokiStack store has an [OTLP endpoint](https://grafana.com/docs/loki/latest/send-data/otel/) to ingest OTLP log streams. OTEL logs are stored in Loki as follows:

* OTEL attribute names are converted to Loki labels. Characters: (`.`,`/`,`-`) are replaced by underscore (`_`). For example, `k8s.namespace.name` becomes `k8s_namespace_name`.
* Selected resource attributes become stream labels, other attributes become structured-metadata labels (details below.)
* The `Body` field of the OTEL log record is stored as the Loki log record.
* Other log record fields become structured-metadata labels (details below.)

## Log Record Structure

The [log data model](https://opentelemetry.io/docs/specs/otel/logs/data-model/#log-and-event-record-definition) defines fields in a log record.

| OTEL Field Name     | Loki Label             | Comment                                                                     |
|:--------------------|:-----------------------|:----------------------------------------------------------------------------|
| `Body`              |                        | Stored as the Loki log record.                                              |
| `Timestamp`         | `timestamp`            | UnixNano format                                                             |
| `ObservedTimestamp` | `observed_timestamp`   | UnixNano format                                                             |
| `SeverityText`      | `severity_text`        | Only on container and journal logs                                          |
| `Resource`          | see attributes section | Describe the _source_ of the logs, same values for all records in a stream. |
| `Attributes`        | see attributes section | Describe individual logs, can have different values for each record.        |

**Note:** Unlike _attribute_ names, field names are not mandated by the spec, they are intended to map to "native" names in preexisting formats, protocols or storage.
For example, the OTLP _protocol_ represents `Timestamp` as `timeUnixNano` to fit JSON-RPC naming conventions.
The Loki label names above are defined by the [Loki OTEL mapping](https://grafana.com/docs/loki/latest/send-data/otel/)

## Attributes

Log entries will have a set of resource, scope and log attributes depending on their source described by the following table.

The "Location" column can be one of the following:

* `resource` for a resource attribute
* `scope` for a scope attribute
* `log` for a log attribute

The "Storage" column shows whether the attribute is stored into a LokiStack using the default `openshift-logging` tenancy mode and where the attribute is stored:

* `stream label` (with an optional "required", if the Loki Operator will enforce this attribute in the configuration)
* `structured metadata`

| Name | Location | Applicable Sources | Storage (LokiStack) | Comment |
| :--- | :------- | :----------------- | :------------------ | :------ |
| `log_source` | resource | all | required stream label | **(DEPRECATED)** Compatibility attribute, contains same information as `openshift.log.source` |
| `log_type` | resource | all | required stream label | **(DEPRECATED)** Compatibility attribute, contains same information as `openshift.log.type` |
| `kubernetes.container_name` | resource | container | stream label | **(DEPRECATED)** Compatibility attribute, contains same information as `k8s.container.name` |
| `kubernetes.host` | resource | all | stream label | **(DEPRECATED)** Compatibility attribute, same information as `k8s.node.name` |
| `kubernetes.namespace_name` | resource | container | required stream label | **(DEPRECATED)** Compatibility attribute, contains same information as `k8s.namespace.name` |
| `kubernetes.pod_name` | resource | container | stream label | **(DEPRECATED)** Compatibility attribute, contains same information as `k8s.pod.name` |
| `openshift.cluster_id` | resource | all | | **(DEPRECATED)** Compatibility attribute, contains same information as `openshift.cluster.uid` |
| `level` | log | container, journal | | **(DEPRECATED)** Compatibility attribute, contains same information as `severityText` |
| `openshift.cluster.uid` | resource | all | required stream label | |
| `openshift.log.source` | resource | all | required stream label | |
| `openshift.log.type` | resource | all | required stream label | |
| `openshift.label.*` | resource | all | structured metadata | |
| `k8s.node.name` | resource | all | stream label | |
| `k8s.namespace.name` | resource | container | required stream label | |
| `k8s.container.name` | resource | container | stream label | |
| `k8s.pod.label.*` | resource | container | structured metadata | |
| `k8s.pod.name` | resource | container | stream label | |
| `k8s.pod.uid` | resource | container | structured metadata | |
| `k8s.cronjob.name` | resource | container | stream label | Conditionally forwarded based on creator of Pod |
| `k8s.daemonset.name` | resource | container | stream label | Conditionally forwarded based on creator of Pod |
| `k8s.deployment.name` | resource | container | stream label | Conditionally forwarded based on creator of Pod |
| `k8s.job.name` | resource | container | stream label | Conditionally forwarded based on creator of Pod |
| `k8s.replicaset.name` | resource | container | structured metadata | Conditionally forwarded based on creator of Pod |
| `k8s.statefulset.name` | resource | container | stream label | Conditionally forwarded based on creator of Pod |
| `log.iostream` | log | container | structured metadata | |
| `process.executable.name` | resource | journal | structured metadata | |
| `process.executable.path` | resource | journal | structured metadata | |
| `process.command_line` | resource | journal | structured metadata | |
| `process.pid` | resource | journal | structured metadata | |
| `service.name` | resource | journal | stream label | |
| `systemd.t.*` | log | journal | structured metadata | |
| `systemd.u.*` | log | journal | structured metadata | |

**Note:** Attributes marked as "Compatibility attribute" are added to support minimal backwards compatibility with the [ViaQ](https://github.com/openshift/cluster-logging-operator/blob/release-6.0/docs/reference/datamodels/viaq/v1.adoc) data model. These attributes should be considered deprecated and will be removed one release after **General Acceptance** of Red Hat OpenShift Logging.

**Note:** Attributes starting with `openshift.` are openshift logging extensions, not (yet) part of the OTEL spec.

### Attributes vs. Structured Logs

Information in the log body is normally _not_ duplicated as attributes.
This is different from the ViaQ model, where structured logs (especially audit logs) were parsed and presented as fields in the ViaQ envelope.

Loki allows you to parse and query structured logs on their fields, so there is no need to extract these fields in the collector.

Examples:

Query for API audit events (JSON body) with a particular event level and user name.

    {openshift_log_type="audit"}|json|k8s_audit_event_level=="MetaData"|k8s_user_name=="Fred"

Query for Linux audit events (logfmt body) for services that were started by the root user.

    {openshift_log_type="audit"}|logfmt|uid=0|~"^SERVICE_START"

## References

* [Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
* [Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
* [General Logs Attributes](https://opentelemetry.io/docs/specs/semconv/general/logs/)
* [Cluster Logging OTEL Support](https://github.com/openshift/enhancements/pull/1684)
* [Ingesting logs to Loki using OpenTelemetry Collector](https://grafana.com/docs/loki/latest/send-data/otel/)
* [Kubernetes Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/resource/k8s/)
