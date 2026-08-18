---
date: 2026-08-17
layout: post
title: "Poor Man's Fury - Parte 4"
subtitle: Catálogo y scaffolding
---

Escenas del episodio anterior: la Parte 3 cerró con `gostalgia-api` viéndose a sí misma de punta a punta — traces, métricas, logs correlacionados, un SLO real con burn rate — y un gancho hacia sistemas distribuidos: inyectar fallas reales, no un endpoint de demo ademas de ver si la instrumentación alcanzaba para diagnosticarlas sin adivinar, fase 4.5.

Esta parte retoma el orden original: Backstage, catálogo y scaffolding, fases 5 y 6.

## Fase 5 - Backstage: el catálogo

> **Backstage**: framework open source de developer portal, creado por Spotify y donado a la CNCF en 2020 (proyecto graduado desde 2022). Es lo más parecido que existe, en software libre, a lo que sería una Fury UI genérica: catálogo de servicios con ownership declarado, scaffolding de apps nuevas, documentación centralizada, y un ecosistema de plugins para enchufar todo lo demás en una sola pantalla. Dato aparte: Mercado Libre no usa Backstage, Fury es anterior y 100% propio — el paralelo de esta serie es genuino, no al revés.

`rathma` no tenía Node instalado. Backstage necesita una LTS activa:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

> **Issue 01**: la guía original usaba `npx @backstage/create-app@latest --name fury-portal`. La versión actual del CLI (`0.9.0`) sacó ese flag por completo — ahora es `--path` para la ubicación, y el nombre de la app se pregunta interactivo aparte. Mismo patrón un poco más adelante: la guía decía `yarn dev`, el script real en el `package.json` generado es `yarn start` (`yarn dev` tira directo `Usage Error: Couldn't find a script named "dev"`).

```bash
cd ~/dev
npx @backstage/create-app@latest --path fury-portal
# ? Enter a name for the app [required] fury-portal
```

El scaffolding en sí corrió bien, pero el install final falló con `yarn: not found` — Backstage usa Yarn vía Corepack, que viene con Node pero no viene habilitado:

```bash
sudo corepack enable
cd ~/dev/fury-portal
yarn install
```

Con la app instalada, dos secciones de `app-config.yaml` para conectarla con lo que ya existe:

```yaml
catalog:
  providers:
    github:
      furyApps:
        organization: 'mamcer-labs'
        catalogPath: '/catalog-info.yaml'
        filters:
          branch: 'main'
        schedule:
          frequency: { minutes: 5 }

kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
    - type: 'config'
      clusters:
        - name: poor-mans-fury
          url: https://127.0.0.1:6443
          authProvider: 'serviceAccount'
          serviceAccountToken: ${K8S_SA_TOKEN}
          caData: ${K8S_CA_DATA}
```

El provider de GitHub escanea la org cada 5 minutos buscando un `catalog-info.yaml` en cada repo — el mismo mecanismo por el que `gostalgia` ya quedó descubrible con un archivo de 15 líneas en su raíz. La parte de Kubernetes necesita credenciales propias, con el mismo principio de mínimo privilegio que ya usa el resto del cluster:

```bash
kubectl create serviceaccount backstage -n fury-infra
kubectl create clusterrolebinding backstage-reader --clusterrole=view --serviceaccount=fury-infra:backstage
```

Un `ClusterRole: view` — solo lectura, sin capacidad de crear ni borrar nada — es más que suficiente para que el catálogo muestre estado real sin poder tocarlo.

Con todo levantado (`yarn start`, que corre frontend y backend juntos en un solo proceso), entrar a la UI desde otra máquina en la LAN tiró un error que no tenía nada que ver con Backstage en sí:

> **Issue 02**: `TypeError: globalThis.crypto.randomUUID is not a function`, al loguearse como guest. Causa: `crypto.randomUUID` (Web Crypto API) solo existe en un *secure context* — HTTPS, o el caso especial de `localhost`. Acceder por `http://192.168.100.100:3000` no cuenta, aunque cargue la página sin drama aparente. Fix sin certificados de por medio: túnel SSH desde la notebook, para que el navegador vea el origen como `localhost` de verdad. Un segundo tropiezo en el mismo lugar — el primer intento de túnel (`ssh -L 3000:localhost:3000 ...`) daba `connect failed: Connection refused` en loop, porque `localhost` del lado del servidor resolvía primero a `::1` (IPv6) y Backstage escuchaba en IPv4. Forzar `127.0.0.1` explícito en el forwarding lo resolvió.

Con el catálogo arriba y `gostalgia` registrado (un `catalog-info.yaml` de 15 líneas, commiteado directo a `main`), la pestaña de Kubernetes del componente mostraba `No Kubernetes resources` — sin ningún error, la auth ya andaba bien.

> **Issue 03**: el plugin de Kubernetes no filtra recursos por namespace solamente, filtra por el label `backstage.io/kubernetes-id=<id>` en cada objeto — `Deployment`, `Service`, y también los `Pod`. El manifiesto real de `gostalgia-api` no tenía ese label en ningún lado. Se vuelve a encontrar, casi textual, el mismo tipo de bug de la Parte 3 con el `ServiceMonitor`: un selector que filtra por labels de un objeto específico, no por el namespace ni por el selector de otro objeto relacionado. El fix va en el YAML fuente, no con `kubectl label` suelto — en tres lugares: metadata del `Deployment`, metadata del template de Pod (para que los pods lo hereden), y metadata del `Service`.

```yaml
metadata:
  labels:
    backstage.io/kubernetes-id: gostalgia-api
```

Con eso aplicado, la pestaña pasó a mostrar el `Deployment` y el `Pod` reales de `fury-dev`, corriendo.

![catálogo de Backstage con gostalgia mostrando el Deployment y el Pod reales en fury-dev](../img/2026-08-17-poor-mans-fury-part-04/backstage-kubernetes-tab.png)

## Fase 6 - Software Templates: el scaffolding

> **Software Template**: el equivalente de Backstage al scaffolding de una Fury UI real — un formulario parametrizado que, al ejecutarse, genera un repo nuevo desde una plantilla, lo publica, y lo conecta con todo lo demás. La idea es que crear un servicio nuevo no dependa de copiar y pegar un repo viejo a mano y limpiar lo que sobra.

Antes de escribir una sola línea del template hubo que resolver un choque de sintaxis nada obvio: Backstage (`${{ }}`) y GitHub Actions (`${{ }}`) usan exactamente la misma notación de templating. Si el pipeline de CI viviera en la parte del skeleton que Backstage procesa y renderiza, intentaría interpretar también las expresiones de GitHub Actions (`${{ github.sha }}`, `${{ secrets.GITHUB_TOKEN }}`) y las rompería. La solución no es escapar cada expresión, es separar el skeleton en dos partes con dos acciones distintas: `fetch:template` (con templating activo) solo para lo que necesita valores reales del formulario — `catalog-info.yaml`, `go.mod`, `main.go`, `README.md` — y `fetch:plain` (copia literal, sin tocar nada) para `Dockerfile` y el workflow de CI, que además se escribieron para no necesitar ningún valor inyectado, usando `${{ github.repository }}` nativo de Actions en vez de un nombre de imagen pasado por parámetro.

El template completo son 6 pasos: generar el código base (templado), copiar CI/CD y Dockerfile (estático), publicar el repo nuevo en GitHub, generar el manifest de deploy en un directorio aparte (templado), abrir un PR con ese manifest contra el repo de infra, y registrar la app en el catálogo.

```yaml
steps:
  - id: fetch-templated
    action: fetch:template
    input:
      url: ./skeleton
      values:
        name: ${{ parameters.name }}
        scope: ${{ parameters.scope }}

  - id: fetch-static
    action: fetch:plain
    input:
      url: ./skeleton-static

  - id: publish-repo
    action: publish:github
    input:
      repoUrl: github.com?owner=mamcer-labs&repo=${{ parameters.name }}

  - id: pr-deploy-manifest
    action: publish:github:pull-request
    input:
      repoUrl: github.com?owner=mamcer-labs&repo=poor-mans-fury
      sourcePath: ./deploy-manifest
      targetPath: deploy/${{ parameters.scope }}

  - id: register
    action: catalog:register
```


La primera corrida real, con un nombre de prueba (`fury-template-test`), pasó los 6 pasos sin un solo error: repo creado, PR abierto contra el repo de infra, app registrada en el catálogo. Pero el CI del repo nuevo falló:

> **Issue 04**: `missing go.sum entry for module providing package github.com/gin-gonic/gin`. El `go.mod` generado solo listaba las dos dependencias directas, sin `go.sum` ni el bloque `// indirect` — con el grafo de módulos podado que usa Go desde la 1.17, comandos en modo `-mod=readonly` (el default) no pueden resolver dependencias transitivas sin eso. El primer intento de fix, agregar `go mod download` al pipeline, no alcanzó: baja los paquetes pero no completa el `go.sum` para todo el árbol transitivo, hace falta `go mod tidy`. En vez de correr eso en cada CI de cada app generada — funcionaría, pero nadie commitea un `go.mod` sin su `go.sum` en un repo Go de verdad — se generó el `go.sum` real una única vez, contra las versiones fijas de `gin`/`client_golang` que usa el template, y quedó embebido como archivo estático: no varía entre apps generadas porque las versiones no varían.

Con el fix aplicado y confirmado en el repo de prueba, el pipeline completo corrió verde de punta a punta — lint, test, build, imagen publicada en GHCR — antes de darlo por bueno en la fuente del template. `fury-template-test` cumplió su propósito y se borró: PR cerrado sin mergear, repo eliminado.

![resultado del template go-service: los 6 steps en verde con los links al repo, al PR del manifest y al catálogo](../img/2026-08-17-poor-mans-fury-part-04/backstage-template-result.png)

## Dónde quedamos

Backstage quedó completo en sus dos mitades: un catálogo que descubre servicios solo y muestra su estado real en Kubernetes, y un scaffolding que genera un servicio nuevo con CI/CD ya verde desde el primer push — sin ese primer push roto por un `go.sum` faltante, que es exactamente el tipo de fricción que un golden path debería absorber.

Lo que no hace todavía: el PR del manifest de deploy no se aplica solo al cluster, y el policy de Vault de una app nueva sigue siendo un paso manual. Conectar eso es literalmente la fase que sigue — el flujo completo de punta a punta.