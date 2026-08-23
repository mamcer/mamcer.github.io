---
date: 2026-08-22
layout: post
title: "Poor Man's Fury - Parte 5"
subtitle: CLI y flujo completo
---

Escenas del episodio anterior: la Parte 4 cerró con Backstage completo, un catálogo que descubre servicios solos y un scaffolding que genera un servicio nuevo con CI/CD ya verde desde el primer push. Faltaban dos cosas para que eso fuera más que una demo: una forma simple de operar lo que el scaffolding genera, y la prueba de que todo el circuito - de crear una app a verla corriendo con métricas y logs solos - cierra de verdad. Fases 7 y 8, las últimas del roadmap original.

## Fase 7 - La CLI propia (pmf)

> **Por qué una CLI, si ya existe `kubectl`?**: `pmf deploy app version scope` es más rápido de escribir y de leer que el `kubectl set image` equivalente, amemas encapsula la convención (qué namespace corresponde a qué scope, qué label busca cada comando) en un solo lugar. `kubectl` no sabe qué es un "scope", `pmf` sí.

Diseño simple: `pmf` asume que el nombre de la app es el mismo string en los tres lugares donde importa — el `Deployment`/`Service`/contenedor en Kubernetes, el repo en el registry (`ghcr.io/mamcer-labs/<app>`), y la base del path en Vault (`fury/apps/<app>/<scope>/config`). Cierto por diseño para cualquier app scaffoldeada en fase 6, así que los 6 subcomandos (`deploy`, `logs`, `status`, `rollback`, `scope list`, `config get`) no necesitan ningún mapeo de nombres.

```bash
pmf status gostalgia-api
```
```
SCOPE      READY        STATUS     IMAGEN
dev        1/1          up         ghcr.io/mamcer-labs/gostalgia-api:latest
staging    -            no-deploy  -
prod       -            no-deploy  -
```

> **Issue 01**: antes de instalarlo, armé un `kubectl` falso en bash para probar el control de flujo del script bajo `set -e`, sin tocar el cluster real. Apareció un patrón de bug genuino: dos lugares usaban `condición && acción` como guard (`[[ "$found" -eq 0 ]] && die "..."`, y el flag `-f` de `logs`). Bajo `set -e`, si la condición de un `&&` da **falso** —el camino normal, no el de error— bash trata eso como un comando que "falló" y aborta el script entero ahí mismo. En la práctica: `pmf logs app scope` sin `-f` (el uso más común) se hubiera cortado en seco cada vez, y lo mismo `pmf scope list` cuando la app sí existe en algún scope — exactamente el camino feliz. El fix es reemplazar esos guards por `if cond; then acción; fi` — el patrón inverso, `comando || die "..."`, sí es seguro bajo `set -e`, por eso se usa en el resto del script sin problema.

```bash
pmf config get gostalgia-api dev
```
```
No value found at fury/data/apps/gostalgia-api/dev/config
```

> **Issue 02**: `gostalgia` es la app piloto, creada en fase 2-3 antes de que existiera la convención de "un solo nombre para todo" que sí siguen las apps scaffoldeadas por el template. Su `Deployment` se llama `gostalgia-api`, pero su path en Vault es `fury/apps/gostalgia/...`, sin el sufijo. No es un bug de diseño de `pmf` — es deuda histórica puntual de una sola app, documentada en vez de agregarle al CLI una normalización de nombres para un caso único que no se repite hacia adelante.

```bash
pmf config get gostalgia dev
```
```json
{
  "DB_HOST": "192.168.100.100",
  "DB_NAME": "gostalgia_dev",
  "DB_USER": "gostalgia"
}
```

El resto se probó contra el cluster real sin sorpresas: `deploy` a un SHA real de un commit anterior, confirmado con `status`, y `rollback` de vuelta a `latest` — con un warning esperado de `kubectl` sobre `last-applied-configuration` (mezclar `kubectl apply` original con `rollout undo` imperativo desincroniza esa anotación puntual, no el estado real del cluster).

## Fase 8 - El flujo completo, de cero a producción

No es una fase de instalación, es la validación de que fases 1-7 funcionan juntas en un solo flujo automático: push a `main` → CI corre lint/test/build → `pmf deploy` sin tocar nada a mano.

> **Qué se automatiza y qué no**: el PR que abre el template de fase 6 con el `Deployment` inicial **no se mergea solo, a propósito**. Bootstrapear un servicio nuevo —su `Deployment`, el policy de Vault que lo acompaña— es una acción infrecuente que amerita un humano mirando una vez. Lo que sí se automatiza es todo deploy *posterior* sobre un `Deployment` que ya existe. Es la misma distinción entre crear infra y desplegar una versión nueva de infra que ya existe — la primera es rara, la segunda pasa todo el tiempo y no debería frenar en un humano.

El `deploy-dev` de `gostalgia` estaba comentado desde fase 2, con un `kubectl set image` de ejemplo. Al reemplazarlo por una sola línea:

```yaml
deploy-dev:
  runs-on: [self-hosted, nuc]
  needs: build-and-push
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  steps:
    - name: Deploy a fury-dev
      run: pmf deploy gostalgia-api ${{ github.sha }} dev
```

> **Issue 03**: el stub comentado tenía `app=${{ env.IMAGE_API }}:...` como nombre de contenedor. El contenedor real de `gostalgia-api` se llama `gostalgia-api`, no `app`. Con `pmf` el problema ni se plantea: el nombre del contenedor es un argumento explícito, no algo escondido en un `kubectl set image` copiado de otro lado.

Push real a `main`, corrida completa observada en vivo:

![pipeline de gostalgia con los 4 jobs en verde, incluyendo deploy-dev](../img/2026-08-22-poor-mans-fury-part-05/gostalgia-full-pipeline.png)

`pmf status gostalgia-api` confirmó la imagen corriendo con el SHA exacto del commit pusheado. Circuito cerrado para una app que ya existía — pero eso no prueba el "PASO 1" del roadmap, crear una app *nueva* desde cero. Para eso, mismo wiring en el template (con el nombre de app derivado de `GITHUB_REPOSITORY`, genérico para cualquier repo) y una app real nueva: `cookbook`.

Los 6 steps del scaffolder corrieron en verde. El primer push a `cookbook` disparó el pipeline solo — y `deploy-dev` **falló**, como correspondía: el `Deployment` todavía no existía.

![primer deploy de cookbook fallando con el mensaje propio de pmf, el resto de los jobs en verde](../img/2026-08-22-poor-mans-fury-part-05/cookbook-first-deploy-fails.png)

El log mostró `pmf: no existe el deployment 'cookbook' en fury-dev` — no un error crudo de `kubectl`, confirma que el manejo de errores de fase 7 funciona también dentro de CI, no solo en una terminal interactiva. Bootstrap manual: mergear el PR, `kubectl apply`, y Vault (secret placeholder, policy, role — mismo patrón que `gostalgia-dev`). El pod pasó de `Init:0/1` (el sidecar de Vault sin poder autenticarse) a `2/2 Running`. Sin pushear un commit nuevo, re-correr el job que había fallado alcanzó:

![el mismo pipeline de cookbook, ahora con deploy-dev en verde tras el bootstrap](../img/2026-08-22-poor-mans-fury-part-05/cookbook-deploy-succeeds.png)

> **Issue 04**: con `cookbook` corriendo, los logs aparecieron solos en Loki —Promtail es cluster-wide, no necesita configuración por app— pero `kubectl get servicemonitor -n fury-dev` solo listaba `gostalgia-api`, armado a mano en la Parte 3. El template de fase 6 nunca generaba un `ServiceMonitor` para apps nuevas: un gap real contra la propia promesa de esta fase ("métricas y logs apareciendo solos"). Fix: agregado como cuarto documento YAML en el manifest de deploy templado, con el mismo cuidado que en la Parte 3 —el `selector` apunta a labels del `Service`, no al `spec.selector` que es para pods—. Confirmado contra la API real de Prometheus: `{"health": "up", "lastError": ""}`.

`cookbook` quedó viviendo como segunda app real de la plataforma — a diferencia de la app de prueba de la Parte 4, esta vale la pena mantenerla: es la prueba de que el golden path no es solo para `gostalgia`.

## Dónde quedamos

Las 8 fases del roadmap que arrancó en la Parte 1 están completas. Un cluster que corre solo, secrets que nunca tocan un `.env`, una app que se ve a sí misma de punta a punta, un catálogo que descubre servicios sin que nadie los registre a mano, un scaffolding que genera código con CI/CD ya verde, y ahora un CLI y un flujo que cierran el círculo completo: de un clic en Backstage a un pod corriendo con métricas y logs, sin que nadie tipee un `kubectl` de memoria.

Poor Man's Fury ya demostró lo que tenía que demostrar. Lo que sigue no es una fase más, es usar todo esto para lo que se armó: evidencia real de cómo se piensa una plataforma, no solo de cómo se usa una.