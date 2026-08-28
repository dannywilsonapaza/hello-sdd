# Spec 001 — MVP de habits-cli

Estado: **revisada, pendiente de aprobación** · Versión: 2.0 · Reemplaza a la 1.0

> Regla de proceso (principio 2 de la constitución): esta spec pasa a **aprobada** solo cuando
> no queda ningún `[NECESITA ACLARACIÓN]` y el responsable la aprueba explícitamente. Hasta
> entonces no se escribe código que dependa de ella.

## 1. Contexto y objetivo

Quien estudia por su cuenta pierde la noción de su constancia: sabe si estudió hoy, pero no
cuántos días seguidos lleva. Sin esa señal, la motivación depende de la memoria.

El objetivo de esta primera funcionalidad es que una persona pueda **registrar sus hábitos de
estudio y ver, en un vistazo, cuántos días consecutivos lleva cumpliendo cada uno**. La racha
es el valor central del producto: convierte el esfuerzo diario en un número que motiva a no
romper la cadena.

El alcance es deliberadamente mínimo: tres acciones (`add`, `done`, `list`) que funcionen de
forma predecible y sin sorpresas. Todo lo demás se pospone.

## 2. Usuarios

- **Estudiante autodidacta (usuario principal).** Trabaja en su propia máquina, cómodo con la
  terminal. Ejecuta la herramienta una o dos veces al día: marca lo que cumplió y mira su
  progreso. Quiere respuestas inmediatas y mensajes que entienda sin documentación.
- **Desarrollador junior que mantiene el proyecto (usuario secundario).** Necesita que el
  comportamiento esté descrito sin ambigüedad para poder cambiarlo con confianza.

## 3. Historias de usuario

- **HU-1.** Como estudiante, quiero crear un hábito con un nombre, para empezar a hacerle
  seguimiento.
- **HU-2.** Como estudiante, quiero marcar un hábito como cumplido hoy, para dejar constancia
  del día en que lo hice.
- **HU-3.** Como estudiante, quiero ver la lista de mis hábitos con su racha de días
  consecutivos, para saber cuál estoy manteniendo y cuál he roto.
- **HU-4.** Como estudiante, quiero que mis registros sigan ahí la próxima vez que abra la
  herramienta, para que la racha tenga sentido a lo largo del tiempo.
- **HU-5.** Como estudiante, quiero que un error mío (nombre repetido, hábito inexistente) me
  lo digan con claridad y sin perder datos, para corregirlo al instante.

## 4. Definiciones

Estas definiciones son normativas: los requisitos las usan y no las repiten.

- **D-1 · Nombre normalizado.** El nombre tal como lo escribe el usuario, tras: (a) recortar
  los espacios en blanco iniciales y finales, y (b) colapsar toda secuencia de espacios en
  blanco internos en un único espacio. Ejemplo: `"  leer   libros "` → `"leer libros"`.
- **D-2 · Nombre mostrado.** El nombre normalizado tal como se escribió al crear el hábito.
  Es el que aparece en todos los mensajes y en el listado.
- **D-3 · Clave de identidad.** El nombre normalizado convertido a minúsculas. Dos hábitos son
  el mismo si y solo si comparten clave de identidad. Los acentos y diacríticos **sí**
  distinguen: `"Leer"` y `"leer"` son el mismo hábito; `"léer"` es otro distinto.
- **D-4 · Hoy.** La fecha del calendario local del equipo, sin hora. Se determina **una sola
  vez al inicio de cada ejecución** y no vuelve a consultarse durante esa ejecución.
- **D-5 · Día cumplido.** Una fecha de calendario en la que el hábito fue marcado. Un día está
  cumplido o no lo está: registrar el mismo día varias veces no lo hace "más cumplido".
- **D-6 · Racha actual.** El número de días consecutivos cumplidos contados hacia atrás desde
  el día ancla, donde el día ancla es *hoy* si hoy está cumplido, o *ayer* si hoy no está
  cumplido pero ayer sí. Si ni hoy ni ayer están cumplidos, la racha actual es 0.

## 5. Requisitos funcionales

Notación EARS: **Cuando** (evento) · **Si… entonces** (condición no deseada) ·
**Mientras** (estado) · **El sistema deberá** (requisito siempre activo).

### RF-1 — Normalización del nombre

El sistema deberá normalizar (D-1) todo nombre recibido del usuario antes de usarlo para
cualquier fin: validar, comparar, registrar, buscar o mostrar.

Criterios de aceptación:
- Cuando el usuario indique un nombre con espacios iniciales, finales o internos repetidos, el
  sistema deberá operar sobre su forma normalizada.
- El sistema deberá guardar y mostrar el nombre normalizado (D-2), nunca la cadena original sin
  normalizar.

### RF-2 — Crear un hábito (`add`)

El sistema deberá permitir crear un hábito identificado por su nombre.

Criterios de aceptación:
- Cuando el usuario cree un hábito cuya clave de identidad (D-3) no existe, el sistema deberá
  registrarlo sin ningún día cumplido, confirmarlo con un mensaje en español que incluya el
  nombre mostrado, y terminar con éxito.
- Cuando un hábito acabe de crearse, el sistema deberá informar una racha actual de 0 al
  listarlo.

### RF-3 — La identidad del hábito es única

Criterios de aceptación:
- Si el usuario intenta crear un hábito cuya clave de identidad (D-3) ya existe, entonces el
  sistema deberá rechazar la operación con un mensaje de error en español, no modificar ningún
  dato existente y terminar con fallo.
- Si el usuario crea `"leer"` y luego intenta crear `"  Leer "`, entonces el sistema deberá
  rechazarlo como duplicado (la normalización y el plegado de mayúsculas se aplican antes de
  comparar).
- Cuando el usuario cree `"leer"` y luego `"léer"`, el sistema deberá tratarlos como dos
  hábitos distintos y aceptar ambos.

### RF-4 — Validación del nombre

Criterios de aceptación:
- Si el nombre normalizado queda vacío (cadena vacía o solo espacios en blanco), entonces el
  sistema deberá rechazar la operación con un mensaje de error en español, no registrar nada y
  terminar con fallo.
- Si el nombre normalizado supera los 100 caracteres, entonces el sistema deberá rechazarlo con
  un mensaje de error en español y terminar con fallo.
- Si el nombre contiene caracteres de control (saltos de línea, tabuladores verticales, nulos),
  entonces el sistema deberá rechazarlo con un mensaje de error en español y terminar con fallo.

### RF-5 — Marcar un hábito como cumplido hoy (`done`)

El sistema deberá permitir registrar que un hábito se cumplió el día actual.

Criterios de aceptación:
- Cuando el usuario marque un hábito existente que aún no está cumplido hoy, el sistema deberá
  registrar *hoy* (D-4) como día cumplido, confirmarlo con un mensaje en español y terminar con
  éxito.
- El sistema deberá registrar únicamente fechas de calendario, sin hora.
- El sistema deberá usar la misma fecha *hoy* durante toda la ejecución, aunque la ejecución
  cruce la medianoche.
- El sistema no deberá ofrecer forma alguna de marcar una fecha distinta de hoy.

### RF-6 — Marcar dos veces el mismo día es inofensivo

Criterios de aceptación:
- Cuando el usuario marque un hábito que ya está cumplido hoy, el sistema deberá dejar los
  datos sin cambios, informar en español que ya estaba marcado hoy y terminar con éxito.
- Mientras un hábito esté cumplido hoy, marcarlo de nuevo no deberá alterar su racha actual.

### RF-7 — Marcar un hábito inexistente

Criterios de aceptación:
- Si el usuario intenta marcar un hábito cuya clave de identidad no existe, entonces el sistema
  deberá informarlo con un mensaje de error en español, no crear el hábito, no modificar ningún
  dato y terminar con fallo.

### RF-8 — Listar los hábitos con su racha (`list`)

Criterios de aceptación:
- Cuando el usuario liste los hábitos y exista al menos uno, el sistema deberá mostrar una línea
  por hábito con su nombre mostrado (D-2) y su racha actual (D-6) en días, y terminar con éxito.
- El sistema deberá ordenar el listado de forma ascendente por la clave de identidad (D-3),
  comparando carácter a carácter por punto de código Unicode.
- El sistema no deberá depender de la configuración regional del equipo para ordenar: el mismo
  conjunto de hábitos produce el mismo orden en cualquier máquina.
- El sistema deberá mostrar solo el nombre y la racha; ningún otro dato forma parte del listado
  en este MVP.

> Nota sobre el orden: ordenar por punto de código sitúa las palabras acentuadas después de sus
> equivalentes sin acento (`leer` antes que `léer`). Se acepta a cambio de un orden idéntico en
> todas las máquinas, exigido por RNF-5.

### RF-9 — Cálculo de la racha actual

El sistema deberá calcular la racha actual según D-6.

Criterios de aceptación:
- Cuando el hábito esté cumplido hoy, el sistema deberá contar los días consecutivos cumplidos
  terminando en hoy.
- Cuando el hábito no esté cumplido hoy pero sí ayer, el sistema deberá contar los días
  consecutivos cumplidos terminando en ayer.
- Si el día cumplido más reciente es anterior a ayer, entonces el sistema deberá informar una
  racha de 0.
- Si el hábito no tiene ningún día cumplido, entonces el sistema deberá informar una racha de 0.
- El sistema deberá detener el conteo en el primer día no cumplido: un día saltado corta la
  racha, y los días cumplidos anteriores a ese corte no se suman.
- El sistema deberá contar cada fecha una sola vez: si una misma fecha aparece repetida en los
  datos guardados, la racha deberá ser la misma que si apareciera una sola vez.
- El sistema deberá producir el mismo resultado sea cual sea el orden en que estén guardadas las
  fechas.
- Cuando un hábito con una racha anterior ya rota se marque hoy, el sistema deberá informar una
  racha de 1: la cadena antigua no revive.
- El sistema deberá contar los días por calendario, de modo que una racha no se rompa al cruzar
  fin de mes o fin de año.

> Justificación (no verificable, solo explica el diseño): admitir *ayer* como ancla evita que la
> racha aparezca rota por la mañana, cuando al usuario aún le queda todo el día para cumplir.

### RF-10 — Lista vacía

Criterios de aceptación:
- Cuando el usuario liste los hábitos y no exista ninguno, el sistema deberá mostrar un mensaje
  en español indicando que aún no hay hábitos y terminar con éxito: la ausencia de hábitos no es
  un error.

### RF-11 — Los datos sobreviven entre ejecuciones

Criterios de aceptación:
- Cuando el usuario vuelva a ejecutar la herramienta, el sistema deberá mostrar los hábitos y
  días cumplidos registrados en ejecuciones anteriores.
- Mientras el usuario no ejecute una acción que modifique datos, el sistema no deberá alterar lo
  guardado.
- Cuando no exista ningún dato previo, el sistema deberá comportarse como si no hubiera hábitos
  (RF-10), no como un error.

### RF-12 — Toda escritura es todo-o-nada

Criterios de aceptación:
- Si una ejecución se interrumpe mientras el sistema guarda datos, entonces en la siguiente
  ejecución los datos deberán reflejar íntegramente el estado anterior o íntegramente el nuevo,
  nunca una mezcla parcial.
- Mientras una operación termine con fallo, los datos guardados deberán quedar exactamente como
  estaban antes de la operación.

### RF-13 — Invocación y códigos de salida

Criterios de aceptación:
- Cuando el usuario invoque una operación válida y esta se complete, el sistema deberá terminar
  con código de salida `0`.
- Si el usuario invoca la herramienta sin ningún argumento, entonces el sistema deberá mostrar
  la ayuda en español y terminar con código de salida distinto de `0`.
- Si el usuario invoca un comando desconocido, entonces el sistema deberá mostrar un mensaje de
  error en español y terminar con código de salida distinto de `0`.
- Si el usuario invoca un comando sin los argumentos que requiere, entonces el sistema deberá
  mostrar un mensaje de error en español y terminar con código de salida distinto de `0`.
- Si el usuario pasa argumentos de más o no reconocidos, entonces el sistema deberá mostrar un
  mensaje de error en español y terminar con código de salida distinto de `0`; el sistema no
  deberá ignorarlos en silencio.
- El sistema deberá comunicar todo error mediante un mensaje comprensible en español, nunca con
  una traza de excepción.

### RF-14 — Datos inaccesibles, inválidos o incoherentes

Criterios de aceptación:
- Si los datos guardados no se pueden interpretar, entonces el sistema deberá mostrar un mensaje
  de error en español, terminar con fallo y no borrarlos ni sobrescribirlos.
- Cuando los datos guardados existan pero estén completamente vacíos, el sistema deberá tratarlo
  como "no hay hábitos" (RF-10), no como datos inválidos.
- Si el sistema no puede leer o escribir los datos por falta de permisos, entonces deberá
  explicarlo en español indicando que se trata de un problema de acceso, y terminar con fallo.
- Si los datos guardados contienen algún día cumplido posterior a *hoy*, entonces el sistema
  deberá informarlo como incoherencia en español y terminar con fallo, en lugar de calcular una
  racha sobre datos imposibles.

### RF-15 — Forma de los mensajes

Criterios de aceptación:
- El sistema deberá concordar el número en singular y plural: `1 día`, `0 días`, `2 días`.
- El sistema deberá incluir el nombre mostrado del hábito afectado en todo mensaje de
  confirmación o de error referido a un hábito concreto.

## 6. Requisitos no funcionales

- **RNF-1 — Inmediatez.** Cualquier comando responde en menos de 1 segundo con un uso normal
  (decenas de hábitos y un año de historial).
- **RNF-2 — Uso local y sin red.** Funciona sin conexión a internet y sin cuentas de usuario.
- **RNF-3 — Idioma.** Todos los mensajes dirigidos al usuario están en español.
- **RNF-4 — Comprensibilidad.** Los mensajes explican qué pasó y, cuando aplique, qué hacer a
  continuación; nada de códigos internos ni jerga técnica.
- **RNF-5 — Determinismo.** Con los mismos datos guardados y la misma fecha *hoy*, la salida es
  byte a byte idéntica en cualquier máquina, sin depender de la configuración regional (el orden
  del listado queda fijado por RF-8).
- **RNF-6 — Integridad.** Ninguna interrupción deja los datos inservibles; la garantía observable
  está en RF-12.

## 7. Casos límite

| # | Caso | Comportamiento esperado |
|---|---|---|
| CL-1 | Marcar dos veces el mismo día | Sin cambios, mensaje informativo, éxito (RF-6) |
| CL-2 | Hábito creado hoy y marcado hoy | Racha 1 |
| CL-3 | Marcado ayer pero no hoy | Racha viva, cuenta hasta ayer (RF-9) |
| CL-4 | Marcado anteayer, ni ayer ni hoy | Racha 0 |
| CL-5 | Creado y nunca marcado | Aparece en el listado con racha 0 |
| CL-6 | Racha rota hace semanas y se marca hoy | Racha 1; la cadena antigua no revive (RF-9) |
| CL-7 | Nombre con espacios sobrantes (`"  leer  "`) | Se normaliza; si queda vacío, se rechaza (RF-1, RF-4) |
| CL-8 | Espacios internos repetidos (`"leer  libros"`) | Se normaliza a `"leer libros"` (RF-1) |
| CL-9 | Crear `"leer"` y luego `"  Leer "` | Rechazado por duplicado (RF-3) |
| CL-10 | Crear `"leer"` y luego `"léer"` | Dos hábitos distintos; en el listado `leer` va antes que `léer` (RF-3, RF-8) |
| CL-11 | Racha que cruza fin de mes o fin de año | La racha continúa (RF-9) |
| CL-12 | Fechas desordenadas o repetidas en los datos | La racha no cambia (RF-9) |
| CL-13 | Primera ejecución, sin datos previos | "No hay hábitos", éxito (RF-11) |
| CL-14 | Datos guardados vacíos (cero contenido) | "No hay hábitos", éxito (RF-14) |
| CL-15 | Datos guardados ilegibles o corruptos | Error claro, fallo, no se sobrescriben (RF-14) |
| CL-16 | Sin permisos de lectura o escritura sobre los datos | Error de acceso en español, fallo (RF-14) |
| CL-17 | Datos con un cumplimiento posterior a hoy (reloj cambiado o viaje) | Error de incoherencia, fallo (RF-14) |
| CL-18 | La ejecución cruza la medianoche | Se usa la fecha capturada al inicio (D-4, RF-5) |
| CL-19 | Interrupción (Ctrl+C o corte) a mitad de escritura | Estado anterior completo o nuevo completo, nunca parcial (RF-12) |
| CL-20 | Invocación sin argumentos | Ayuda en español, salida distinta de 0 (RF-13) |
| CL-21 | Argumentos de más o no reconocidos | Error en español, salida distinta de 0 (RF-13) |
| CL-22 | Nombre de más de 100 caracteres o con caracteres de control | Rechazado con error (RF-4) |

## 8. Fuera de alcance (MVP)

- Borrar o renombrar hábitos.
- Marcar fechas distintas de hoy, deshacer una marca o editar el historial.
- Recordatorios, notificaciones, estadísticas, porcentajes de cumplimiento, gráficos y récord
  histórico de racha más larga.
- Multiusuario, cuentas, sincronización en la nube, exportar o importar datos.
- Categorías, etiquetas, metas por hábito y frecuencias distintas de la diaria (p. ej. "3 veces
  por semana").
- Ordenación alfabética según las reglas del idioma (colación local); el MVP ordena por punto de
  código Unicode (RF-8).

## 9. Criterios de finalización

1. Los tres comandos (`add`, `done`, `list`) cumplen RF-1 a RF-15 de forma observable desde la
   terminal.
2. Cada criterio de aceptación tiene al menos una prueba automatizada asociada, y los casos
   límite CL-1 a CL-22 están cubiertos.
3. La suite de pruebas pasa completa (principio 4 de la constitución).
4. Un usuario nuevo puede crear un hábito, marcarlo y ver su racha sin leer documentación,
   guiándose solo por los mensajes de la herramienta.
5. Ninguna entrada inválida ni ningún estado de los datos produce una traza de excepción.
6. No queda ningún `[NECESITA ACLARACIÓN]` sin resolver en esta spec.

## 10. Dudas abiertas

Ninguna. Todas las de la versión 1.0 fueron resueltas y están recogidas como decisiones en las
secciones 4 y 5.

## Apéndice — Cambios respecto a la versión 1.0

Resueltos tras el informe QA:

- **Nueva sección 4 (Definiciones)** para nombre normalizado, clave de identidad, *hoy*, día
  cumplido y racha: elimina la ambigüedad de comparar y ordenar nombres.
- **Identidad de los hábitos:** insensible a mayúsculas, sensible a acentos; la unicidad se
  evalúa sobre el nombre ya normalizado (antes indefinido).
- **Orden del listado:** fijado por punto de código Unicode e independiente del locale, lo que
  resuelve el conflicto entre RNF-5 (determinismo) y el orden alfabético.
- **Racha:** añadida la regla de deduplicación de fechas, que faltaba y hacía inalcanzable el
  antiguo CL-8; la justificación no observable salió de los criterios y quedó como nota.
- **Invocación:** separada en criterios unívocos (sin argumentos, comando desconocido,
  argumentos faltantes, argumentos sobrantes), sin alternativas "ayuda **o** error".
- **Fecha:** *hoy* se captura una vez por ejecución y se rechazan los cumplimientos futuros.
- **Datos:** cubiertos vacío, corrupto, sin permisos e incoherente; la garantía de escritura
  todo-o-nada pasó de intención (RNF-6) a requisito observable (RF-12).
- **Mensajes:** fijados singular/plural y la presencia del nombre del hábito.
- **Nombres de comandos:** `add`, `done`, `list`, con mensajes en español.
