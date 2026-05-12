# belo-helm-charts

Helm chart maestro (`pythonapps`) para desplegar aplicaciones Python FastAPI con ArgoRollouts y Tekton CI/CD.

## Estructura

```
belo-helm-charts/
├── charts-core/
│   └── pythonapps/                        ← chart maestro, replicable (javaapps, goapps…)
│       ├── Chart.yaml
│       ├── values.yaml                    ← defaults del chart
│       └── templates/
│           ├── _helpers.tpl
│           ├── rollout.yaml               ← ArgoRollout (bluegreen | canary | rollingupdate)
│           ├── service.yaml               ← Services stable + preview (siempre, ambos)
│           ├── ingress.yaml               ← Ingress stable + preview (siempre, ambos)
│           ├── hpa.yaml
│           ├── servicemonitor.yaml
│           └── pipeline-templates/
│               ├── tekton-sa.yaml         ← SA + RBAC (Triggers + Rollouts + ArgoCD reader)
│               ├── event-listener.yaml    ← EventListener + Ingress para webhook de GitHub
│               ├── trigger-binding.yaml   ← extrae repo-url, app-name, env, strategy, tag
│               ├── trigger-template.yaml  ← crea PipelineRun con params y PVCs efímeros
│               ├── task-clone.yaml
│               ├── task-build-kaniko.yaml
│               ├── task-bump-gitops.yaml  ← yq: image.tag + rollout.strategy en values; git push
│               ├── task-wait-argocd.yaml  ← polling ArgoCD Synced+Healthy + Rollout Paused|Healthy
│               ├── task-load-test.yaml    ← k6; emite result outcome=passed|failed; nunca falla el pipeline
│               ├── task-promote-rollback.yaml ← actúa sobre outcome; Canary hace 3 fases con k6 intermedio
│               └── pipeline-pythonapps.yaml   ← Pipeline de 6 stages
└── values-apps-files/
    ├── webserver-api01/
    │   ├── build-time/
    │   │   └── app-values.yaml            ← constantes de la app (sin rollout.strategy)
    │   ├── values-dev-webserver-api01.yaml
    │   ├── values-staging-webserver-api01.yaml
    │   ├── values-prod-webserver-api01.yaml
    │   ├── values-testing-webserver-api01.yaml
    │   └── values-lab-webserver-api01.yaml
    └── webserver-api02/
        ├── build-time/
        │   └── app-values.yaml
        ├── values-dev-webserver-api02.yaml
        ├── values-staging-webserver-api02.yaml
        ├── values-prod-webserver-api02.yaml
        ├── values-testing-webserver-api02.yaml
        └── values-lab-webserver-api02.yaml
```

## Cómo se dispara un deploy

La strategy **no es una propiedad fija de la app**: se determina en el tag Git y se
escribe en el values file del ambiente vía `bump-gitops`. El formato del tag es:

```
refs/tags/<env>/<strategy>/<semver>
```

Ejemplos:

```bash
# BlueGreen a prod
git tag prod/bluegreen/v1.4.0
git push origin prod/bluegreen/v1.4.0

# Canary a staging
git tag staging/canary/v1.4.0
git push origin staging/canary/v1.4.0

# RollingUpdate rápido a dev
git tag dev/rollingupdate/v1.4.0-rc1
git push origin dev/rollingupdate/v1.4.0-rc1
```

Tags con formato distinto (sin los tres segmentos) son ignorados por el EventListener.

## Flujo CI/CD (6 stages)

```
git tag <env>/<strategy>/<version>
        │
        ▼
GitHub webhook → EventListener CEL (filtra y extrae env/strategy/tag)
        │
        ▼
TriggerTemplate → PipelineRun en nodo cicd (taint workload=cicd)
        │
        ├─ Stage 1: clone          → git clone --depth 1 del repo de la app
        ├─ Stage 2: build-push     → Kaniko build + push a Docker Hub
        ├─ Stage 3: bump-gitops    → yq: image.tag + rollout.strategy en values file; git push
        ├─ Stage 4: wait-argocd    → polling: ArgoCD Synced+Healthy, Rollout Paused|Healthy
        ├─ Stage 5: load-test      → k6 contra preview/stable service; emite outcome (passed|failed)
        └─ Stage 6: promote-rollback → actúa sobre outcome:
                                        BlueGreen: promote o abort
                                        Canary:    promote faseado (25%→k6→50%→k6→100%) o abort
                                        Rolling:   no-op o undo
```

## Estrategias de rollout

La strategy se elige por tag. Los values de cada ambiente tienen un default que
`bump-gitops` sobreescribe en cada deploy:

| Ambiente  | Default          |
|-----------|------------------|
| dev       | `rollingupdate`  |
| staging   | `canary`         |
| prod      | `bluegreen`      |
| testing   | `rollingupdate`  |
| lab       | `rollingupdate`  |

### Topología de red (fija, independiente de la strategy)

Los Services e Ingresses `stable` y `preview` existen siempre para ambas apps.
Esto evita que cambiar de strategy entre deploys destruya y recree la infraestructura
de red. ArgoRollouts manipula los selectors; el chart no varía.

| Recurso               | Propósito                                              |
|-----------------------|--------------------------------------------------------|
| `<app>-stable` svc    | Tráfico activo (producción real)                       |
| `<app>-preview` svc   | Nueva versión antes de promotion (k6 apunta acá)       |
| `<app>-stable` ing    | Entry point externo del tráfico estable                |
| `<app>-preview` ing   | Entry point de acceso aislado para validación manual   |

### Comportamiento por strategy

**BlueGreen** (`autoPromotionEnabled: false`):
- Argo despliega la nueva versión en pods detrás del `preview` service
- El pipeline corre k6 contra `preview` (sin tráfico real)
- Si k6 pasa → `argo rollouts promote` (switch de tráfico a stable)
- Si k6 falla → `argo rollouts abort` (destruye green, stable intacto)

**Canary** (pasos 5% → 25% → 50% → 100%):
- Argo despliega canary pods y pausa en 5%
- Pipeline corre k6 contra `preview` (canary pods)
- Si pasa → promote a 25% → k6 → 50% → k6 → promote --full
- Cualquier fallo → `argo rollouts abort` (rollback inmediato)

**RollingUpdate** (`setWeight: 100`, sin pausa):
- Argo reemplaza todos los pods inmediatamente
- Pipeline corre k6 smoke contra `stable` (mix durante el rollout)
- Si pasa → deploy confirmado
- Si falla → `argo rollouts undo` (revierte al ReplicaSet anterior)

## Render manual

```bash
# api01 en dev (desde la raíz de belo-helm-charts/)
helm template webserver-api01 charts-core/pythonapps/ \
  -f values-apps-files/webserver-api01/build-time/app-values.yaml \
  -f values-apps-files/webserver-api01/values-dev-webserver-api01.yaml \
  -n dev

# api02 en staging
helm template webserver-api02 charts-core/pythonapps/ \
  -f values-apps-files/webserver-api02/build-time/app-values.yaml \
  -f values-apps-files/webserver-api02/values-staging-webserver-api02.yaml \
  -n staging
```

## build-time vs. runtime — por qué separados

- **build-time** (`build-time/app-values.yaml`): registry, nombre de imagen, repo_url y
  owner. Cambia raramente — solo cuando se migra el registry o se hace un fork.
  Es metadata de la app, no del ambiente. **`rollout.strategy` no va acá** —
  la strategy se determina en el tag Git y `bump-gitops` la escribe en el values
  del ambiente en cada deploy.
- **runtime por ambiente** (`values-<env>-<app>.yaml`): réplicas, recursos,
  `rollout.strategy`, hostname de Ingress, límites del HPA. `rollout.strategy`
  acá es el último valor escrito por el pipeline — refleja la strategy del deploy
  más reciente en ese ambiente.

## Secretos requeridos (namespace `tekton-pipelines`)

| Secret                 | Descripción                                                           |
|------------------------|-----------------------------------------------------------------------|
| `dockerhub-credentials`| `{ ".dockerconfigjson": <base64 docker config> }`                     |
| `gitops-github-token`  | `{ "token": <GitHub PAT con write access a belo-helm-charts> }`      |

> Si `gitops-github-token` no existe, bump-gitops intenta el push sin auth
> (funciona solo si el repo es público y tiene push sin credenciales, lo que no es el caso).

## RBAC del SA tekton-triggers-sa

El SA tiene tres ClusterRoles adicionales al Role base de Tekton Triggers:

- `tekton-rollouts-operator` — get/watch/update sobre Rollouts en todos los namespaces
- `tekton-argocd-reader` — get/watch sobre Applications en el namespace `argocd`
- `tekton-triggers-clusterrole` — interceptores cluster-scoped de Tekton Triggers
