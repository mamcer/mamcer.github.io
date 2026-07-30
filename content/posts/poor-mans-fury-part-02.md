---
date: 2026-07-30
layout: post
title: "Poor Man's Fury - Parte 2"
subtitle: Vault y observabilidad
---

Como escenas del episodio anterior, en el primer post dejamos: un runner corriendo como servicio en el NUC, un pipeline en verde en cada push a main, y una imagen real publicada en GitHub Container Registry. Pero nada de eso tenía todavía forma segura de manejar un secret. Cualquier credencial que la app necesitara iba a terminar en por ejemplo un .env suelto.

En esta parte se implementa la gestión de secrets con Vault, además se agrega observabilidad con Prometheus, Loki y Grafana. Al final de esta parte una aplicación debería correr en el cluster con sus credenciales inyectadas automáticamente exponiendo sus métricas y logs sin necesidad de que alguien se loguee a un pod a mano para verlo.

## Fase 3 - Vault como Configuration Service

Si mal no recuerdo en Fury esto se llama Configuration Service: config separada del código, organizada por scope. Vault de HashiCorp sería un equivalente open source.

> Vault: gestor de secrets centralizado. En lugar de que cada app tenga sus credenciales repartidas en .env sueltos, todo vive encriptado en un solo lugar, organizado por paths, con políticas de acceso granulares por quién (o qué ServiceAccount) puede leer qué.

### Instalar y desbloquear

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

```bash
cat > vault-values.yaml << 'EOF'
server:
  dev:
    enabled: false
  standalone:
    enabled: true
  dataStorage:
    size: 5Gi
ui:
  enabled: true
EOF
```

```bash
helm install vault hashicorp/vault \
  --namespace fury-infra \
  --values vault-values.yaml
```

> Nota: se eligió standalone no dev. El modo dev de Vault no persiste nada, se pierde todo en cada reinicio del pod.

Vault nace sellado, encriptado e inaccesible hasta que le des la master key:

```bash
kubectl exec -n fury-infra vault-0 -- vault operator init \
  -key-shares=1 -key-threshold=1 -format=json > vault-init.json

kubectl exec -n fury-infra vault-0 -- vault operator unseal \
  $(cat vault-init.json | jq -r '.unseal_keys_b64[0]')
```

`vault-init.json` guarda la unseal key y el root token, la única vez que Vault los muestra en texto plano. Sin ese archivo no hay forma de recuperar lo que se guarde en el vault. No va en ningún repo de app, guardar backup(s).

### Secrets por scope, y autenticación vía Kubernetes

```bash
kubectl exec -n fury-infra vault-0 -- vault login $(cat vault-init.json | jq -r '.root_token')
kubectl exec -n fury-infra vault-0 -- vault secrets enable -path=fury kv-v2
```

```bash
kubectl exec -n fury-infra vault-0 -- vault kv put \
  fury/apps/gostalgia/dev/config \
  DB_HOST="192.168.100.100" DB_PORT="3306" \
  DB_USER="gostalgia" DB_PASSWORD="[db_pwd]" DB_NAME="gostalgia_dev"
```

Para que un pod pueda leer esto sin que vos le pases un token a mano, Vault se autentica directo contra el propio Kubernetes:

```bash
kubectl exec -n fury-infra vault-0 -- vault auth enable kubernetes
kubectl exec -n fury-infra vault-0 -- vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"
```

```bash
cat > gostalgia-policy.hcl << 'EOF'
path "fury/data/apps/gostalgia/dev/config" {
  capabilities = ["read"]
}
EOF
```

```bash
kubectl cp gostalgia-policy.hcl fury-infra/vault-0:/tmp/
kubectl exec -n fury-infra vault-0 -- vault policy write gostalgia-dev /tmp/gostalgia-policy.hcl
```

```bash
kubectl exec -n fury-infra vault-0 -- vault write auth/kubernetes/role/gostalgia-dev \
  bound_service_account_names=gostalgia-api \
  bound_service_account_namespaces=fury-dev \
  policies=gostalgia-dev ttl=1h
```

### Deployment con inyección automática

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gostalgia-api
  namespace: fury-dev
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "gostalgia-dev"
        vault.hashicorp.com/agent-inject-secret-config: "fury/data/apps/gostalgia/dev/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "fury/data/apps/gostalgia/dev/config" -}}
          export DB_HOST="{{ .Data.data.DB_HOST }}"
          export DB_PASSWORD="{{ .Data.data.DB_PASSWORD }}"
          {{- end -}}
    spec:
      serviceAccountName: gostalgia-api
      imagePullSecrets:
        - name: ghcr-pull-secret
      containers:
        - name: gostalgia-api
          image: ghcr.io/TU_ORG/gostalgia-api:latest
          command: ["sh", "-c", "source /vault/secrets/config && exec /app/gostalgia-api"]
```

El Vault Agent corre como sidecar, escribe un archivo con las variables ya resueltas, y el command del container las carga antes de arrancar el binario real.

**Issue 01:** Mi sesión de gh no tenía el scope read:packages. Sin un imagePullSecret, el pod queda en ErrImagePull. El error es un 403 Forbidden seco, sin ninguna pista de qué scope falta. Aquí sin un LLM seguramente hubiese estado un buen tiempo martillando.

```bash
gh auth refresh -h github.com -s read:packages
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io --docker-username=TU_USER \
  --docker-password=$(gh auth token) --namespace=fury-dev
```

Ahora `kubectl get pods -n fury-dev` muestra `2/2 Running`, el container de la app y el sidecar de Vault, ambos sanos, sin que un solo secret haya pasado por un .env.

## Fase 4 - Prometheus, Loki, Grafana

### Métricas

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
```

```bash
cat > prometheus-values.yaml << 'EOF'
grafana:
  enabled: true
  adminPassword: "[admin_pwd]"
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 20Gi
EOF
```

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace fury-monitoring --create-namespace \
  --values prometheus-values.yaml
```

> kube-prometheus-stack: Un solo helm install trae Prometheus (recolección y almacenamiento de métricas), Grafana (visualización), AlertManager (alertas) y los exporters que se necesitan para que el cluster empiece a reportar de entrada, sin instrumentar nada a mano.

### Logs

```bash
cat > loki-values.yaml << 'EOF'
deploymentMode: SingleBinary
loki:
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
singleBinary:
  replicas: 1
  persistence:
    size: 20Gi
read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0
EOF
```

```bash
helm install loki grafana/loki --namespace fury-monitoring --values loki-values.yaml
```

```bash
helm install promtail grafana/promtail --namespace fury-monitoring \
  --set "config.lokiAddress=http://loki-gateway.fury-monitoring.svc.cluster.local/loki/api/v1/push" \
  --set "config.snippets.extraClientConfigs=tenant_id: poor-mans-fury"
```

> Loki: el "Prometheus de los logs" (?). En vez de indexar el contenido completo de cada línea (como por ejemplo Elasticsearch), solo indexa metadata (labels) y comprime el resto, mucho más liviano para un homelab. Promtail es el agente que corre en cada nodo, junta los logs de todos los pods y se los manda.

**Issue 02**: el chart de Loki cambió de esquema entre versiones y no avisa de la mejor forma. Las versiones recientes vienen en modo SimpleScalable por default, con componentes read, write y backend separados. Si solo configurás singleBinary sin declarar deploymentMode: SingleBinary explícito, el chart detecta ambos modos activos a la vez y directamente se niega a instalar: "You have more than zero replicas configured for both the single binary and simple scalable targets."

**Issue 03**: Loki, por default, exige multi-tenancy. Sin el header X-Scope-OrgID en cada request, todo se rechaza con 404 no org id. Promtail necesita el tenant_id en su config (como en el comando de arriba), y cualquier datasource de Grafana que apunte a Loki necesita el mismo header agregado a mano en "HTTP Headers" al momento de configurarlo.

### Exponer y conectar

```bash
cat > grafana-ingress.yaml << 'EOF'
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: grafana
  namespace: fury-monitoring
spec:
  entryPoints: [web]
  routes:
    - match: Host(`grafana.fury.local`)
      kind: Rule
      services:
        - name: monitoring-grafana
          port: 80
EOF
```
```bash
kubectl apply -f grafana-ingress.yaml
```

### Primeros dashboards

Adentro de Grafana: Data Sources > Loki
URL `http://loki-gateway.fury-monitoring.svc.cluster.local`
header X-Scope-OrgID: poor-mans-fury

Luego se pueden agregar cuatro dashboards prearmados por ID (Import > 'número'): Kubernetes Cluster Overview (7249) para el estado general del cluster, K8s Namespace Overview (15757) para desglosar por scope, Node Exporter Full (1860) para el hardware del NUC en sí, y Loki Dashboard (13639) para explorar logs sin tener que hacer queries a mano contra la API.

## Dónde quedamos

Una app corriendo en `fury-dev` con sus secrets inyectados por Vault, sus métricas y logs ya aparecen en Grafana sin haber tenido que instrumentar nada a mano para eso.

Lo que queda pendiente es un catalogo, una UI, algo que se parezca a un producto. Ahi entra Backstage como front, capa UI sobre nuestro Fury.