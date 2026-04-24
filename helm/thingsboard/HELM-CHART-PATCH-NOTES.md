# Helm chart patch notes

This document explains local chart changes used for a managed Kubernetes
microservices deployment (ThingsBoard CE 4.3.1.1).

## Why these patches were needed

The upstream chart needed small compatibility changes for this deployment model:

- External managed PostgreSQL is used instead of the bundled `postgresql-ha` chart.
- Kafka runs in KRaft mode without Zookeeper.
- Redis is provided by external Valkey (`externalRedis.host`) in this setup.
- Old pinned Bitnami chart versions were no longer available in the public repo.

Without these changes, `helm dependency build`, `helm template`, or runtime startup
would fail in this environment.

## Patched files and behavior

1. `helm/thingsboard/Chart.yaml`
- Updated dependency versions to currently available Bitnami chart versions.
- Added `condition` fields so optional dependencies can be disabled cleanly:
  - `postgresql-ha.enabled`
  - `kafka.enabled`
  - `redis.enabled`

2. `helm/thingsboard/templates/node-db-configmap.yaml`
- Added `externalPostgresql` branch to set:
  - `SPRING_DATASOURCE_URL`
  - `SPRING_DATASOURCE_USERNAME`
  - `SPRING_DATASOURCE_PASSWORD`
- Keeps existing in-cluster `postgresql-ha` behavior as fallback.

3. `helm/thingsboard/templates/initializedb-job.yaml`
- Wrapped postgres readiness `initContainer` in:
  - `if not .Values.externalPostgresql`
- Prevents template/runtime issues when the in-cluster PostgreSQL chart is disabled.

4. `helm/thingsboard/templates/node.yaml`
- Changed `ZOOKEEPER_ENABLED` from hardcoded `"true"` to configurable value:
  - `{{ .Values.zookeeperEnabled | default "false" }}`
- Made `REDIS_HOST` configurable via external host:
  - `{{ .Values.externalRedis.host | default (printf "%s-redis-master" .Release.Name) }}`

## Risk profile

These patches are low risk and mainly increase configurability.
They do not change the core ThingsBoard application logic.
