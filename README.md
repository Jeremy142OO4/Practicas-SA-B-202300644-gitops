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
secrets/      ExternalSecret sin valores sensibles.
```

## Promocion

`api-gateway` usa una estrategia canary con pesos de 10%, 30% y 60%. Cada
etapa ejecuta una validacion de humo, integracion o carga. Un resultado fuera
del umbral aborta el Rollout y conserva la version estable.

## Requisitos del cluster

Antes de sincronizar la aplicacion deben estar instalados ArgoCD, Argo
Rollouts, Kyverno, External Secrets Operator y el `ClusterSecretStore`
`sa-platform-secrets`. El store se configura fuera de este repositorio para no
guardar credenciales en texto plano.
