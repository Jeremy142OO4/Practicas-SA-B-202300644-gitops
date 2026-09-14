# Practicas-SA-B-202300644-gitops

Repositorio publico e independiente de manifiestos declarativos para la
Practica 8.

## Fuente de verdad

El repositorio principal contiene el codigo de los microservicios y el
workflow. Este repositorio contiene unicamente el estado deseado de
Kubernetes, la aplicacion de ArgoCD, los Rollouts, los analisis, las politicas
y las referencias a secretos externos.

ArgoCD es el unico componente autorizado para aplicar estos manifiestos al
cluster. GitHub Actions solo abre un Pull Request para cambiar la version de
la imagen despues de ejecutar las validaciones.

## Estructura

```text
argocd/       Aplicacion ArgoCD.
manifests/    Recursos base por ambiente.
policies/     Politicas Kyverno de admision y firma.
rollouts/     Rollout canary, Services y AnalysisTemplates.
secrets/      ExternalSecret y acceso de solo lectura a secretos de P7.
```

## Promocion

`api-gateway` usa una estrategia canary con pesos de 10%, 30% y 60%. Cada
etapa ejecuta una validacion de humo, integracion o carga. Un resultado fuera
del umbral aborta el Rollout y conserva la version estable.

## Requisitos del cluster

ArgoCD, Argo Rollouts, Kyverno y External Secrets Operator deben estar
instalados. El `ClusterSecretStore` `sa-p7-kubernetes` usa un ServiceAccount
con permisos de solo lectura sobre los secretos de `sa-p7`; no se guardan
credenciales en texto plano en este repositorio.
