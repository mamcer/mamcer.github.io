---
date: 2026-11-22
layout: post
title: "Poor Man's Fury - Extra: Immich"
subtitle: 143 mil fotos y videos, un NUC de 2017 y un cooler parado
---

[El Extra anterior]({{< ref "poor-mans-fury-extra-ordenar-el-archivo.md" >}}) dejó el archivo ordenado: una sola copia de cada cosa, con sus fechas. Lo siguiente era poder mirar las fotos y los videos de la familia, y para eso, Immich fue el elegido. Gostalgia cumplió su ciclo: decidido, se le daba un cierre al proyecto. Lo que seguía era liberar esa silla y buscar opciones maduras, iteradas para hacer lo mismo.

## La idea

Navegar todo el archivo de fotos y videos desde la web y (opcionalmente) el celular: línea de tiempo, mapa, caras y búsqueda por contenido. Accesible inicialmente solo LAN, nada expuesto a internet. Igual que con la música.

> **Immich**: alternativa self-hosted a Google Photos, open source. Server en Node, una parte de machine learning en Python (caras, búsqueda por contenido con CLIP, OCR), Postgres con una extensión vectorial y Valkey para las colas de trabajo. Es bastante más pesado que Navidrome: cuatro pods en vez de uno.

Los requisitos, los mismos que con la música: **las fotos no se mueven ni se modifican**, y todo lo que genera Immich (base, miniaturas, videos convertidos) vive aparte y se puede regenerar. Immich lo resuelve con las *External Libraries*: indexa una carpeta tal cual, sin importar ni copiar nada. Con el mount read-only, aunque quisiera escribir no podría.

## El deploy

Sin Helm: una traducción 1:1 del `docker-compose.yml` oficial a manifiestos, en el mismo namespace `media` que Navidrome. Postgres, Valkey, machine learning y el server, cada uno con su Deployment. Las fotos entran igual que la música:

```yaml
volumeMounts:
  - name: picture
    mountPath: /mnt/archive/picture
    readOnly: true
volumes:
  - name: picture
    hostPath:
      path: /run/media/mario/homunculus/picture
      type: Directory
```

Postgres es obligatorio: Immich usa la extensión vectorial para la búsqueda por contenido, así que no sirve el MySQL que ya corría en rathma. Hasta ahí, nada raro. Pero no arrancó...

> **Issue 01**: Postgres nunca terminaba de iniciar. El proceso que pasa de root al usuario `postgres` (`gosu`) quedaba al 100 % de CPU, sin hacer nada. Corriendo el pod directamente como el usuario de Postgres avanzó un paso más y falló en otro lado: `could not signal for checkpoint: Permission denied`. Un proceso no podía mandarle una señal a otro **dentro del mismo contenedor**.

El kernel:

```
apparmor="DENIED" operation="signal" profile="cri-containerd.apparmor.d" ... peer="cri-containerd.apparmor.d//&unconfined"
```

Buscando, aparentemente es un bug conocido de Ubuntu 26.04 con containerd. Ubuntu trae activada una restricción nueva de AppArmor que hace que los procesos del contenedor queden con una etiqueta "apilada" (`//&unconfined`), y el perfil de containerd solo permite señales entre procesos con la etiqueta simple. `gosu` giraba por lo mismo: Go usa señales internamente.

Detalle interesante: **no era solo Immich**. Revisando el log del kernel había ~244.000 bloqueos de Vault, y también de Navidrome, Prometheus y la API de Gostalgia. Todo el cluster venía chocando contra esto sin que lo notara porque esos servicios funcionaban igual.

Había tres salidas: desactivar AppArmor en los pods de Immich, bajar la restricción en todo el host, o corregir el perfil. La primera y la segunda, descartadas. Quedó la tercera: el perfil de containerd, con la etiqueta apilada agregada en las reglas de `signal` y `ptrace`, cargado desde `/etc/apparmor.d` para que esté antes de que arranque k3s.

```bash
sudo install -m 0644 apparmor/cri-containerd.apparmor.d /etc/apparmor.d/
sudo apparmor_parser -r -W /etc/apparmor.d/cri-containerd.apparmor.d
```

253 bloqueos en los 8 minutos anteriores, 0 desde la recarga. Y se aplicó en caliente, sin reiniciar ningún pod (Vault ni se enteró, por ejemplo).

> Crédito: mucha ayuda de Claude Code en todo el proceso, pero muy especialmente en esta depuración y luego fix de este issue.

## La primera pasada

143.482 fotos y videos. Immich los procesa en etapas: metadatos, miniaturas, búsqueda por contenido, caras, OCR. En el NUC fueron dos días y medio con el CPU al 100 %, temperatura al límite y un ventilador externo dando soporte.

> **Issue 02**: con las miniaturas al 94 %, el server entró en crash loop: 28 reinicios. Al arrancar, Immich arma el índice vectorial de la búsqueda, y Postgres tenía un límite de 1 GB de memoria. El build lo pasaba, el kernel mataba a Postgres, el server perdía la conexión, salía y volvía a empezar. Fix: 4 GB para Postgres. Con eso el índice se armó en un minuto.

Cada build abortado había dejado sus archivos en disco. Postgres no limpia los archivos de un índice cuya creación muere a la mitad. **47 GB de archivos huérfanos**, contra ~4 GB de datos reales, y el disco de rathma pasó de 85 a 31 GB libres. Fix: dump completo de la base y restore sobre un volumen nuevo, comparando la cantidad de filas de las 85 tablas antes y después. Pasó de 52 GB a 7,7 GB.

Al final de la pasada:

| | Cantidad |
|---|---|
| Fotos y videos | 143.482 |
| Miniaturas | 140.429 (los 3.035 que faltan son clips ocultos de Live Photos, más 18 archivos rotos de origen) |
| Caras | 140.383 fotos analizadas, 9.224 personas detectadas |
| Videos convertidos | 1.323 |

![búsqueda por contenido en Immich: "gato negro"](../img/2026-11-22-poor-mans-fury-extra-immich/busqueda-por-contenido.png)

## El NUC también pidió mantenimiento

Casi tres días con el CPU al 100 % dejan ver cosas. Durante la primera pasada el NUC estuvo entre 93 y 97 °C. El día que finalizó todo el escaneo y se estaban revisando resultados, **el cooler dejó de girar**: la protección del procesador se activó y todo funcionaba súper lento. Así me di cuenta, y también porque la carcasa y todo el NUC volaban de temperatura.

Por ejemplo, jamás le había cambiado la pasta térmica. Tuve un intento serio hace unos años, pero llegué al paso de tener que desconectar cables muy pequeños y, por miedo a romper algo, me quedé ahí. Jamás pude llegar al cooler. Ahora, pocas opciones: o lo logramos o de todas formas ya no sirve más.

Desarme completo: el cooler tenía mucha mugre adentro y la pasta térmica con 9 años, era yeso. Limpieza, pasta nueva y rearmado.

![el disipador del NUC, con la pasta térmica original de 9 años](../img/2026-11-22-poor-mans-fury-extra-immich/nuc-desarmado.jpg)

El NUC no le muestra a Linux las RPM del cooler (solo se ven en el BIOS), así que la verificación fue indirecta: una prueba de carga con los 4 hilos al 100 %.

| Momento | CPU |
|---|---|
| Reposo | 54 °C |
| Carga 100 %, 60 s | 86 °C |
| Carga 100 %, 120 s | 90 °C |
| 5 s después de cortar | 68 °C |

Sin cooler, ese procesador llega a 100 °C en segundos, así que estamos bien por ahora: nueva vida.

## Privacidad: qué sale de la red

Una de las razones para hacer esto en casa es que las fotos no salgan. Lo revisé con Claude Code:

- Caras, búsqueda por contenido y OCR corren en el pod de machine learning, dentro del cluster. Ninguna foto ni resultado sale.
- Lo único que bajó de internet fueron los modelos, la primera vez.
- Los tiles del mapa los pide el navegador a un servidor de Immich (ve la IP y la zona que estás mirando, no las fotos). Y el chequeo de versión nueva consulta GitHub.

![el mapa de Immich, con las fotos agrupadas por lugar](../img/2026-11-22-poor-mans-fury-extra-immich/mapa.png)

## Clientes

La app oficial de Immich, en Android e iOS, apuntando a `http://192.168.100.100:2283`. Android e iOS no leen `/etc/hosts`, así que en el celular es por IP y puerto, como con Navidrome.

## Dónde quedamos

Immich usó la misma base que Navidrome (K3s, un namespace, volúmenes `hostPath` read-only) y fue bastante más exigente: sacó un bug de AppArmor que afectaba a todo el cluster, y llevó al NUC a su límite térmico. Algunos de estos problemas venían de antes; Immich solo los hizo visibles.

Por el momento, súper contento con el resultado. Este orden de fotos y videos, y tenerlos disponibles dentro de la LAN para toda la familia, era una deuda personal. Gostalgia me dejó el mayor aprendizaje en sus primeras etapas; después era más un freno que otra cosa.

Lo que sigue: sist2, un buscador para todo lo demás del archivo (documentos, comprimidos, backups viejos).
