# belo-helm-charts

Helm chart maestro (`pythonapps`) para desplegar aplicaciones Python FastAPI con ArgoRollouts y Tekton CI/CD.

## Estructura

```
pythonapps/
├── Chart.yaml
├── values.yaml                        # defaults del chart
├── templates/
│   ├── _helpers.tpl
│   ├── rollout.yaml                   # ArgoRollout (bluegreen | canary | rollingupdate)
│   ├── service.yaml                   # Services stable + preview
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── servicemonitor.yaml
│   └── pipeline-templates/
│       ├── tekton-sa.yaml             # ServiceAccount + RBAC para Tekton Triggers
│       ├── task-clone.yaml            # Task: git clone
│       ├── task-build-kaniko.yaml     # Task: build + push con Kaniko
│       ├── task-load-test.yaml        # Task: k6 load test (skip si no hay scripts)
│       ├── task-bump-gitops.yaml      # Task: actualiza image.tag en gitops-files
│       ├── pipeline-pythonapps.yaml   # Pipeline principal
│       ├── trigger-binding.yaml       # TriggerBinding (payload GitHub → params)
│       ├── trigger-template.yaml      # TriggerTemplate (crea PipelineRuns)
│       └── event-listener.yaml        # EventListener (webhook GitHub en tag push)
└── apps/
    ├── webserver-api01/
    │   ├── build-time/app.yaml        # constantes de la app (strategy: bluegreen)
    │   ├── dev/values-webserver-api01-dev.yaml
    │   ├── staging/values-webserver-api01-staging.yaml
    │   ├── production/values-webserver-api01-production.yaml
    │   ├── testing/values-webserver-api01-testing.yaml
    │   └── lab/values-webserver-api01-lab.yaml
    └── webserver-api02/
        ├── build-time/app.yaml        # constantes de la app (strategy: canary)
        ├── dev/values-webserver-api02-dev.yaml
        ├── staging/values-webserver-api02-staging.yaml
        ├── production/values-webserver-api02-production.yaml
        ├── testing/values-webserver-api02-testing.yaml
        └── lab/values-webserver-api02-lab.yaml
```

## Flujo CI/CD

1. Push de tag en el repo de la app → GitHub webhook → EventListener
2. CEL interceptor filtra `refs/tags/*` y extrae el tag limpio
3. TriggerTemplate crea un PipelineRun en el nodo `cicd` (taint `workload=cicd`)
4. Pipeline: **clone** → **build+push (Kaniko)** → **load test (k6)** → **bump gitops**
5. bump-gitops actualiza `image.tag` en `apps/<app>/<env>/values-*.yaml` de este repo
6. ArgoCD detecta el cambio y sincroniza el Rollout

## Rollout strategies

| App | Strategy | Descripción |
|-----|----------|-------------|
| webserver-api01 | `bluegreen` | Requiere promoción manual (`argo rollouts promote`) |
| webserver-api02 | `canary` | Avanza 5% → 25% → 50% → 100% con pausas manuales |

## Render manual

```bash
# api01 en dev
helm template webserver-api01 pythonapps/ \
  -f pythonapps/apps/webserver-api01/build-time/app.yaml \
  -f pythonapps/apps/webserver-api01/dev/values-webserver-api01-dev.yaml \
  -n dev

# api02 en staging
helm template webserver-api02 pythonapps/ \
  -f pythonapps/apps/webserver-api02/build-time/app.yaml \
  -f pythonapps/apps/webserver-api02/staging/values-webserver-api02-staging.yaml \
  -n staging
```

## Secretos requeridos (namespace `tekton-pipelines`)

| Secret | Descripción |
|--------|-------------|
| `dockerhub-credentials` | `{ ".dockerconfigjson": <base64 docker config> }` |
| `gitops-deploy-key` | SSH deploy key con write access a gitops-files (si usa SSH) |

