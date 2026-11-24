---
date: 2026-11-23
layout: post
title: "Poor Man's Fury - Extra: sist2"
subtitle: Un buscador para todo lo demás
---

Con la música en [Navidrome]({{< ref "poor-mans-fury-extra-navidrome.md" >}}) y las fotos y videos en [Immich]({{< ref "poor-mans-fury-extra-immich.md" >}}), en `homunculus` quedaba lo más difícil de navegar: `doc/`. 145 GB y 110 mil archivos de 26 años, desde 2000 hasta hoy: trabajos de la facultad, la tesis, backups de máquinas que ya no existen, mails exportados, zips dentro de zips y muchos archivos sueltos.

La primera versión de [Gostalgia]({{< ref "poor-mans-fury-extra-ordenar-el-archivo.md" >}}) ya permitía hacer búsquedas rápidas por API, pero bueno, había mucho que se podía mejorar, especialmente desde la parte web: búsquedas avanzadas más amigables y el indexado del contenido de zips, metadata de archivos, etc.

## La idea

Un buscador para todo lo demás: nombre, contenido de documentos, metadata y lo que hay adentro de los comprimidos, desde el browser. Solo LAN, como venimos trabajando hasta ahora.

> **sist2** ("simple incremental search tool"): escanea una carpeta sin modificarla, extrae texto y metadata (PDF, Office, HTML, EXIF, ID3, y lo que hay adentro de zip/tar/7z/rar) y arma un índice con una interfaz web de búsqueda con filtros. Históricamente usaba Elasticsearch; las versiones nuevas también funcionan con SQLite, que en un NUC con otras cosas corriendo es la diferencia entre precisar 2 GB de RAM y casi nada.

Mismos requisitos que con la música y las fotos: **el archivo no se toca**. `doc/` se monta read-only y el índice vive aparte, en el disco del sistema. Si se pierde, se regenera.

## La prueba

Primero el binario estático (v4.2.3) sobre un solo backup, `doc/backup/backup-20`: 22 segundos, 21.149 documentos contando lo de adentro de los comprimidos, 125 MB de índice. La interfaz convenció.

## El deploy

En el namespace `media`, al lado de Navidrome e Immich. Dos piezas que comparten un volumen con el índice:

- un **CronJob** que los domingos a las 03:00 escanea `doc/` (incremental) y arma el índice de búsqueda;
- un **Deployment** con la web, que lee el SQLite en vivo: los documentos nuevos aparecen sin reiniciarla.

La imagen trae el binario en `/root` con permisos 700, así que no puede correr como `nobody` como Navidrome. Corre como root, pero sin ninguna capability y con `doc/` montado read-only:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
volumeMounts:
  - name: doc
    mountPath: /doc
    readOnly: true
```

La web tiene basic auth (el secret queda fuera de git), sale por Traefik en `files.fury.local` y, para el celular, por un `LoadBalancer` en el puerto 4090: Android e iOS no leen `/etc/hosts`.

## Un Thread

El primer escaneo completo se disparó a mano desde el CronJob, con 3 threads. Al rato:

> **Issue 01**: el procesador estaba casi al 100 %, `sensors` marcaba 96 °C y el contador de throttling del paquete iba por 2.114. Con 2 threads, 95 °C y seguía frenándose.

K3s 1.36 permite cambiar los recursos de un pod en marcha, sin reiniciarlo:

```bash
kubectl -n media patch pod <pod> --subresource resize \
  --patch '{"spec":{"containers":[{"name":"scan","resources":{"limits":{"cpu":"1"}}}]}}'
```

El CronJob quedó con `--threads 1` y `limits.cpu: "1"`. Más lento, pero el primer escaneo se hace una sola vez; después son incrementales de minutos.

Esto fue al día siguiente de cambiar la pasta térmica (ver [el Extra de Immich]({{< ref "poor-mans-fury-extra-immich.md" >}})): evidentemente el NUC aguanta carga parcial, pero con todos los threads al 100 % sostenido llega a su límite. Con 1 thread bajó a 79 °C en 45 segundos, aunque durante el escaneo completo llegó a picos de 94 °C.

## OOMKilled

> **Issue 02**: el primer escaneo murió por memoria dos veces seguidas: `Memory cgroup out of memory: Killed process ... (sist2) anon-rss:2567776kB`, con un límite de 3 GiB. Medido en la prueba, sist2 usaba 450 MB.

sist2 no dice qué archivo está procesando salvo con `--verbose`, y un log dentro del pod se pierde cuando el OOM lo mata. El camino para encontrar a los potenciales culpables:

1. Sacar del índice las rutas que sí se habían indexado (un pod con `sqlite3` montando el volumen).
2. Compararlas con un `find` de `doc/`: faltaban 12.117 archivos.
3. Correr un pod de prueba con `--verbose --list-file` sobre esos faltantes, con el log en un `hostPath` para que sobreviva al OOM.

Aparecieron de a uno:

- un zip con un Thunderbird portable de 2017 y un mbox `All Mail` de 3,8 GB adentro;
- un `tar.gz` de 8 GB con comprimidos adentro: zips de 300 MB y un 7z de 1,7 GB;
- otro zip de Thunderbird, con un mbox de 6,5 GB;
- un `all-mail.mbox` de 970 MB que sist2 parte en mensajes, uno de ellos de 756 MB.

sist2 carga cada entrada de un comprimido entera en memoria, e ignora `--mem-buffer`. El fix fueron dos flags: `--archive shallow` (no entra en comprimidos dentro de comprimidos) y un `--exclude` para los mails, que para buscar no aportan y sí rompen. Con eso, la prueba sobre los 12 mil faltantes pasó en 6 minutos sin superar 1,5 GiB.

Los mails quedan afuera por ahora, para evaluar más a futuro. Así se pudo seguir avanzando en el proceso.

## Las filas a medias

> **Issue 03**: con el escaneo ya completo, armar el índice de búsqueda fallaba: `NOT NULL constraint failed: document_index.name`. Eran las filas que el OOM había dejado a medio escribir (con `json_data` en NULL), y como el escaneo es incremental, las daba por hechas y no las volvía a procesar.

Fix: borrarlas junto con todo lo que colgaba de ellas (30.091 filas, porque un zip a medio procesar arrastra a sus entradas) y volver a escanear:

```sql
PRAGMA foreign_keys=OFF;
CREATE TEMP TABLE bad AS WITH RECURSIVE t(id) AS (
  SELECT id FROM document WHERE json_data IS NULL AND parent IS NULL
  UNION SELECT d.id FROM document d JOIN t ON d.parent = t.id)
SELECT id FROM t;
DELETE FROM document WHERE id IN (SELECT id FROM bad) OR json_data IS NULL;
```

## La web

La web no puede arrancar sin índice, así que tenía un `initContainer` que esperaba a que existiera `search.sist2`.

> **Issue 04**: la web arrancó a mitad del índice y falló con `database is locked`. `sqlite-index` crea el archivo apenas empieza, así que "existe" no quería decir "terminó". Fix: el CronJob hace `touch /data/.indexed` al final, y el `initContainer` espera ese archivo.

```yaml
initContainers:
  - name: wait-for-index
    command: ["/bin/sh", "-c", "until [ -e /data/.indexed ]; do sleep 30; done"]
```

Quedaba una duda: ¿y los domingos, cuando el CronJob reindexa con la web ya levantada? Testeado antes del primer domingo con un backup del índice por las dudas: un escaneo incremental más el reindexado, 4 minutos y medio, con la web respondiendo cada 15 segundos. Sin errores, y el CPU no pasó de 88 °C. SQLite aguanta un lector y un escritor mientras el lector no tenga una transacción abierta todo el tiempo.

## Dónde quedamos

1.249.499 documentos (contando las entradas de comprimidos) en 213 tipos MIME, un índice de 7,1 GB más 2,2 GB del de búsqueda, y un escaneo semanal de 5 minutos: el domingo 22 corrió solo, con la web levantada y sin un error.

![búsqueda de "gato" en sist2](../img/2026-11-23-poor-mans-fury-extra-sist2/sist2.png)

Como en los últimos posts, mucho crédito a la IA en todo este trabajo, especialmente al momento de diagnosticar un issue, donde históricamente podían ser horas o días investigando: trabajo ingrato y por momentos muy frustrante, buscando respuestas en Stack Overflow, tratando de aislar un problema para reproducirlo y tener más información, etc.

Igual que con Navidrome e Immich, sist2 no usa casi nada de lo que hace a Poor Man's Fury una plataforma de desarrollo: ni Backstage, ni `pmf`, ni Vault. Un manifiesto, un volumen y un CronJob. La diferencia es que acá la parte difícil no fue el deploy sino el archivo: 26 años de cosas guardadas, muchas de ellas "por las dudas", y hubo que negociar con los propios límites de un NUC de 2017. De cualquier forma, se logró el objetivo y se cerró otra deuda histórica personal. Trabajar con una IA termina siendo muy ágil y satisfactorio, en gran parte, creo yo, cuando se tiene claro el destino al que se quiere llegar y al menos una idea de qué está sucediendo en cada paso.
