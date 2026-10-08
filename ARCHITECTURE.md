# ARCHITECTURE.md — sc-pilot-sandbox

> Repo de pruebas E2E de la compañía (F4 del piloto de los bots de rol). Sin código de producción ni despliegue.
> Escrito por `architect` (INFRA-550).

## 1. Clientes y versiones

Ninguno: no es un producto. Es un campo de pruebas con rama única `main`; ArgoCD no lo despliega (no hay Application).

## 2. Dependencias, en ambos sentidos

- **Depende de** — `pocharlies-org/k8s-gitops-pocharlies`: `.github/workflows/reusable-pr-review.yml@main` y
  `reusable-duplicados.yml@main`; secretos de organización (`LITELLM_CI_KEY`, `TELEGRAM_CI_BOT_TOKEN`,
  `TELEGRAM_CI_CHAT_ID`) por `secrets: inherit`.
- **Dependen de él** — los bots de rol (E2E del piloto) y, desde INFRA-550, la prueba de disparo de la alerta de cola de CI:
  `ci-queue-probe.yml` → `ci-queue-exporter` (gitops, `ci-queue/`) → `VMRule pocharlies-arc-ci`
  (`k8s-observability-pocharlies`, `manifests/arc-rules.yaml`) → regla de correlación `ci-queue-degraded` de Keep → topic Infra 1248.

## 3. Stack

Solo GitHub Actions. Sin runtime propio. `pull_request`, nunca `pull_request_target` (este corre con secretos del repo base
sobre código de un fork).

## 4. Componentes compartidos

| concepto | pieza canónica | ruta | quién la usa |
|---|---|---|---|
| Review automática de PR | `reusable-pr-review.yml` | gitops `.github/workflows/` | `pr-review.yml` |
| Detector de copias | `reusable-duplicados.yml` (check `duplicados / duplicados`) | gitops `.github/workflows/` | `duplicados.yml` |

## 5. Cómo se construye aquí

Los workflows estándar se copian tal cual desde gitops y se llaman al reusable. La excepción es la sonda
`ci-queue-probe.yml` (INFRA-550): sin reusable, solo `workflow_dispatch`, `permissions: {}`.

## 6. Tests y validaciones

Sin tests de código. Validación: los checks `duplicados / duplicados` y `review / Review del PR`. La prueba E2E de la sonda la
ejecuta qa: `gh workflow run ci-queue-probe.yml -R pocharlies-org/sc-pilot-sandbox`, ver `CIQueueLabelWithoutPool` (~10 min),
el incidente `CIQ-…` en Keep y el aviso en el topic 1248 (≤ 25 min), y **siempre** `gh run cancel <run-id>` al terminar.

## 7. CI/CD y despliegue

Sin despliegue. Merge a `main` por PR; el guardia de merge exige el check `duplicados / duplicados`.

## 8. Decisiones y trampas

- `ci-queue-probe-sin-pool` no debe llegar a ser nunca un `runnerScaleSetName` ni un label de runner real: si lo es, la sonda
  deja de probar nada (su único paso hace `echo` para que se note).
- El exporter solo mira repos con push en los últimos 7 días: un commit al repo lo mantiene visible.
- Solo ve runs en cola SIN ningún job arrancado: la sonda es de un solo job a propósito.
- Un run de sonda olvidado deja un incidente abierto en Keep: cancelarlo siempre.

Última verificación contra el código: 2026-10-08 · 377f7647
