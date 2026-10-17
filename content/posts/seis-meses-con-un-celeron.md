---
date: 2026-10-17
layout: post
title: Seis meses con un Celeron
subtitle: Celeron N4100, 4 GB de RAM, 64 GB de disco. Disciplina, foco y mucha terminal
---

Allá por 1999, con mucho esfuerzo de mi madre, pude acceder a mi primera computadora personal. Una Celeron de 333 MHz con 32 MB de RAM y un disco de 4.2 GB de 5400 RPM. Windows 98 como sistema operativo. Era austera en su momento, pero ya comenzando el segundo año de Ingeniería de Sistemas era muchísimo. Hasta entonces, programas en papel y/o el laboratorio de la facultad; con este equipo se puede decir que aprendí a programar en C++: incontables pequeños programas, horas y horas probando, leyendo, configurando algo que no andaba, buscando entender cómo funcionaba lo que había adentro.

Veintisiete años después, por nostalgia más que por cualquier otra cosa, decidí volver a trabajar con un Celeron.

En marzo del año pasado [escribí]({{< ref "development-on-an-educational-laptop.md" >}}) sobre una Latitude 3190 reacondicionada, muy parecida a las netbooks del programa educativo: Celeron N4100, 4 GB de RAM, 64 GB de disco. En ese momento la idea era entender si se podía aprender a programar con un equipo así. La respuesta fue que sí, con matices.

En abril de este año tomé una decisión deliberada: convertir esa misma Latitude 3190, con Debian 13, en mi equipo principal para todo lo que hago fuera del trabajo diario, y pasar el NUC que usaba hasta entonces a ser un home server. Deliberada porque la idea era justamente forzarme a trabajar dentro de esos límites como en los viejos tiempos. Seis meses después siento que ese tiempo se está terminando. Antes de que termine, quería dejar por escrito aquí lo que me dejó.

![La Latitude 3190 cerrada al costado del monitor, conectada como equipo principal](../img/2026-10-11-seis-meses-con-un-celeron/01-setup.jpg)

## Lo que hice con esto

Con esta máquina, en estos seis meses:

- Armé la serie [Poor Man's Fury]({{< ref "poor-mans-fury-part-01.md" >}}): cinco partes reconstruyendo una plataforma de desarrollo interna con herramientas open source sobre un servidor casero (el NUC).
- Escribí [Gostalgia](https://github.com/mamcer-labs/gostalgia) (en realidad lo reescribí, con mucha ayuda de Claude Code), un motor de indexación y organización de archivos personales en Go.
- Colaboré en dos proyectos de mi hija: The Weekly Loop, un sistema de curaduría de noticias para las pantallas de los espacios comunes, y Bibliotech, un sistema de catálogo y préstamos para bibliotecas escolares sin soporte técnico.
- Publiqué varios posts en este blog.
- Y una cantidad de notas, borradores y bitácoras que no se ven, pero que son la mitad del trabajo.

Nada de eso necesitó más que lo que hay acá. Bueno, casi nada.

## Ser consciente del hardware

Con 4 GB de RAM y 64 GB de disco no hay forma de olvidarse de que la computadora existe. Cada decisión tiene un costo que se siente en el momento.

No se pueden tener 15 pestañas abiertas (diría que ni 3). Un navegador con muy pocas pestañas o VS Code (es un fuerte 'o'), una terminal, y lo que estés haciendo en ese momento. Si abrís algo más, algo se tiene que cerrar.

Al principio eso molesta, frustra. Después se transforma en otra cosa: en una forma de trabajar, en foco.

En cualquier equipo más o menos moderno todo parece gratis hasta que deja de serlo. Abrimos, acumulamos, dejamos corriendo. Los recursos parecen infinitos o al menos eso nos hace creer el sistema operativo. Acá no. Acá cada proceso tiene que justificar su lugar.

### El disco

Si la memoria es la limitación que más se nota, el disco es la que más pesa.

Es un SSD SATA de 64 GB, de esos genéricos que vienen de fábrica, de un modelo que ni siquiera figura en la base de datos de `smartctl`. En lectura secuencial, con el sistema tranquilo, `hdparm -t` marca entre 120 y 150 MB/s, en el rango de un disco mecánico de notebook.

La buena noticia: está sano. SMART sin errores, temperatura normal. Es, simplemente, un disco de entrada, muy justo en capacidad y en velocidad.

```
$ sudo hdparm -t /dev/sda
 Timing buffered disk reads:  38 MB in  4.74 seconds =   8.02 MB/sec   # sistema cargado
 Timing buffered disk reads: 458 MB in  3.00 seconds = 152.66 MB/sec   # sistema tranquilo
```

Ese primer número es el que importa: con el sistema cargado, la lectura cae a 8 MB/s, casi veinte veces menos.

Y es el verdadero cuello de botella. Cada repositorio clonado, cada imagen de Docker, cada dependencia instalada se piensa dos veces. En 1999 eran 4.2 GB a 5400 RPM y a veces había que elegir qué instalar. Veintisiete años después son 64, y la sensación es sorprendentemente parecida.

### Cómo la configuré

Debian 13 con MATE anda bien de fábrica, pero con 4 GB cada megabyte cuenta. Algunos ajustes que hice:

- **zram.** Swap comprimido en RAM con `zram-tools` (`lz4`, 50% de la memoria), para que el sistema no tenga que ir a swapear a ese disco.
- **Menos presión sobre la memoria.** `vm.swappiness` y `vm.vfs_cache_pressure` ajustados vía `sysctl`.
- **Logs acotados.** `journald` limitado a 50 MB; en 64 GB de disco, cada GB de logs se nota.
- **VS Code al mínimo.** Sin minimap, sin sugerencias, sin hover, sin semantic highlighting y con los file watchers recortados.
- **Videos fuera del navegador.** `yt-dlp` + `mpv` con decodificación por hardware en lugar de YouTube en el navegador.

Todas las notas, con los comandos y archivos de configuración, están en [sorceress.md](https://github.com/mamcer/spellbook/blob/main/spells/sorceress.md) de mi repo `spellbook`.

> Históricamente, al empezar en una nueva posición o al recibir una computadora nueva, le dedico tiempo a dejarla especializada y lista para la tarea en cuestión (en su momento, por ejemplo, desarrollo de software como individual contributor): 'tener el hacha siempre afilada'. Lo hacía a partir de Stack Overflow, notas, investigación y prueba y error. Hoy, con Claude Code, eso se puede llevar mucho más allá, sumando trucos de más bajo nivel que no conocía y que están incluidos arriba.

## Disciplina y foco

Creo que lo más valioso de estos seis meses no fue técnico.

Una máquina limitada te obliga a hacer una cosa por vez. No por elección, por necesidad. Y resulta que hacer una cosa por vez es, en general, la mejor forma de hacer las cosas bien.

Menos pestañas.  
Menos notificaciones.  
Menos cambio de contexto.  
Más terminal.  
Más foco.

La terminal se volvió el centro de todo. Git, el editor, los scripts, las consultas a una API, la generación de documentos, cmus o ncspot (Spotify) para escuchar música, mycli, jq. Todo en texto, todo liviano, todo a un atajo de distancia. No es nostalgia, aunque también lo sea: es que la terminal consume casi nada, y en este equipo eso no es un detalle.

![Escritorio con Alacritty y tmux](../img/2026-10-11-seis-meses-con-un-celeron/02-alacritty.png)

![Escritorio MATE con el gestor de archivos abierto](../img/2026-10-11-seis-meses-con-un-celeron/03-desktop.png)

Fuera de la terminal, el escritorio es igual de austero: MATE. Hay escritorios más livianos, pero acá pesó la nostalgia de GNOME 2.

Y cuando algo de verdad necesita potencia, no lo fuerzo acá. Para eso está el servidor casero de Poor Man's Fury: el clúster, los pipelines, la observabilidad, todo corre ahí. Esta laptop es la consola. Separar dónde se piensa de dónde se ejecuta fue una de las mejores decisiones de estos meses.

## Programar con IA en un Celeron

Esto merece un capítulo aparte.

Buena parte del trabajo de estos seis meses lo hice con Claude Code, en la terminal. Y acá pasa algo interesante: el trabajo pesado no ocurre en la laptop. El modelo corre en otro lado. Lo que corre acá es una terminal, el código y las herramientas de siempre.

De alguna forma, la IA cambió la ecuación del hardware. Lo que antes requería sin excepciones un IDE pesado (Visual Studio, GoLand, IntelliJ) indexando todo el proyecto en memoria, hoy se puede hacer desde una terminal con un asistente que lee, busca y escribe código. Analizar repositorios enteros, cruzar datos de una API, escribir y revisar documentos.

Una aclaración importante: que sea 'factible' no significa que sea recomendable. Esto no reemplaza a una máquina de trabajo. El desarrollo de software profesional sigue necesitando un IDE, herramientas web pesadas, potencia para compilar proyectos medianos o grandes en tiempos aceptables, correr suites de tests completas, etc. Todo eso es impensable en este equipo.

Lo que sí anda muy bien es Go en terminal: los tiempos de compilación y la performance son más que aceptables incluso con tan pocos recursos. No fue una sorpresa: ya he trabajado en proyectos personales con Go que terminaban deployados en un Intel Compute Stick, con un Atom, 2 GB de RAM y Debian 12 sin entorno gráfico.

No es magia, y la Latitude igual sufre cuando le pedís demasiado. Pero el piso de lo que se puede hacer con poco hardware subió mucho. Y eso, para alguien que empezó con 32 MB de RAM, no deja de sorprenderme.

## Lo que me llevo

Fueron seis meses de uso diario. Tanto, que ni la apagaba, en parte para no esperar el arranque con ese disco: la apagué recién hace unas semanas, por un viaje de trabajo. Debian 13 anduvo impecable todo este tiempo.

Hubo frustraciones, claro. El home banking, una Google Sheet, una videollamada, un video de YouTube: cada una, para funcionar, exige ser casi lo único que está haciendo la máquina en ese momento.

El tiempo pasó rápido. Pero hay cosas que me quiero llevar:

- **Preguntarme si lo necesito** antes de abrir, instalar o dejar corriendo algo.
- **Una cosa por vez.** El foco no es un lujo, es la forma de trabajar.
- **La terminal primero.** La adopté hace años, pero acá la llevé más lejos. Liviana, rápida, sin distracciones.
- **Ser consciente del costo de las cosas**, aunque sobren los recursos.

Volver a un Celeron fue, en parte, volver al principio. A esa época en que cada megabyte importaba y la computadora no era una herramienta transparente sino algo con lo que había que negociar.

Seis meses después de este experimento exagerado, seguramente no recomendable, me doy cuenta de que esa negociación me hizo trabajar mejor.