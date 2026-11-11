---
date: 2026-11-11
layout: post
title: "Poor Man's Fury - Extra: Navidrome"
subtitle: Plataforma de desarrollo con un server de música
---

[La Parte 5]({{< ref "poor-mans-fury-part-05.md" >}}) cerró el roadmap: las 8 fases completas y un cluster que corre solo. Este post queda fuera de la serie: es el primer uso de la plataforma para algo que no es la plataforma.

En un disco de backup personal hay 65 GB de música que vengo juntando desde mi época en la facultad: CDs propios ripeados, carpetas que pasaron de disco en disco, flacs y, más tarde, algún mp3 bajado de YouTube. Música "de mi época": la onda que traigo ya no es onda.

![la onda que traigo ya no es onda](../img/2026-11-11-poor-mans-fury-extra-navidrome/onda.jpeg)

Estaba ahí y se usaba, porque me gusta mucho escuchar música en sesiones de trabajo por ejemplo, cuando no estoy en alguna reunión. De hecho tengo un DAC externo y unos auriculares "decentes" (eso es otra historia, pero sí, vengo mejorando el setup). El tema es que sabía que se podía hacer algo más completo, y ahora tenía un mini cluster Kubernetes donde hacerlo.

## La idea

Servir esa biblioteca desde rathma, con interfaz web, apps en el celular y un usuario por persona de la casa. Solo LAN, nada expuesto a internet, por decisión.

> **Navidrome**: server de música open source, escrito en Go, compatible con la API de Subsonic. Eso último es lo que importa: cualquier app que hable Subsonic (hay varias) funciona conectándose a Navidrome sin configuración especial. Un solo binario y una base SQLite: ideal también para un NUC de casi 10 años que ya tiene otras cosas corriendo.

Un requisito no negociable: **Navidrome no modifica la música original**. Es una copia de respaldo, no una carpeta de trabajo: nada de moverla, "limpiarla" ni editarle los tags.

## Los tags

Un script de solo lectura con [mutagen](https://mutagen.readthedocs.io/) sobre los 3.138 temas mostró lo que ya se veía con `cmus`, esperable después de años de ripear y arreglar tags con distintos programas:

| Variantes | Temas |
|---|---|
| `AC/DC` · `AC_DC` · `AC-DC` · `ACDC` | 52 · 19 · 16 · 10 |
| `Hermetica` · `Hermética` | 93 · 51 |
| `Guns N' Roses` · `Guns n' Roses` | 70 · 30 |

En la época en que usaba Windows usé durante años una versión portable de [Mp3Tag](https://www.mp3tag.de/en/) para dejar todo completo y consistente. De nuevo, "I was there, Gandalf, I was there 3000 years ago": horas de trabajo manual que valían la pena por el placer de ver cada tema completo y con su tapa, pero que en un momento fueron superadas por la fiaca.

Veinticinco artistas escritos de varias formas, ~230 temas sin tags y ~1.100 sin carátula. La tentación era limpiar todo antes de empezar. Con IA ese trabajo de repente se vuelve más simple: la barrera bajó acá también, así que vamos por todo. Pero primero, Navidrome andando sobre la música tal como estaba; la limpieza vino después, y aparte.

## Read-only, dos veces

Una prueba descartable con Docker confirmó que la interfaz convencía y que indexar los 65 GB tardaba menos de 12 segundos. En el cluster quedó un Deployment en un namespace `media`, aparte de las apps de Fury, con un volumen `local-path` para la base y la música montada desde el disco:

```yaml
spec:
  strategy:
    type: Recreate              # SQLite en un volumen RWO: nunca dos pods a la vez
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534        # nobody
      containers:
        - name: navidrome
          volumeMounts:
            - name: music
              mountPath: /music
              readOnly: true
      volumes:
        - name: music
          hostPath:
            path: /run/media/mario/homunculus/music
            type: Directory
```

Dos capas de protección: el mount es read-only y eso lo aplica el kernel (`touch /music/x` → `Read-only file system`), y además el proceso corre como `nobody`, que no es dueño de nada. Si fallara una, queda la otra.

> **`type: Directory`**: sin eso, si el disco no está montado el container arranca igual, con un directorio vacío, y Navidrome ve una biblioteca vacía, como si la música hubiera desaparecido. Con `type: Directory` el pod no arranca hasta que la ruta existe. Prefiero un server de música caído a uno que funciona sin música.

Los logs aparecieron solos en Loki, sin configurar nada, igual que con `cookbook` en la Parte 5.

![interfaz web de Navidrome](../img/2026-11-11-poor-mans-fury-extra-navidrome/navidrome.png)

## Tres falsos "no anda"

Navidrome respondía desde rathma pero "no andaba" desde otros dispositivos. Tres veces, y ninguna era culpa del server.

> **Issue 01**: timeout desde la notebook. Era un typo en la IP [facepalm]. Lo dejo porque es uno de los errores que más tiempo hace perder, justamente por obvio pero repetido en mi caso.

> **Issue 02**: desde la notebook `curl` llegaba pero los browsers no. En macOS 15 cada app necesita permiso para acceder a la red local (Privacidad y seguridad → Red local). Sin eso, el browser falla como si el server no existiera. No lo sabía, así que también estuve martillando unos minutos.

> **Issue 03**: en Android, el navegador entraba pero la app Substreamer no: `ConnectException`. Otra vez, permisos de red de la app (en Android 17, el permiso *Nearby devices*). Las opciones que uno toca primero ("legacy login", certificados autofirmados desde la misma pantalla de login de la app) no tienen nada que ver: la conexión falla antes de autenticar.

Los logs de Navidrome no recibían nada durante las pruebas. Si al server no le llega nada, el corte estaba del lado del cliente.

## El firewall que no decidía nada

Revisando el firewall para descartarlo apareció algo interesante: la regla de `ufw` que abría el puerto **no es lo que deja pasar el tráfico**.

Un Service `LoadBalancer` en K3s entra por DNAT y sigue por la cadena `FORWARD`, no por `INPUT`, que es donde actúan las reglas de ufw. Y en `FORWARD` las reglas de kube-proxy y flannel aceptan el tráfico antes de que ufw opine. Con la regla o sin ella, el puerto está abierto. Lo mismo vale para las reglas que abrí en [la Parte 1]({{< ref "poor-mans-fury-part-01.md" >}}) para Traefik.

![recorrido de un paquete: ufw decide en INPUT, pero el tráfico de un Service LoadBalancer va por DNAT y FORWARD](../img/2026-11-11-poor-mans-fury-extra-navidrome/ufw-vs-servicelb.png)

En casa, detrás del router, no es un riesgo. Pero conviene saberlo antes de que importe: en Kubernetes, ufw no es el firewall de los Services. Si hace falta acotar quién llega, va en el propio Service (`loadBalancerSourceRanges`).

## Los tags, sin tocar el audio

Con Navidrome andando, la limpieza vino aparte y con la misma regla: la música no se toca, solo los tags. Dos scripts con mutagen. Uno de solo lectura arma el plan: por cada carpeta de artista, el nombre reconocido (`AC_DC` → `AC/DC`), las colaboraciones a `albumartist` y los tags con artista y disco pegados (`Horcas/Vence`), separados. El otro lo aplica sobre una copia temporal, verifica que el audio sea idéntico, conserva la fecha de modificación y recién ahí reemplaza el original de forma atómica.

Resultado: 464 archivos retagueados, 0 con audio distinto. En el medio, la reorganización del disco sumó música que estaba repartida en otros backups: hoy son 6.192 temas y 78 GB, y Navidrome los muestra unificados.

> **Issue 04**: la verificación con `ffmpeg -c copy -f md5` daba "audio changed" en 11 MP3 sin que el audio cambiara. El hash del stream copiado depende del tag ID3, justo lo que se estaba cambiando. Fix: comparar el MD5 del audio decodificado (PCM) más el SHA-256 de las tramas entre los tags.

> **Issue 05**: el archivo temporal terminaba en `.tagtmp`, así que la primera pasada no entró en la rama de MP3 y guardó 295 tags como ID3v2.4 en vez de conservar la versión original. cmus y Navidrome leen 2.4 sin problema, pero la idea era no cambiar nada que no hiciera falta.

> **Issue 06**: cmus seguía mostrando `AC_DC` después de `:update-cache`. cmus solo relee un archivo si cambió la fecha de modificación, y el script la conserva a propósito. Fix: cerrar cmus y borrar `~/.config/cmus/cache`.

> El 4 y 5 en particular, imposible para mí sin una IA asistiendo en el proceso y levantando los puntos. Hubiesen sido horas de troubleshooting o directamente hubiesen pasado desapercibidos e ignorados esos issues.

## Clientes

- **Substreamer** (Android + iOS): la que usa la familia, la misma app en todos los teléfonos.
- **Tempus** (Android, open source), **Amperfy** (iOS) y **Feishin** (escritorio) como alternativas (ninguna de estas las probé al momento). O directamente la web.

Un usuario por persona, sin permisos de admin, y los flac salen tal cual, sin transcodear.

![Substreamer en el celular, pantalla de reproducción](../img/2026-11-11-poor-mans-fury-extra-navidrome/substreamer.png)

## Dónde quedamos

Navidrome no usó casi nada de lo que hace a Poor Man's Fury una plataforma "de desarrollo": ni Backstage, ni `pmf`, ni CI, ni Vault. Usó la base: K3s, Traefik, un volumen y logs que aparecen solos. Y con eso alcanzó para que sumar un servicio a la casa sea un manifiesto y un `kubectl apply`.

Lo que sigue: Jellyfin para los videos e Immich para las fotos.