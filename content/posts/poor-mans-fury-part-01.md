---
date: 2026-07-21
layout: post
title: "Poor Man's Fury - Parte 1"
subtitle: Cluster, ingress y el primer pipeline
---

En mis casi seis años trabajando en Mercado Libre aprendí y crecí como profesional de una forma que no lo había hecho antes. Como todo aprendizaje eso ya forma parte de uno y te acompaña adonde vayas. Pero hay cosas que uno no se lleva. De esas, la que más había naturalizado y que la realidad de trabajar en una startup se encargó de demostrarme, fue Fury.

Fury es la plataforma interna de Mercado Libre para crear y operar microservicios: CI/CD, secrets, observabilidad y catálogo. Todo eso ya resuelto por la plataforma. Vos creás el servicio, la plataforma se encarga del resto.

En ese contexto también oculta, se trabaja a otro nivel de abstracción. En casi cualquier startup los niveles de abstracción son de pequeños a nulos. Un comportamiento anómalo te puede tener rápidamente en un war room frente a Datadog, traces, logs, un nodo y una DB a tu disposición para diagnosticar y estabilizar una situación. Nunca nadie está solo y hoy además se suman los superpoderes que nos da un LLM.

Pero justamente por eso, entender qué está sucediendo under-the-hood creo que es más valioso que nunca. Un LLM te puede sugerir un diagnóstico, tener un entendimiento de la situación te permite confirmarlo o darte cuenta cuando está equivocado y ahorrarte preciados minutos persiguiendo falsas hipótesis.

En 2017 compré un Intel NUC: i7, 16GB de RAM para aprender armando un home server de datos y deployar toy applications: Jenkins, SonarQube, Docker, MySQL. Esto fue aprendizaje propio, entender qué está sucediendo, cómo se conectan los engranajes. Aquí una guía dividida en diferentes partes acerca del viaje para crear otro tipo de monstruo, un Poor Man's Fury con herramientas open source y presupuesto cero.

![nuc](../img/2026-07-21-poor-mans-fury-part-01/nuc.jpg)

## Fase 1 - La base: Ubuntu Server, K3s, Traefik

La base es **Ubuntu Server 26.04**, instalación mínima, con SSH habilitado desde el instalador. IP fija vía netplan:

```yaml
# vim /etc/netplan/0-installer-config.yaml
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: no
      addresses:
        - 192.168.100.100/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Firewall básico:

```bash
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 6443/tcp   # K3s API
sudo ufw allow 443/tcp    # Traefik HTTPS
sudo ufw allow 80/tcp     # Traefik HTTP
sudo ufw enable
```

**K3s**, sin el Traefik que trae embebido por default (para instalar aparte con más control):

> K3s es una distribución liviana de Kubernetes hecha por [Rancher](https://www.rancher.com/products/k3s): mismo API y compatibilidad, pero con un binario único, sin componentes que en un clúster pequeño no hacen tanto sentido (etcd por defecto reemplazado por SQLite, sin cloud controller manager, etc.) pensada para edge/IoT y setups chicos.

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \
  --write-kubeconfig-mode 644

mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
kubectl get nodes
```

**Namespaces por scope**, la misma idea de aislamiento por ambiente que tenía Fury, en miniatura:

```bash
kubectl create namespace fury-infra
kubectl create namespace fury-monitoring
kubectl create namespace fury-dev
kubectl create namespace fury-staging
kubectl create namespace fury-prod

kubectl label namespace fury-dev      fury/scope=dev     fury/env=test
kubectl label namespace fury-staging  fury/scope=staging fury/env=demo
kubectl label namespace fury-prod     fury/scope=prod    fury/env=production
```

**Traefik** vía Helm como `LoadBalancer`. K3s trae su propio service-lb, no hace falta nada externo:

> Traefik = ingress controller/reverse proxy para k8s. Se integra nativo con la CRD de k8s (menos YAML), tiene dashboard propio para debuggear rutas, y hace service discovery automático, cero reload manual cuando se agrega un servicio nuevo.

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

cat > traefik-values.yaml << 'EOF'
deployment:
  replicas: 1
service:
  type: LoadBalancer
ingressRoute:
  dashboard:
    enabled: true
additionalArguments:
  - "--api.insecure=true"
  - "--log.level=INFO"
EOF

helm install traefik traefik/traefik \
  --namespace kube-system \
  --values traefik-values.yaml
```

> **Nota de versión:** los `IngressRoute` de Traefik migraron su CRD de
> `traefik.containo.us/v1alpha1` a `traefik.io/v1alpha1`. Si copiás YAMLs
> de guías más viejas y te tira `no matches for kind "IngressRoute"` seguramente es
> por esto. Podés confirmar con la versión que tenés con: `kubectl get crd | grep traefik`.

## Fase 2 - CI/CD con runner self-hosted

Decisión de diseño: **GitHub Actions con un runner propio en el NUC**, no Gitea/Woodpecker por ejemplo. Dos razones: repos actualmente en GitHub e intención de mantenerlos ahí y el runner hace polling saliente, cero puertos entrantes que abrir.

### Instalar el runner

```bash
gh auth login   # scopes: repo, workflow, read:org

mkdir -p ~/actions-runner && cd ~/actions-runner
RUNNER_VERSION=2.335.1
curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L \
  https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
tar xzf actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

./config.sh --url https://github.com/TU_ORG --token TOKEN_DE_GITHUB \
  --labels self-hosted,linux,x64,nuc

sudo ./svc.sh install
sudo ./svc.sh start
```

> **Issue 01:** si corrés `config.sh` sin `--labels` explícito (el
> wizard interactivo no lo pide), el runner queda solo con las labels
> autodetectadas (`self-hosted`, `linux`, `x64`), sin `nuc`. Si tu
> workflow pide `runs-on: [self-hosted, nuc]`, el job se queda en
> `Queued` para siempre, sin error visible. Se agrega la label después
> desde GitHub > org > Settings > Actions > Runners.

> **Issue 02:** si el repo es **público**, GitHub bloquea
> por default que runners self-hosted tomen esos jobs (un PR malicioso
> podría ejecutar código en tu hardware). Mismo síntoma: `Queued` eterno.
> Con un proyecto personal, lo más simple es pasar el repo a privado.

### El pipeline

```yaml
# .github/workflows/release.yml
name: Release Process

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  IMAGE_API: ghcr.io/TU_ORG/TU_APP-api

jobs:
  lint:
    runs-on: [self-hosted, nuc]
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v6
        with:
          go-version-file: 'go.mod'
          cache: false
      - run: go vet ./...
      - run: |
          fmt_out=$(gofmt -l .)
          [ -n "$fmt_out" ] && { echo "$fmt_out"; exit 1; } || true
      - uses: golangci/golangci-lint-action@v9
        with:
          version: v2.5

  test:
    runs-on: [self-hosted, nuc]
    needs: lint
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v6
        with:
          go-version-file: 'go.mod'
          cache: false
      - env:
          CGO_ENABLED: 1
        run: go test ./... -race -cover

  build-and-push:
    runs-on: [self-hosted, nuc]
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v6
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile.api
          push: true
          tags: |
            ${{ env.IMAGE_API }}:${{ github.sha }}
            ${{ env.IMAGE_API }}:latest
```

Esta es la versión que quedó funcionando. Algunos issues en el camino:

> **`go-version-file` en lugar de un número fijo.** La primera versión tenía
> `go-version: '1.22'` hardcodeado, mientras el `go.mod` real pedía
> `>= 1.25.6`. Leer la versión directo del `go.mod` evita esta desincronización.

> **`golangci-lint` pinneado a `v2.5`.** El binario default que instala la
> action (`v1.64.8`) está compilado con una versión de Go más vieja que la
> del proyecto y golangci-lint no analiza código de una versión de Go
> más nueva que la propia. El error es explícito en este caso: `the Go language version
> used to build golangci-lint is lower than the targeted Go version`.

> **`CGO_ENABLED: 1` solo en el step de test.** El flag `-race` necesita
> `cgo`, que viene apagado por default. Esto es puntual al test, no
> al build de la imagen, ahí `cgo` se mantiene apagado a propósito para
> generar un binario estático más liviano. Además, si tu base es Ubuntu Server
> mínimo, probablemente te falte `gcc`: `sudo apt install build-essential`.

> **`docker/setup-buildx-action` explícito.** En runners hosteados por
> GitHub, buildx viene preinstalado, en self-hosted no. Y si instalaste
> Docker con el paquete `docker.io` de Ubuntu (en vez del repo oficial
> `docker-ce`), tampoco tenés el plugin, hay que sumarlo a mano:

```bash
mkdir -p ~/.docker/cli-plugins
curl -SL https://github.com/docker/buildx/releases/download/v0.34.1/buildx-v0.34.1.linux-amd64 \
  -o ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx
```

### Verificar

```bash
docker login ghcr.io -u TU_USER --password-stdin <<< $(gh auth token)
docker pull ghcr.io/TU_ORG/TU_APP-api:latest
```

Si baja la imagen sin error, el circuito completo: push > lint > test > build > publish, está cerrado.

## Dónde quedamos

El runner quedó corriendo como servicio, sobreviviendo reboots. Pipeline en verde en cada push a `main`, imagen real publicada en GitHub Container Registry. Nada de esto tiene forma segura de manejar un secret aún, eso es parte de los próximos pasos.