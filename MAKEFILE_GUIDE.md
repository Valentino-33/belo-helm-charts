# Guía rápida de comandos — belo-helm-charts

> Este repo no tiene Makefile. Los comandos son `helm` directo desde la raíz del repo.

---

## Verificar el chart antes de aplicar (dry-run)

```bash
# api01 en dev
helm lint pythonapps/ \
  -f pythonapps/apps/webserver-api01/build-time/app.yaml \
  -f pythonapps/apps/webserver-api01/dev/values-webserver-api01-dev.yaml

helm template webserver-api01 pythonapps/ \
  -f pythonapps/apps/webserver-api01/build-time/app.yaml \
  -f pythonapps/apps/webserver-api01/dev/values-webserver-api01-dev.yaml \
  --set image.tag=v0.1.0 | kubectl apply --dry-run=client -f -

# api02 en staging
helm lint pythonapps/ \
  -f pythonapps/apps/webserver-api02/build-time/app.yaml \
  -f pythonapps/apps/webserver-api02/staging/values-webserver-api02-staging.yaml
```

---

## Render del chart (debug de templates)

```bash
# Ver el YAML generado completo para api01 en dev
helm template webserver-api01 pythonapps/ \
  -f pythonapps/apps/webserver-api01/build-time/app.yaml \
  -f pythonapps/apps/webserver-api01/dev/values-webserver-api01-dev.yaml \
  --set image.tag=v0.1.0

# Ver solo el Rollout
helm template webserver-api01 pythonapps/ \
  -f pythonapps/apps/webserver-api01/build-time/app.yaml \
  -f pythonapps/apps/webserver-api01/dev/values-webserver-api01-dev.yaml \
  -s templates/rollout.yaml
```

---

## Render de los templates de Tekton

```bash
# Ver la Task de build
helm template tekton pythonapps/ \
  -f pythonapps/apps/webserver-api01/build-time/app.yaml \
  -s templates/pipeline-templates/task-build-kaniko.yaml

# Ver el Pipeline completo
helm template tekton pythonapps/ \
  -f pythonapps/apps/webserver-api01/build-time/app.yaml \
  -s templates/pipeline-templates/pipeline-pythonapps.yaml
```

---

## Estructura de valores por app y ambiente

```
pythonapps/apps/<app>/
├── build-time/app.yaml           # constantes: image name, registry, strategy
├── dev/values-<app>-dev.yaml     # valores de dev: replicas, recursos, ingress
├── staging/values-<app>-staging.yaml
├── production/values-<app>-production.yaml
├── testing/values-<app>-testing.yaml
└── lab/values-<app>-lab.yaml
```

Para cambiar la imagen de api01 en dev:
```bash
# Editar image.tag en:
# pythonapps/apps/webserver-api01/dev/values-webserver-api01-dev.yaml
# (Tekton hace esto automáticamente en cada build exitoso)
```

---

## Ambientes disponibles

| Ambiente | App 01 values | App 02 values |
|----------|---------------|---------------|
| dev | `dev/values-webserver-api01-dev.yaml` | `dev/values-webserver-api02-dev.yaml` |
| staging | `staging/values-webserver-api01-staging.yaml` | `staging/values-webserver-api02-staging.yaml` |
| production | `production/values-webserver-api01-production.yaml` | `production/values-webserver-api02-production.yaml` |
| testing | `testing/values-webserver-api01-testing.yaml` | `testing/values-webserver-api02-testing.yaml` |
| lab | `lab/values-webserver-api01-lab.yaml` | `lab/values-webserver-api02-lab.yaml` |
