# Recon map: Obj_notes (web / desktop, local-first)

Alcance: un MVP chico que mezcla lo central de tres apps: el modelo de objetos y tipos de Anytype, las páginas de bloques y las tablas con fórmulas de Notion, y la escritura en outline con grabación de audio de Logseq.
Para: uso personal de Fabricio (un solo usuario, un solo dispositivo). Producto vendible queda para después.
Fecha: 2026-10-25

Etiquetas de confianza: **[Cierto]** verificado en una fuente pública enlazada abajo, **[Probable]** inferencia razonable a partir de docs o del uso conocido de la app, **[Suposición]** decisión o estimación mía sin evidencia directa.

## Lo incómodo primero

- [Suposición] "Un clon de Anytype con todas las funcionalidades de Notion" es un proyecto XL: son dos productos con equipos de decenas de personas y años de trabajo. Si el objetivo sigue siendo "todo", el proyecto no termina. Este documento corta a propósito.
- [Probable] Lo más caro de Anytype no es la UI sino la sincronización local-first cifrada entre dispositivos (CRDTs, red P2P). El MVP la deja **afuera**: datos locales en un solo dispositivo, con export. Meter sync desde el día uno multiplica el tamaño.
- [Probable] Las fórmulas tipo Notion 2.0 son un lenguaje de programación chico (parser, tipos, listas, `let`). El MVP implementa un subconjunto; no apunta a compatibilidad con Notion.
- [Cierto] La grabación de audio de Logseq es una función de la app móvil (iOS y Android), con transcripción automática como bloques hijos. La transcripción queda fuera del MVP; la grabación sí entra.

## Fuentes

| # | fuente | URL | notas |
| --- | --- | --- | --- |
| 1 | Anytype docs: Types | https://doc.anytype.io/anytype/organize/types | tipos como plantilla de objetos, propiedades, vistas, templates, layouts |
| 2 | Anytype docs: Collections | https://doc.anytype.io/anytype/organize/collections | colecciones manuales vs queries automáticas |
| 3 | Anytype docs: Objects | https://doc.anytype.io/anytype/create/objects | todo es un objeto |
| 4 | Notion help: Formulas 2.0 | https://www.notion.com/help/guides/new-formulas-whats-changed | tipos ricos, listas, `map`/`filter`, `let`, referencias a bases relacionadas |
| 5 | Logseq DB changelog | https://discuss.logseq.com/t/logseq-db-changelog/30013 | grabación de voz en iOS/Android, transcripción automática como bloques hijos |
| 6 | Logseq foro: audio recorder | https://discuss.logseq.com/t/audio-recorder-option/3778 | historial del pedido; en desktop no estaba |
| 7 | Logseq en App Store | https://apps.apple.com/app/id1601013908 | listado móvil |

No se usó ninguna cuenta, ni código fuente, ni APIs privadas. Solo docs públicas.

## Inventario corto por app

### Notion
- [Cierto] Bases de datos con propiedades tipadas y fórmulas 2.0: multi-línea, tipos de salida ricos (fechas, personas, listas, páginas), `map()`/`filter()`/`length()`, `let`/`lets`, acceso directo a propiedades de bases relacionadas (fuente 4).
- [Probable] Editor de bloques con menú `/` (texto, títulos, listas, to-do, toggle, código, cita, callout, imagen, archivo, embed), arrastrar y soltar bloques.
- [Probable] Vistas de base de datos: tabla, tablero (kanban), galería, lista, calendario, timeline; filtros, orden, agrupación.
- [Probable] Relaciones y rollups entre bases, plantillas de página, páginas anidadas, compartir y permisos, comentarios, historial de versiones, IA, API pública, integraciones, sitios publicados.

### Anytype
- [Cierto] Todo es un Objeto; cada objeto tiene un Tipo que define su forma (propiedades). Se puede cambiar el tipo de un objeto y se conservan sus propiedades (fuente 1).
- [Cierto] Tipos propios o predefinidos, propiedades (fecha, tags, etc.), templates por tipo, layouts por tipo (fuente 1).
- [Cierto] Colecciones (objetos elegidos a mano) y Queries (objetos filtrados por reglas que se actualizan solos), ambas con Vistas (fuente 2).
- [Cierto] Spaces con claves de cifrado separadas (fuente 1).
- [Probable] Local-first con sync P2P cifrada, grafo de objetos, widgets en el sidebar, import/export (Markdown, Notion), multi-dispositivo, compartir spaces.

### Logseq
- [Cierto] Grabación de audio en móvil (iOS con speech API, Android WAV), con transcripción automática y offline guardada como bloques hijos (fuente 5).
- [Probable] Outliner: todo es un bloque con sangría, diario automático por día (journal), links `[[página]]` y referencias a bloques, backlinks, tags, queries, grafo, tareas TODO/DOING, flashcards, whiteboards, plugins, archivos Markdown locales.

## Core loop

[Suposición] Abrir la app, escribir en una página de bloques (o en el diario del día), convertir cosas en objetos con tipo (una Tarea, una Persona, un Libro), y verlas juntas en una tabla con columnas calculadas por fórmulas. Grabar una nota de voz dentro de la página cuando escribir no conviene.

## MVP: lo que entra

| # | feature | origen | por qué |
| --- | --- | --- | --- |
| 1 | Páginas con editor de bloques: párrafo, H1-H3, lista, to-do, cita, código, divisor, menú `/`, sangría (outline) y reordenar | Notion + Logseq | es la base de todo lo demás |
| 2 | Objetos con Tipo: cada página es un objeto; Tipos definidos por el usuario con propiedades (texto, número, fecha, checkbox, select, link a objeto) | Anytype | es lo que diferencia de un editor de notas común |
| 3 | Vista tabla por Tipo (query automática) con filtros y orden simples | Anytype + Notion | ver los objetos juntos |
| 4 | Columnas de fórmula en tablas: aritmética, comparación, `if`, texto, fechas, `prop("X")`, unas 20 funciones | Notion | pedido explícito |
| 5 | Bloque de audio: grabar desde el micrófono, guardar archivo local, reproducir | Logseq | pedido explícito |
| 6 | Links `[[objeto]]` y backlinks | Logseq + Anytype | convierte notas sueltas en grafo |
| 7 | Diario del día (una página por fecha, se abre al iniciar) | Logseq | barato y es el punto de entrada diario |
| 8 | Sidebar con tipos y páginas, búsqueda rápida por título | todos | navegación mínima |
| 9 | Almacenamiento local (un archivo o base local) + export a Markdown | Anytype | local-first sin sync |

## Fuera de alcance del MVP (explícito)

- Sincronización multi-dispositivo, P2P, cifrado de extremo a extremo, colaboración en tiempo real.
- Cuentas, login, compartir, permisos, comentarios, publicar sitios.
- Vistas kanban, galería, calendario, timeline, grafo visual (vienen después; la tabla primero).
- Relaciones con rollups entre tablas, fórmulas sobre listas (`map`/`filter`), `let`, compatibilidad con fórmulas de Notion.
- Transcripción de audio a texto.
- Templates por tipo, historial de versiones, papelera con restauración.
- Import desde Notion/Anytype/Logseq, API pública, plugins, IA.
- Apps móviles nativas (el MVP es web/desktop; [Suposición] el micrófono funciona igual en el navegador).
- Flashcards, whiteboards, queries avanzadas de Logseq.

## Lo que no se puede clonar (skip)

- La comunidad, el marketplace de plantillas y la galería de integraciones de Notion.
- La red de sync de Anytype (any-sync) como servicio: se puede construir algo propio, no usar la suya.
- Marcas, íconos, textos y diseño visual de las tres apps.

## Pantallas del MVP

| ID | pantalla | cómo se llega | propósito | componentes clave | estados |
| --- | --- | --- | --- | --- | --- |
| S01 | Shell: sidebar + área principal | al abrir | navegar | sidebar, lista de tipos, lista de páginas | vacío (primer uso), lleno |
| S02 | Editor de página/objeto | click en una página | escribir | bloques, menú `/`, header de propiedades | vacío, lleno |
| S03 | Diario del día | al abrir / botón "Hoy" | entrada rápida | igual que S02, título = fecha | vacío, lleno |
| S04 | Gestor de Tipos | sidebar > Tipos | crear y editar tipos | lista, editor de propiedades | sin tipos propios, lleno |
| S05 | Vista tabla de un Tipo | sidebar > un tipo | ver objetos juntos | tabla, filtros, orden, celda editable | vacío, lleno, error de fórmula |
| S06 | Editor de fórmula | encabezado de columna fórmula | escribir fórmula | input con resaltado y error | válido, error |
| S07 | Bloque de audio | `/audio` en S02 | grabar y reproducir | botón grabar, timer, player | permiso denegado, grabando, guardado |
| S08 | Buscador rápido | Ctrl+K | saltar a un objeto | input, resultados | sin resultados |

## Flujos

```
F01 Escribir en el diario
    S03 -> escribir bloques
    clicks del camino feliz: 0 (abre en el diario)

F02 Crear un Tipo y llenar su tabla
    S04 nuevo tipo -> agregar propiedades -> S05 nueva fila -> editar celdas
    clicks: ~6
    edge: renombrar una propiedad usada en una fórmula, borrar un tipo con objetos

F03 Agregar una columna de fórmula
    S05 nueva columna tipo fórmula -> S06 escribir -> ver resultados
    clicks: ~3
    edge: error de sintaxis, referencia a propiedad inexistente, división por cero, ciclos

F04 Grabar una nota de voz
    S02 -> /audio -> grabar -> parar -> reproducir
    clicks: 3
    edge: el navegador niega el micrófono, grabación larga, cerrar la página grabando

F05 Linkear y volver
    S02 escribir [[ -> elegir objeto -> abrir -> ver backlinks
    edge: linkear un objeto que todavía no existe (crearlo)
```

## Modelo de datos inferido

```
Object     id, type_id, title, created_at, updated_at, properties (json: prop_id -> valor)
           evidencia: Anytype "todo es un objeto, cada objeto tiene un tipo" (fuente 1, 3)
           confianza: alta

Type       id, name, icon, property_ids (orden)
           evidencia: fuente 1
           confianza: alta

Property   id, name, kind (text | number | date | checkbox | select | object_link | formula),
           options (json; para select), formula (texto; para formula)
           evidencia: fuentes 1 y 4
           confianza: alta para los tipos; [Suposición] para guardar la fórmula como texto

Block      id, object_id, parent_block_id, order, kind (paragraph | heading | list | todo |
           quote | code | divider | audio | link), content (json)
           evidencia: outliner de Logseq, bloques de Notion
           confianza: media

Attachment id, object_id, block_id, mime, path, duration_ms
           evidencia: audio de Logseq (fuente 5)
           confianza: media

Link       from_object_id, from_block_id, to_object_id   (derivada del contenido, para backlinks)
           confianza: [Suposición] tabla derivada
```

Relaciones: Type 1-n Object, Type n-n Property, Object 1-n Block, Block 1-n Block (hijos), Block 0-1 Attachment, Object n-n Object vía Link.

## Feature matrix

Ver `features.csv`. Must: 9, should: 6, could: 9, skip: 3.

## Tamaño

Pantallas 8, flujos 5, entidades 6.

Partes difíciles:
1. [Probable] El editor de bloques con outline (selección, foco, sangría, pegar texto). Conviene usar una librería de editor abierta en vez de escribirlo de cero.
2. [Probable] El motor de fórmulas: parser, tipos, mensajes de error y recálculo cuando cambia una celda.
3. [Probable] Que el almacenamiento local quede listo para sumar sync después sin rehacer el modelo.

Tamaño: **M** [Suposición]: unas 4 a 8 semanas para una persona con ayuda de IA, si el alcance se mantiene. Con sync multi-dispositivo pasa a **L**; con "todo Notion" pasa a **XL**.

Siguiente paso: `/replica-architect` para elegir stack y esquema.
