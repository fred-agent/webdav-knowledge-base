# webdav-knowledge-base

A [Fred](https://github.com/ThalesGroup/fred) Knowledge Base that mirrors a folder shared
over WebDAV into a Fred library. It runs next to an existing Fred platform, and Fred shows
it to the teams a platform admin enables it for; each team then points its own instance
at a share.

This chart is Fred's generic
[`knowledge-base`](https://github.com/ThalesGroup/fred/tree/swift/deploy/charts/knowledge-base)
chart with this pod's image and defaults: every value sits under `knowledge-base:`.

## TL;DR

```bash
kubectl -n fred create secret generic knowledge-base-m2m \
  --from-literal=client-secret=...

helm install webdav-knowledge-base \
  oci://ghcr.io/fred-agent/webdav-knowledge-base-helm/webdav-knowledge-base \
  --version <version> -n fred
```

The install prints the entry to add to Fred's configuration (see
[Register and enable](#register-and-enable-in-fred)). Chart `X.Y.Z` runs image `vX.Y.Z`.

## What it deploys

| Resource | Role |
| --- | --- |
| `Deployment <release>` | the worker, `run`: polls Temporal and serves runs; opens no inbound port |
| `Job <release>-publish` | `publish`, a hook after each install, upgrade and rollback: declares the Knowledge Base to Fred, then exits |
| `ConfigMap <release>-config` | the pod's `configuration.yaml`, rendered from `knowledge-base.configuration` |
| `Service <release>-metrics` | the worker's metrics, ports 9000 (`fred_kb_*`) and 9001 (`temporal_*`) |

`<release>` is the release name, suffixed with `-knowledge-base` unless the name already
contains it.

## Prerequisites

What the Fred platform must provide; the defaults name the services of a default Fred
install (`keycloak`, `temporal:7233`, `control-plane-backend:8222`,
`knowledge-flow-backend:8000`).

- **Keycloak**, realm `app`: the pod's own confidential client `kb-fred.webdav`, with a
  service account holding the client role `app/service_agent`.
- **Temporal**: Fred dispatches runs to the pod through it.
- **The shares**: reachable from the pod over HTTPS (or HTTP). Each team gives its own
  address, and credentials if any, on its instance's form in Fred.

## Configure

### Credentials

The pod reads one secret, the client's, from the environment variable
`FRED_KB_CLIENT_SECRET`. By default it comes from the Secret `knowledge-base-m2m`, key
`client-secret`. To read it elsewhere, replace `knowledge-base.extraEnvVars`, or use
`knowledge-base.secret.existingSecret` (a Secret with the key `FRED_KB_CLIENT_SECRET`)
and set `knowledge-base.extraEnvVars: []`.

### Addresses

Override under `knowledge-base.configuration` whatever differs from a default Fred install:

| Value | Default |
| --- | --- |
| `knowledge_base.control_plane_url` | `http://control-plane-backend:8222/control-plane/v1` |
| `knowledge_base.knowledge_flow_url` | `http://knowledge-flow-backend:8000/knowledge-flow/v1` |
| `security.m2m.realm_url` | `http://keycloak/realms/app` |
| `scheduler.temporal.host` / `namespace` | `temporal:7233` / `default` |

`configuration` is the pod's configuration file: the chart's schema refuses a misspelled
key in Fred's sections at `helm install`, and the pod refuses one in `webdav:` at start-up.

### Optional

| Value | Default | Meaning |
| --- | --- | --- |
| `knowledge-base.configuration.webdav.concurrency` | `4` | documents one run fetches at once (1-64); times 10 MiB, a run's memory ceiling |
| `knowledge-base.configuration.webdav.ingestion_wait_seconds` | `600` | how long a run follows one document's ingestion before reporting it pending |
| `knowledge-base.configuration.app.runtime_id` | `webdav-kb` | the pod's identity in metrics and logs; give a second release its own |
| `knowledge-base.image` | `ghcr.io/fred-agent/webdav-knowledge-base`, tag `v<appVersion>` | another registry or tag |
| `knowledge-base.metricsService.serviceMonitor.enabled` | `false` | a ServiceMonitor for the Prometheus Operator |

`helm show values oci://ghcr.io/thalesgroup/fred-helm/knowledge-base` lists every other
value (resources, security contexts, scheduling).

### A private certificate authority

A share signed by a corporate root the image does not trust: mount the PEM and name it in
`WEBDAV_KB_CA_FILE`, which is added to the public authorities, not substituted for them.
A list replaces, so restate the client secret:

```yaml
knowledge-base:
  extraEnvVars:
    - { name: FRED_KB_CLIENT_SECRET, valueFrom: { secretKeyRef: { name: knowledge-base-m2m, key: client-secret } } }
    - { name: WEBDAV_KB_CA_FILE, value: /etc/ssl/private-ca/corporate-root.pem }
  extraVolumes:
    - { name: webdav-ca, configMap: { name: corporate-root-ca } }
  extraVolumeMounts:
    - { name: webdav-ca, mountPath: /etc/ssl/private-ca, readOnly: true }
```

## Register and enable in Fred

Fred accepts the declaration only from a client it lists with the prefix it owns.
`helm install` prints the entry (`helm get notes <release>` prints it again):

1. in the Fred chart's values, under `applications.control-plane-backend.configuration`:

   ```yaml
   knowledge_bases:
     publishers:
       - client_id: kb-fred.webdav
         prefix: fred.webdav
   ```

   then restart the Control Plane. If this release was installed first, its `publish`
   Job was refused: `helm upgrade` runs it again;
2. a platform admin opens **Admin > Features > Knowledge Bases** and enables it for the
   teams that should use it. Until then, no team does.

## Upgrade and uninstall

Each upgrade re-runs `publish`; a change to `configuration` restarts the worker.
`helm uninstall` removes the workloads; what the Knowledge Base ingested stays in Fred.

## Local development

[fred-deployment-factory](https://github.com/fred-agent/fred-deployment-factory) installs
this chart on a local k3d instance: `make k3d-app APP=webdav-kb` the published release,
`DIR=../webdav-knowledge-base` added to build the image from a checkout (its
`deploy/k3d/build`). Its `k3d-apps/webdav-kb/values.yaml` is an example of a real
instance's values.
