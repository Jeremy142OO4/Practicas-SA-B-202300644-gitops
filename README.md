# Practicas-SA-B-202300644-gitops

Repositorio público de manifiestos declarativos para la Práctica 8.

## Principio

Este repositorio no contiene código de aplicación ni Dockerfiles. El código fuente permanece en el repositorio principal:

https://github.com/Jeremy142OO4/Practicas-SA-B-202300644

ArgoCD utilizará este repositorio como fuente de verdad para el estado deseado del clúster.

## Estructura

```text
argocd/       Aplicaciones de ArgoCD.
manifests/    Recursos declarativos por ambiente.
policies/     Políticas Kyverno u OPA.
rollouts/     Rollouts y AnalysisTemplates.
secrets/      Referencias a secretos cifrados o externos.
```

La configuración inicial de `manifests/dev/` solo prepara el namespace de trabajo. La sincronización automática se habilitará después de validar los manifiestos de la aplicación.
