---
date: 2026-10-20
layout: post
title: '"Esto ya no funciona"'
subtitle: Cómo salimos de la crisis y por qué dejamos Scrum
---

A fines de 2025 lo dije en voz alta: esto ya no funciona.

Focos de conocimiento, estimaciones que se alargaban, compromisos que no se cumplían, incidentes en producción, un equipo que era más un grupo de personas.

Los síntomas estaban a la vista:

- 4 de cada 10 ítems pasaban por más de un sprint antes de terminarse.
- Lo que se mergeaba podía tardar semanas en llegar a producción.
- Big bang releases a producción, incertidumbre, rollbacks.
- Cansancio, plannings que no predecían mucho, la sensación de estar siempre apagando incendios y corriendo por detrás de las expectativas.

## Arreglarlo a pulmón

Lo primero que hicimos fue intentar que Scrum funcionara mejor. Plannings más cuidadas, mejor refinamiento, sprints más realistas. Y a principios de 2026 los números mejoraron.

Cuando algo no funciona, una acción básica puede ser involucrarse mucho más. Y funcionó, pero también la sensación era que estaba encima de cada tema: reacomodando sprints, persiguiendo lo trabado, empujando cada release.

La conclusión inmediata de esa mejora es que no escalaba, no era sostenible. Y no era un problema de esfuerzo ni de disciplina del equipo. Era un problema del sistema: estábamos mejor, pero ahora funcionaba porque alguien lo empujaba a mano.

## El cuello de botella se había movido

Mientras tanto, la IA se iba metiendo en el día a día. Ya usábamos Cursor, y a principios de 2026 sumamos Claude Code. Tareas que antes llevaban días empezaron a salir en horas.

Ahí entendí que no alcanzaba con hacer mejor Scrum. Scrum está diseñado alrededor de un recurso escaso: la capacidad de escribir código. Los story points, la velocity y el sprint con alcance fijo existen para administrar ese cuello de botella. Con IA, escribir código dejó de ser lo escaso. Seguir puliendo ese proceso era administrar con mucho cuidado algo que ya no era el problema.

El cuello de botella estaba en todo lo que rodea al código:

- El trabajo abierto en paralelo: muchas cosas empezadas, pocas terminadas.
- Los PRs esperando revisión.
- La coordinación entre servicios que dependen unos de otros.
- La distancia entre "está mergeado", "la user story está done" y "está en producción".

Scrum no estaba mal. Optimizaba algo que ya no era nuestra restricción.

## Lo que armamos

En mayo de 2026 cambiamos la forma de trabajar. Lo llamamos Watts Flow. No es un invento: es una combinación de prácticas que otros equipos ya usaban, adaptada a nuestra realidad.

- **Appetite en lugar de estimación.** La pregunta dejó de ser "¿cuánto va a tardar esto?" y pasó a ser "¿cuánto vale la pena invertir en esto?". El tiempo queda fijo y lo que se ajusta es el alcance. Viene de Shape Up, de Basecamp.
- **Un tablero de flujo con límites de trabajo en curso.** Viene de Kanban. Más cosas abiertas no es más productividad. Por la Ley de Little, es más tiempo de espera para cada una.
- **Done significa "en producción".** Mergeado no cuenta. User story finalizada no cuenta. Viene de Continuous Delivery.
- **Una daily con un foco:** ¿qué está en riesgo de no avanzar hoy?
- **Métricas de entrega en lugar de story points.** Miramos cuánto tarda el trabajo en llegar a producción y dónde se traba, con las métricas de DORA, no cuánto "esfuerzo" estimamos.
- **Owners de tracks.** Lo sumamos hace muy poco y viene muy bien.

Seguimos planificando cada dos semanas. Pero la planning ya no es para llenar un sprint con puntos: es una decisión de inversión sobre qué problemas valen la pena en este ciclo.

Lo central no fue la lista de prácticas. Fue que el proceso dejó de depender de una persona. El tablero muestra lo que está trabado sin que nadie tenga que preguntarlo. El appetite resuelve en la planning lo que antes se renegociaba durante todo el sprint.

El detalle completo está en el handbook, por si a alguien le sirven ideas para su realidad: [Watts Flow](https://github.com/mamcer/spellbook/blob/main/spells/watts-flow.md)

## Los tres momentos, en números

Comparo tres períodos: la crisis (noviembre 2025 a enero 2026), la recuperación a pulmón (febrero a abril 2026) y los primeros cuatro meses completos con Watts Flow (junio a septiembre 2026). Los datos salen de Azure DevOps y del historial de git.

- **Del merge a producción (mediana, servicio principal del backend):** casi 6 días en la crisis, casi 4 a pulmón, 1 día con Watts Flow.
- **Deploys a producción del backend por semana:** 2.7, después 5.1, ahora 6.4.
- **Ítems que pasan por más de un sprint antes de terminarse:** 40%, después 18%, ahora 15%.
- **Tiempo de ciclo, desde que se empieza un ítem hasta que se termina (mediana):** 5 días, 5 días, 3 días.
- **Reverts sobre pasajes a producción:** 1 de 36, después 9 de 65, ahora 5 de 112.
- **Cambios mergeados por mes:** 154, después 217, ahora 316. Con un equipo un 36% más chico que el de la recuperación a pulmón.
- **Movimientos de ítems entre sprints hechos por el manager:** 1 de cada 5, 1 de cada 6, 1 de cada 12.

Para ser justo con los números:

- La IA ya estaba en el equipo durante la recuperación a pulmón, pero su uso creció mucho más después, en paralelo con el cambio de proceso. No le atribuyo todo al framework. Lo que sí muestran los números es que la IA sola no alcanzaba: con IA y sin el proceso, el tiempo de ciclo no se movió. Y el proceso existe justamente porque la IA cambió dónde está el cuello de botella.
- "Cambios mergeados" no es lo mismo que valor entregado. Es una señal de flujo, no de impacto.
- En la crisis casi no había reverts, pero también se desplegaba muy poco.

## Lo que conté antes y lo que cuento ahora

Ya escribí sobre esto antes: en mayo, en este blog ([AI Didn't Replace Our Bottlenecks. It Moved Them.]({{< ref "from-scrum-to-ai-centered-engineering.md" >}})), y hace unos meses en LinkedIn. Diferencias que nos dejó el tiempo:

En ese momento lo presenté como una adopción de IA, "una forma de trabajar centrada en AI". Mirando los datos con más distancia, lo que hizo la diferencia fue el proceso alrededor. La IA cambió dónde estaba el cuello de botella; el proceso es lo que nos permitió aprovecharlo.

Los números miden cosas distintas:

- **Los resultados de mayo eran de Q1, antes de Watts Flow.** Eran la IA más el esfuerzo a pulmón. El proceso recién arrancaba.
- **4.6 horas de lead time (mayo) vs. casi 4 días del merge a producción.** El de mayo medía desde el commit hasta el merge: la parte que la IA aceleró. Lo que pasaba después del merge no lo medía, y en ese mismo período seguía en casi 4 días. Ahí estaba el cuello de botella que se había movido.
- **0.5 → 2 releases por semana (LinkedIn) vs. 2.7 → 6.4 deploys por semana.** El primero cuenta releases planificados, y el 2 era el mínimo que nos pusimos, no el promedio. Acá cuento todos los pasajes a producción del backend, incluidos hotfixes, cambios de infraestructura y respuestas a incidentes, que hoy son un tercio del total.
- **0% de releases revertidos (LinkedIn) vs. 5 reverts de 112.** El 0% era sobre releases planificados. Los 5 reverts de acá salen de todos los pasajes a producción, así que también incluyen hotfixes revertidos.
- **1.7x de productividad individual (LinkedIn) vs. cambios mergeados.** El 1.7x fue una medición de las primeras semanas. Acá muestro cambios mergeados por mes del equipo entero, que es una señal de flujo y no de productividad individual.

El de LinkedIn también contaba lo que este no cuenta: lo que llegó a los usuarios (la nueva experiencia de pago con tarjeta, el motor de descuentos, 25% menos tickets de soporte) y la estabilidad de la plataforma (83+ días estables, incluido el arranque del Mundial). Y eso se sostuvo: hace meses que no tenemos un incidente que nos lleve a un warroom. Los números de flujo de acá son la otra mitad: cómo llegamos a poder entregar eso.

## Lo que todavía no resolvimos

- Un tercio de lo que llega a producción todavía va por fuera del release normal: hotfixes, cambios de infraestructura, respuestas a incidentes. Hay que separar y medir mejor cada caso.
- En mobile el cambio es menor: los stores marcan el ritmo.
- Todavía no medimos bien el tiempo que un PR espera revisión.

Y el proceso en sí: como el objetivo se mueve (un equipo tomando desafíos nuevos, más grandes, nuevas responsabilidades cada Quarter), el proceso también se tiene que mover. Sigue en beta continuo.

## Para cerrar

Lo que más valoro es que ya no depende de que nadie (particularmente yo) esté encima de cada tema. Hoy lo sostiene, y en gran parte lo evoluciona, el equipo. Lo que no puede garantizar ningún proceso es que se generen las condiciones para que las personas crezcan, trabajen como un equipo, una unidad. Un equipo que puede cometer errores como cualquier otro, pero que difícilmente pueda quedarse derrotado.
