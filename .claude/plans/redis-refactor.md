# Redis refactor: replace Bitnami Redis with Valkey

**Status:** Proposed · **Target chart version:** `3.0.0` (breaking) · **Target Chatwoot:** `v4.18.0` · **Researched:** 2026-10-02

## 1. Summary

Replace the Bitnami `redis` subchart (`16.12.2`, Redis `6.2.7`, image `bitnamilegacy/redis`) with the
**official Valkey Helm chart** (`valkey-io/valkey-helm`, chart `valkey` `0.12.0`, Valkey `9.1.2`,
image `docker.io/valkey/valkey`).

- **Valkey** is the Linux Foundation fork of Redis 7.2.4. It is BSD-3-Clause licensed and backed by AWS,
  Google Cloud, Oracle, Ericsson and Snap. It uses the same protocol, commands and clients as Redis.
- Both the chart and the image are published by the Valkey project. The chart is BSD-3-Clause and the
  image is built from `valkey-io/valkey-container`. Neither comes from a single vendor or an individual.
- The dependency is **aliased as `redis`**. Chatwoot keeps reading `redis.enabled` and `redis.host` and
  the `REDIS_*` env vars. Resource names stay `<release>-chatwoot-redis`, and external-Redis users see
  no change.
- **Built-in Redis Sentinel is removed and will not come back in this chart.** If you need Sentinel,
  bring your own Redis/Valkey and point Chatwoot at it (section 6). External Sentinel keeps working
  exactly as before.

## 2. Why the change is needed

- Since 2025-08-28 Bitnami no longer publishes free versioned images. Every existing tag was moved to
  `docker.io/bitnamilegacy`, which **gets no further updates or security patches**. Maintained images
  are only available through the paid *Bitnami Secure Images* product.
- This chart already uses that workaround (`redis.image.repository: bitnamilegacy/redis`). As a result it
  ships **Redis 6.2.7 from 2022**, frozen and unpatched.
- Redis itself has changed licence twice. In 2024 it moved to RSALv2/SSPLv1. In 2025, Redis 8 added
  AGPLv3 as a third option. Valkey has stayed on the permissive BSD-3 licence and is governed by a
  foundation, not a company.

## 3. Options considered

| Option | Image | Governance / licence | Verdict |
|---|---|---|---|
| **Official Valkey chart** (`valkey-io/valkey-helm`) | `valkey/valkey` (Valkey project) | Linux Foundation, BSD-3 (chart and server) | ✅ **Chosen.** First-party and actively maintained (releases in Jul, Aug and Sep 2026). Has a JSON schema, ACL auth, existing-secret support, PDBs, metrics, TLS and replication. GitLab is evaluating it as their own Bitnami Redis replacement. |
| Bitnami Valkey chart | Bitnami images | Broadcom | ❌ Still Bitnami, which is the thing we are moving away from. |
| `dandydeveloper/redis-ha` | `redis` (Docker Official Image) | Community, single maintainer; Redis is AGPL/RSAL/SSPL | ⚠️ Mature HA (Sentinel + HAProxy), but it ties us to Redis licensing and a single maintainer. |
| CloudPirates / groundhog2k / helmforge charts | various | Small vendors or individuals | ❌ These are the "random" charts we want to avoid. |
| Operators (`valkey-operator`, OT-Container-Kit `redis-operator`) | various | varies | ❌ They need CRDs and cluster-scoped installs, which is too heavy for an app chart's bundled cache. |
| Our own StatefulSet in `templates/` using `valkey/valkey` | `valkey/valkey` | us | ⚠️ Viable fallback with no chart dependency. We would own all HA and upgrade logic. |

**Chatwoot compatibility (checked against `v4.18.0` source):**

- Chatwoot uses `redis` 5.0.6, `redis-client` 0.26.4, `sidekiq` 7.3.10 and `actioncable` 7.2.
- Sidekiq officially supports Valkey 7.2+ and treats Redis 7.2.4 compatibility as canonical, which is
  exactly what Valkey forked from.
- Chatwoot uses no Redis Stack modules.
- Every consumer passes `REDIS_PASSWORD` explicitly: `lib/redis/config.rb` (Sidekiq, Alfred, Velma,
  Rack::Attack) and `config/cable.yml`. A password-only `AUTH` maps to Valkey's ACL `default` user.

## 4. Target design

### 4.1 Dependency

```yaml
# charts/chatwoot/Chart.yaml
dependencies:
  - condition: redis.enabled
    name: valkey
    alias: redis                     # keep .Values.redis.* and resource names stable
    repository: https://valkey.io/valkey-helm/
    version: 0.12.0                  # Valkey 9.1.2; pin exactly, bump deliberately
```

I rendered this aliased setup locally with Helm 3.18 and the values from 4.2. The Valkey schema has no
`additionalProperties: false`, so it ignores Chatwoot-only keys (`host`, `port`, `password`,
`existingSecret`). The rendered resources were:

| Kind | Name |
|---|---|
| Deployment, Service, PVC, ServiceAccount | `<rel>-chatwoot-redis` |
| Secret (key `default-password`) | `<rel>-chatwoot-redis-auth` |
| ConfigMaps | `<rel>-chatwoot-redis-config`, `<rel>-chatwoot-redis-init-scripts` |

The image is `docker.io/valkey/valkey:9.1.2`.

### 4.2 Default topology: standalone

The old default was Bitnami `architecture: replication` with one replica. **Chatwoot never reads from
replicas**, so that replica only cost resources. The new default is a standalone instance (a Deployment
with one PVC):

```yaml
# charts/chatwoot/values.yaml
redis:
  enabled: true
  nameOverride: chatwoot-redis
  # Image defaults to docker.io/valkey/valkey:<chart appVersion>. Override with redis.image.tag.
  auth:
    enabled: true
    aclUsers:
      default:
        permissions: "~* &* +@all"
        password: redis
        # passwordKey: default        # key inside usersExistingSecret (defaults to the username)
    # When set, passwords are read from this secret instead of aclUsers.*.password
    # usersExistingSecret: secret-name
  dataStorage:
    enabled: true
    requestedSize: 8Gi               # same size as the old Bitnami master PVC
    keepPvc: true                    # Bitnami StatefulSet PVCs survived uninstall; keep that behaviour
    # className: ""
    # persistentVolumeClaimName: ""  # use an existing PVC instead of creating one
  deploymentStrategy: Recreate       # RWO PVC + RollingUpdate deadlocks when the new pod lands on another node
  valkeyConfig: |
    appendonly yes
  replica:
    enabled: false
    # Built-in Sentinel is not supported. For Sentinel, set redis.enabled=false and use
    # env.REDIS_SENTINELS / env.REDIS_SENTINEL_MASTER_NAME against your own Redis.
  # ---- Only used when redis.enabled=false (external Redis/Valkey) ----
  # host: redis
  # port: 6379
  # password: redis
  # existingSecret: secret-name
  # existingSecretKey: password
```

> **Persistence parity:** Bitnami set `appendonly yes` and `save ""`. Valkey's default is RDB snapshots
> only. Sidekiq keeps its queues, scheduled jobs and retries in Redis, so we keep AOF on with
> `valkeyConfig`.

> **Security context change:** Bitnami ran as UID/GID `1001`. The Valkey chart runs as `1000` with
> `readOnlyRootFilesystem: true`, drops all capabilities and uses `seccompProfile: RuntimeDefault`. This
> is stricter than before.

## 5. File-by-file changes

### 5.1 `charts/chatwoot/Chart.yaml`

- Swap the `redis` dependency for the aliased `valkey` one in 4.1.
- Bump `version: 2.0.27` → `3.0.0`.
- Keep the `postgresql` and `common` entries (out of scope, see section 9).

### 5.2 `charts/chatwoot/Chart.lock` and `charts/chatwoot/charts/`

- Run `helm repo add valkey https://valkey.io/valkey-helm/ && helm dependency update charts/chatwoot`.
- `git rm charts/chatwoot/charts/redis-16.12.2.tgz` and commit `charts/chatwoot/charts/valkey-0.12.0.tgz`.
  The `.tgz` files are tracked in this repo.

### 5.3 `charts/chatwoot/values.yaml`

- Replace the `redis:` block with the one in 4.2.
- Remove `image.repository: bitnamilegacy/redis`, `auth.password`, the commented `sentinel.image`
  (Bitnami), `master.persistence` and `replica.replicaCount`.

### 5.4 `charts/chatwoot/templates/_helpers.tpl`

Every Redis helper gets new paths. Use `dig` to read subchart keys so a missing map never causes a
nil-pointer render error.

```gotemplate
{{/* Must match the subchart's valkey.fullname. The alias makes the subchart's .Chart.Name "redis". */}}
{{- define "chatwoot.redis.fullname" -}}
{{- if .Values.redis.fullnameOverride -}}
{{- .Values.redis.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default "redis" .Values.redis.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}
```

The old helper hard-coded `"chatwoot-redis"` in `printf` while checking `nameOverride`, so its output only
matched the subchart's naming when `nameOverride` was left at its default. The version above fixes that.

| Helper | Old behaviour | New behaviour |
|---|---|---|
| `chatwoot.redis.host` | `<fullname>-master`, or `<fullname>-headless` with Sentinel | `<fullname>` (the Valkey Service). External: `redis.host`, unchanged. |
| `chatwoot.redis.port` | hard-coded `6379` | `dig "service" "port" 6379 .Values.redis`. External: unchanged. |
| `chatwoot.redis.password` | `redis.auth.password` | `dig "auth" "aclUsers" "default" "password" "" .Values.redis`. External: unchanged. |
| `chatwoot.redis.url` | embeds `.Values.redis.auth.password` | embeds `include "chatwoot.redis.password" .`. Nothing else changes. |
| `chatwoot.redis.secret` / `.secretKey` | defined but **never used** (they reference Bitnami's `redis-password` key) | **Delete.** |
| `chatwoot.redis.sentinels` / `.sentinelMasterName` | built from Bitnami `redis.sentinel.*` and `redis.replica.replicaCount` | **Delete.** Built-in Sentinel is not supported (section 6). |
| **new** `chatwoot.redis.existingSecret` | — | Internal: `tpl redis.auth.usersExistingSecret`. External: `redis.existingSecret`. |
| **new** `chatwoot.redis.existingSecretKey` | — | Internal: `redis.auth.aclUsers.default.passwordKey`, default `"default"`. External: `redis.existingSecretKey`, default `"password"`. |
| **new** `chatwoot.redis.validate` | — | Fails the render, naming the replacement key, when old Bitnami keys are present (see below). |

```gotemplate
{{- define "chatwoot.redis.validate" -}}
{{- if .Values.redis.enabled -}}
{{- $r := .Values.redis -}}
{{- $hint := "Chart 3.0.0 replaced Bitnami Redis with Valkey." -}}
{{- if dig "auth" "password" "" $r }}{{ fail (printf "redis.auth.password was removed; use redis.auth.aclUsers.default.password. %s" $hint) }}{{ end -}}
{{- if dig "auth" "existingSecret" "" $r }}{{ fail (printf "redis.auth.existingSecret was removed; use redis.auth.usersExistingSecret (key 'default'). %s" $hint) }}{{ end -}}
{{- if hasKey $r "architecture" }}{{ fail (printf "redis.architecture was removed; use redis.replica.enabled. %s" $hint) }}{{ end -}}
{{- if hasKey $r "master" }}{{ fail (printf "redis.master.* was removed; use redis.dataStorage.*. %s" $hint) }}{{ end -}}
{{- $byo := "Built-in Sentinel is not supported. Set redis.enabled=false and use env.REDIS_SENTINELS / env.REDIS_SENTINEL_MASTER_NAME with your own Redis." -}}
{{- if hasKey $r "sentinel" }}{{ fail (printf "redis.sentinel.* was removed. %s %s" $byo $hint) }}{{ end -}}
{{- if dig "replica" "sentinel" "enabled" false $r }}{{ fail (printf "redis.replica.sentinel.enabled is not supported. %s" $byo) }}{{ end -}}
{{- end -}}
{{- end -}}
```

### 5.5 `charts/chatwoot/templates/env-secret.yaml`

- Add `{{- include "chatwoot.redis.validate" . }}` at the top.
- Change `{{- if not .Values.redis.existingSecret }}` to
  `{{- if not (include "chatwoot.redis.existingSecret" .) }}` around `REDIS_PASSWORD`.
- Remove the built-in Sentinel block (lines 24–27, which emit `REDIS_SENTINELS` and
  `REDIS_SENTINEL_MASTER_NAME` from the subchart).
- Simplify the `env` loop (line 29) back to a plain `range` with no Sentinel filter. `env.REDIS_SENTINELS`
  and `env.REDIS_SENTINEL_MASTER_NAME` then pass straight through, which is all an external Sentinel
  needs.

### 5.6 `templates/web-deployment.yaml`, `templates/worker-deployment.yaml`, `templates/migrations-job.yaml`

This also fixes an **existing bug**. Web and worker read `redis.existingSecret` and
`redis.existingSecretKey`, but the migration job reads `redis.auth.existingSecret` and
`redis.auth.existingSecretPasswordKey`. As a result, at most one of the two ever received the password
from the secret. All three templates should use the same block:

```gotemplate
{{- with include "chatwoot.redis.existingSecret" . }}
- name: REDIS_PASSWORD
  valueFrom:
    secretKeyRef:
      name: {{ . }}
      key: {{ include "chatwoot.redis.existingSecretKey" $ }}
{{- end }}
```

The `init-redis` initContainer in `migrations-job.yaml` needs no change, because it resolves
`chatwoot.redis.host`.

### 5.7 `charts/chatwoot/values.ci.yaml`

Replace the `redis:` block (`bitnamilegacy/redis`, `architecture: standalone`) with nothing, so the
defaults apply. Alternatively, set `redis.dataStorage.enabled: false` to speed up ephemeral CI clusters.

### 5.8 `charts/chatwoot/values.sentinel-test.yaml`

**Delete it.** It only exercises the built-in Sentinel, which no longer exists. CI does not reference
this file.

### 5.9 `.github/workflows/lint-test.yaml` and `.github/workflows/release.yaml`

Add `helm repo add valkey "https://valkey.io/valkey-helm/"` next to the Bitnami line. Keep
`helm repo add bitnami` while `postgresql` still comes from Bitnami.

### 5.10 `charts/chatwoot/README.md`

- **Redis variables:** document `redis.auth.aclUsers.default.password`, `redis.auth.usersExistingSecret`,
  `redis.dataStorage.*`, `redis.image.tag` and `redis.valkeyConfig`, and link to the Valkey chart README
  for everything else.
- **Redis Sentinel (when using the built-in Redis):** delete the section. In **External Redis**, add a
  line saying that Sentinel is supported only with your own Redis, using `env.REDIS_SENTINELS`,
  `env.REDIS_SENTINEL_MASTER_NAME` and optionally `REDIS_SENTINEL_PASSWORD` (see section 6).
- **Other Parameters:** replace `redis.master.persistence.enabled` with `redis.dataStorage.enabled`.
- **Redis section:** state that the bundled server is Valkey and is wire-compatible with Redis.
## 6. Sentinel: not supported in this chart

**Decision:** this chart does not deploy Redis Sentinel, now or later. The bundled Valkey is a simple
single instance for small and medium installs. Anyone who needs high availability runs their own
Redis/Valkey (a managed service such as ElastiCache, Memorystore or Azure Cache, or their own Sentinel
deployment) and points Chatwoot at it.

**Why:**

- Keeping HA out of the bundled cache keeps this chart small and avoids more divergence from upstream.
- The Valkey chart has not released Sentinel yet. Support (valkey-helm PR #234) is only on `main`, and
  `0.12.0` does not include it.
- Chatwoot `v4.18.0` could not log in to the Valkey chart's Sentinels anyway. The chart disables the
  `default` user on Sentinel and only exposes a `sentinel` ACL user, while Chatwoot sends a password with
  no username.

**What the chart does:**

- Drops every built-in Sentinel helper, template branch and test values file (sections 5.4, 5.5 and
  5.8).
- Fails the render if `redis.sentinel.*` or `redis.replica.sentinel.enabled` is set, with a message
  pointing at the bring-your-own path.
- Leaves `redis.replica.enabled` available for anyone who wants a warm read replica. It gives no
  automatic failover.

**Bring your own Redis with Sentinel:**

```yaml
redis:
  enabled: false
  password: <redis password>          # or existingSecret / existingSecretKey
env:
  REDIS_SENTINELS: "sentinel-0.example:26379,sentinel-1.example:26379,sentinel-2.example:26379"
  REDIS_SENTINEL_MASTER_NAME: mymaster
  # REDIS_SENTINEL_PASSWORD: ...      # only if the Sentinels use a different password from Redis
                                      # (or supply it through existingEnvSecret)
```

**Requirement on your Sentinels:** Chatwoot `v4.18.0` authenticates to Sentinel with a **password only,
no username**. Your Sentinels must therefore accept password-only `AUTH`, using `requirepass` or a
password on the `default` user, or have no auth at all. Sentinels that only allow a named ACL user will
not work, and the Valkey chart's own Sentinel mode is one of these.

## 7. Test plan

1. Run `helm dependency build charts/chatwoot && helm lint charts/chatwoot` and
   `ct lint --target-branch main`.
2. Run `helm template` across this matrix and check the rendered `<rel>-env` Secret and `REDIS_PASSWORD`
   env:
   - default values
   - `values.ci.yaml`
   - internal Redis with `usersExistingSecret`
   - external Redis with `host`/`port`/`password`
   - external Redis with `existingSecret`
   - external Redis with `env.REDIS_TLS=true`
   - external Redis with `env.REDIS_SENTINELS`
3. Negative render tests: each old Bitnami key checked by `chatwoot.redis.validate` (section 5.4) must
   fail with its message. So must `redis.replica.sentinel.enabled: true`. Check that `env.REDIS_SENTINELS` and
   `env.REDIS_SENTINEL_MASTER_NAME` now reach the `<rel>-env` Secret unchanged in both internal and
   external mode.
4. Install on kind with defaults and check:
   - the migrate job completes (`init-redis` resolves the host)
   - `/api` is ready
   - a new conversation triggers Sidekiq jobs that are processed
   - ActionCable pushes live updates to a second browser
   - the Valkey pod restarts with data intact (AOF)
5. Optional: re-enable the commented-out `ct install` steps in `lint-test.yaml` now that no images depend
   on the frozen `bitnamilegacy` catalog for Redis.

## 8. Risks and open questions

| Risk | Mitigation |
|---|---|
| **Fork divergence.** Upstream `chatwoot/charts` still uses Bitnami, so every upstream merge will conflict on `Chart.yaml` (version line, deps), `Chart.lock`, `values.yaml` (`redis:` block) and README. | Keep the change contained to the files listed above. Consider proposing it upstream; GitLab and others are making the same move. |
| `valkey` chart is pre-1.0 (`0.x`), so minor bumps can break, e.g. `0.5.0` renamed `common.image`. | Pin an exact version and read the release notes before each bump. |
| Chart version `3.0.0` jumps ahead of upstream's `2.0.x` numbering. | Required by semver for a breaking change. On each upstream merge, keep ours on the `version:` line. |
| Valkey 9 is three major versions ahead of the Redis 6.2 currently deployed. | Chatwoot and Sidekiq use only core commands. Covered by test plan step 4. If needed, pin `redis.image.tag: 8.1.10` (latest 8.1.x) instead. |
| The default topology drops the unused replica pod. | Intentional; Chatwoot never reads from replicas. Users who want a warm standby can set `redis.replica.enabled: true`. |

## 9. Out of scope (Bitnami still present after this change)

This change removes Bitnami **Redis** only. The chart will still depend on Bitnami in two places:

- **`postgresql` 11.6.7** from `charts.bitnami.com`. The image is already `ghcr.io/chatwoot/pgvector`,
  but the chart is Bitnami's. This needs its own plan. Candidates are the CloudNativePG operator (a
  CNCF project) or an in-tree StatefulSet with the pgvector image.
- **`common` 2.31.4** from `oci://registry-1.docker.io/bitnamicharts`. It is only used for
  `common.tplvalues.render` on `nodeSelector` in the web, worker and migrate templates. A three-line
  local helper can replace it so the dependency can be dropped. This is a quick follow-up.

## Sources

- Bitnami catalog changes: [bitnami/charts#35164](https://github.com/bitnami/charts/issues/35164), [Northflank summary](https://northflank.com/blog/bitnami-deprecates-free-images-migration-steps-and-alternatives)
- Valkey Helm chart: [valkey-io/valkey-helm](https://github.com/valkey-io/valkey-helm), [releases](https://github.com/valkey-io/valkey-helm/releases), [Sentinel PR #234](https://github.com/valkey-io/valkey-helm/pull/234), [announcement](https://valkey.io/blog/valkey-helm-chart/)
- Valkey image: [hub.docker.com/r/valkey/valkey](https://hub.docker.com/r/valkey/valkey), [valkey-io/valkey-container](https://github.com/valkey-io/valkey-container)
- Redis licensing: [InfoQ: Redis 8 AGPL](https://www.infoq.com/news/2025/05/redis-agpl-license), [Phoronix](https://www.phoronix.com/news/Redis-8.0-Goes-AGPLv3)
- Others replacing Bitnami Redis: [GitLab chart #6227](https://gitlab.com/gitlab-org/charts/gitlab/-/issues/6227), [VSHN ADR 0038](https://kb.vshn.ch/app-catalog/adr/0038-appcat-redis-alternative.html), [Passbolt](https://www.passbolt.com/blog/bitnami-legacy-changes-passbolts-migration-plan-for-open-source-secure-helm-deployments)
- Sidekiq datastore support: [Introducing Sidekiq 8.0](https://mikeperham.com/2025/03/05/introducing-sidekiq-8.0)
- Chatwoot `v4.18.0`: [`lib/redis/config.rb`](https://github.com/chatwoot/chatwoot/blob/v4.18.0/lib/redis/config.rb), [`config/cable.yml`](https://github.com/chatwoot/chatwoot/blob/v4.18.0/config/cable.yml), [`Gemfile.lock`](https://github.com/chatwoot/chatwoot/blob/v4.18.0/Gemfile.lock)
