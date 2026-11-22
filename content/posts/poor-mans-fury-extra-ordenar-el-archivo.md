---
date: 2026-11-21
layout: post
title: "Poor Man's Fury - Extra: ordenar el archivo"
subtitle: 25 años de copias de copias, una sola copia de cada cosa
---

[El Extra de Navidrome]({{< ref "poor-mans-fury-extra-navidrome.md" >}}) dejó la música andando. Lo siguiente eran las fotos, pero antes había que resolver otra cosa: qué fotos, cuáles videos y en qué estado.

¿Qué había antes? Muchos proyectos e ideas, algunas materializaciones, pero faltaba. Desde que tuve acceso a una computadora me recuerdo haciendo pequeños programas en distintos lenguajes de programación a través del tiempo, muchos de ellos sin una mayor utilidad práctica relevante más que testear una idea o capacidad, me quedaba con el aprendizaje. Otros un poco más ambiciosos, que buscaban poner a prueba todo lo que se sabía al momento, tipo en Matrix cuando Neo le dice "I know kung fu" a Morpheus y él le responde "Show me" y básicamente lo pone a prueba.

![Morpheus: show me](../img/2026-11-21-poor-mans-fury-extra-ordenar-el-archivo/show-me.png)

*The Matrix* (1999), [la escena en YouTube](https://www.youtube.com/watch?v=B06J_9-XnxA).

En ese contexto 'Gostalgia' venía a indexar, detectar y descartar gigas de información duplicada recopilada durante más de 20 años, y permitiría luego búsquedas rápidas, previsualización de imágenes, texto, etc. Pero por sobre todo venía a poner a prueba todo lo que se había aprendido al momento sobre Golang y cómo estructurar, iterar y mantener un proyecto un poco más serio.

Long story short: tuvo varios releases, llegué a escanear todos los archivos, una web muy básica, generar un 'maestro' sin archivos repetidos. Con IA se puede iterar código muy rápido, pero también, como el objetivo nunca fue generar un producto en sí, se perdió un poco la motivación de llevar esa versión 0.9 a algo más completo.

El disco del archivo, `homunculus`, es un SSD de 4 TB con todo lo que fui juntando desde la facultad: fotos de cámaras que ya no existen, volcados de celulares, CDs de backup, discos de la familia y años de "por las dudas copio esto acá". Gostalgia por definición copia los archivos a un maestro, manteniendo links a repetidos en una base de datos pero con una única copia física. En ese contexto, las fechas de la copia son del momento en que se copió (2025, último scan completo) manteniendo en la base de datos la fecha original.

La decisión fue dar por finalizado Gostalgia y rehacer el análisis.

Punto de partida: 1,8 TB, ~730.000 archivos.

## La regla

Antes de mover nada, una regla que no se negocia: **nada se borra**. Lo repetido va a una carpeta `_cuarentena`, y solo si está *probado* que existe una copia idéntica fuera de ella. Probado es SHA-256 de los dos lados, recalculado en el momento, no el de una lista vieja. La cuarentena se borra a los 30 días, después de sincronizar los backups.

La segunda regla era para la IA: durante el relevamiento se miran **nombres, tamaños y fechas**, nunca el contenido de los archivos. Es mi archivo personal, y para decidir qué está repetido no hace falta abrirlo. La única excepción fueron las fotos de costado (más abajo).

## Primero, el manifiesto

Lo primero no fue deduplicar sino tener una foto del estado: un archivo en formato `sha256sum` con el hash de cada archivo, rutas relativas a la raíz del disco (así sirve también contra los backups), y un snapshot de ruta, tamaño y fecha para poder deshacer cualquier cambio.

```bash
python3 manifest.py /run/media/mario/homunculus nostalgia/home ~/dedup/home-2026-11-07
# verificar en cualquier momento, en este disco o en una copia:
sha256sum -c --quiet ~/dedup/home-2026-11-07.SHA256SUMS
```

251.548 archivos en 41 minutos. Con el disco siendo un SSD, hashear 1,2 TB era cuestión de CPU más que de disco.

## Recuperar las fechas

Gostalgia no había borrado los originales: seguían en el disco, a propósito, y con su fecha intacta. Ya que estaban ahí, se decidió releerlos en lugar de ir a la base de datos de Gostalgia. Con [fclones](https://github.com/pkolaczk/fclones) (un binario estático, sin instalar nada) se agruparon los archivos idénticos entre la copia y los originales:

```bash
fclones group nostalgia/home null/cd -. -A --cache -f json -o ~/dedup/null-cd.json
# Found 260583 (269.6 GB) redundant files      15 minutos
```

> **fclones**: busca duplicados por etapas: primero agrupa por tamaño, después por los primeros bytes, y recién al final hashea completo lo que sigue coincidiendo. Por eso tardó 15 minutos y no las dos horas que hubiera llevado hashear todo.

Para cada archivo de la copia, la fecha más vieja *plausible* entre sus originales (nada antes de 1985 ni en el futuro: un reloj de cámara reseteado marca 1980). Un plan en CSV, revisado antes de aplicar, y un script que aplica exactamente ese CSV y deja un log para deshacer. **103.231 archivos recuperaron su fecha** en 2,7 segundos.

Dos controles: la música comercial quedó afuera (su fecha solo dice cuándo fue ripeada, no aporta mucho) pero sí se restauraron las grabaciones personales, como audios de WhatsApp o cassettes digitalizados. Y de 5.929 archivos con la fecha en el nombre (`IMG-20200809-…`), el 94% quedó a dos días o menos de esa fecha.

## La cuarentena

Con las fechas a salvo, los originales ya no aportaban nada. Para moverlos no usé `fclones move`: su hash rápido no es criptográfico y la regla pedía SHA-256. Un script propio con tres pasos: `validate` (solo lectura, recalcula el hash del original *y* el de su par), `move` (vuelve a chequear tamaño y fecha justo antes de mover, nunca pisa nada) y `undo`.

```
{"sources": 260752, "move": 259328, "no_move": 1424, "reasons": {"empty file": 1424}}
```

259.328 archivos, 269 GB, todos con su par verificado. Los 1.424 que no se movieron eran todos archivos vacíos. Mover tardó 21 segundos: en el mismo filesystem, mover es renombrar.

## Cuando bytes iguales no alcanza

Algo interesante sucedió con los volcados de celulares, donde "idéntico" resultó no ser suficiente:

- **Ediciones del iPhone**: un `.AAE` (la edición de una foto) y el `.MOV` de una Live Photo se asocian a su foto **por nombre**. Dos `.AAE` con los mismos bytes pero de fotos distintas no son duplicados: borrar uno rompe la edición de la otra. Regla nueva: solo se deduplica si además coincide el nombre.
- **La misma foto en menor calidad**: la copia que pasó por WhatsApp, la versión "baja" de un fotógrafo. No son bytes iguales, pero sí la misma captura. Se detectan por la fecha EXIF más una huella visual (dHash de 8×8 sobre una miniatura), y se queda la de más resolución.
- **Videos "convertidos"**: un backup viejo tenía ~346 GB de videos originales (`.mov`, `.avi`) de los que el archivo solo tenía la versión `.mp4`. Antes de rescatarlos, `ffprobe`: los `.mov` se habían pasado a `.mp4` sin recodificar (mismo stream, mismo bitrate) y el resto se había recodificado con igual o más bitrate. Los originales no aportaban calidad.

## Fotos de costado

Las cámaras viejas (un Nokia, una BlackBerry, una compacta) no tenían sensor de orientación: cientos de fotos de costado. Girarlas recomprimiendo el JPEG pierde calidad, así que la corrección es **solo el tag EXIF Orientation**, y el script verifica que los píxeles decodificados queden idénticos antes y después.

Encontrarlas fue otro tema. Primera pasada: un detector de caras (YuNet, de OpenCV) sobre cada foto girada; si solo encuentra caras a 90°, la foto está de costado. 1.005 candidatas, revisadas a mano en hojas de contacto: 621 eran ciertas. El 40% restante eran bebés y gente acostada, donde la foto está bien.

Después se midió cuánto se le escapaba: de 764 fotos que se sabía que estaban de costado, las caras encontraron 237. **Cobertura del 31%**. Segunda pasada sumando un detector de objetos (YOLOX): 547 más. Y otras 527 salieron de pares que había en el archivo: el original de cámara de costado y una copia que alguien giró y volvió a guardar. Se quedó el original, con la orientación de la copia.

> Acá sí hubo que mirar el contenido, pero todo local: los detectores son modelos de OpenCV que corren en el NUC, nada sale a internet. La IA escribió los scripts y ordenó las candidatas por probabilidad; las hojas de contacto las revisé yo manualmente, y tomé la decisión sobre cada foto con dudas. Mismo criterio que la cuarentena: la herramienta acelera, no decide.

## Issues

> **Issue 01**: a la mañana del primer día el NUC se congeló y hubo que reiniciarlo. Sospecha obvia: el trabajo con el disco. No fue: `gnome-shell` pasó de 239 MB a 7,7 GB en una hora. Y el OOM killer nunca lo mató: elige por `oom_score_adj`, y los pods `BestEffort` de K3s tienen 1000 contra 100 de `gnome-shell`. Mató en loop a Prometheus, Loki y Grafana, que K3s volvía a levantar, mientras el culpable seguía creciendo. Después del reinicio, las fechas restauradas se verificaron contra el snapshot: 0 cambios.

> **Issue 02**: un `awk` sobre el log de movimientos para deshacer cinco archivos daba `missing source` en todos. El CSV tenía finales de línea CRLF y `awk` dejaba un `\r` al final de cada ruta. Fix: leerlo con el módulo `csv` de Python. Clásico, pero con un log de deshacer no hay margen.

> **Issue 03**: el primer cruce de "misma foto, menor calidad" por fecha EXIF daba 7.726 copias. 1.545 eran fotos distintas: relojes de cámara reseteados y ráfagas sin fracciones de segundo comparten fecha y hora exacta. Fix: exigir además la huella visual y excluir números de archivo consecutivos.

> **Issue 04**: la IA estimó la segunda pasada de fotos de costado en 3 horas, sin medir. Eran ~1,2 segundos por foto: 20 horas. Fix: ordenar las carpetas por la proporción de fotos de costado ya encontradas en cada una, empezar por las cámaras sin sensor y cortar cuando el rinde baja. Revisadas 20.943 de 65.075, todas las de riesgo.

Mucho soporte de IA en todo el proceso pero especialmente en los issues y sus fixes.

## El cierre

La verificación final fue `sha256sum -c` del manifiesto completo contra el disco, de noche y con prioridad baja:

```bash
nice -n 10 ionice -c3 sha256sum -c --quiet ~/dedup/archivo-baseline.SHA256SUMS
echo $?   # 0
```

Exit 0 sobre 276.153 archivos. Ese manifiesto es ahora el baseline contra el que se verifican los backups. Y de paso, primer `smartctl` de los tres discos: los tres `PASSED`.

| | Antes | Después |
|---|---|---|
| Archivos | ~730.000 | 276.153 (+ cuarentena) |
| Copias de cada cosa | 2 o 3, más volcados repetidos | 1 (salvo ~1 GB de backups viejos que se dejaron a propósito) |
| Fechas | casi todo "2025" | 103.231 recuperadas, más ~950 fotos fechadas desde su nombre |
| Fotos de costado corregidas | — | 1.695, solo con el tag EXIF |
| Estructura | `nostalgia/home`, `null/cd`, `dev`… | `picture/ video/ music/ doc/ iso/` |

## Dónde quedamos

Más de una vez en mi historia le dediqué varias horas a ordenar archivos, rotar fotos, eliminar duplicados, etc. Pero mientras con mucho esfuerzo en tiempo se avanzaba en completar lo pendiente, nuevos archivos se sumaban de a gigas. Con Gostalgia ese trabajo se automatizó y se pudo tener una primera versión, un primer maestro. Con IA se pudo ir mucho más allá: ayudó a ordenar las tareas, a decidir qué ejecutar y a comparar opciones, buscando redundancia a otro nivel, ayudando a identificar archivos que aunque no duplicados no tenía sentido conservar, corrigiendo la orientación de las fotos, etc etc.

Nada de esto usó la plataforma: fueron scripts de Python, `sha256sum`, fclones y mucha revisión. Pero era la condición para lo que sí la usa: Immich, con `picture/` y `video/` montadas read-only, igual que la música en Navidrome. Eso queda para el próximo Extra.
