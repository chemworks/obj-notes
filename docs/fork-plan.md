# Plan: Obj_notes como fork de Anytype

Fecha: 2026-10-25. Estado: plan, sin código.
Decisión de base: uso personal, así que se parte de un fork de Anytype (ver `replica/recon.md`, sección "Opción: partir de un fork de Anytype").

Etiquetas: **[Cierto]** verificado en el código o en la licencia, **[Probable]** inferencia razonable, **[Suposición]** decisión o estimación sin evidencia directa.

## Lo incómodo primero

- [Cierto] Anytype ya tiene una "fórmula" en las tablas, pero es la del **pie de columna**: Count, Distinct, Empty, % Empty, Min/Max de fechas, sumas. Está en el cliente (`src/ts/lib/relation.ts`, `formulaByType`; enum `FormulaType` en `src/ts/interface/block/dataview.ts`). No es una columna calculada por fila como en Notion. Eso es lo que hay que construir.
- [Cierto] El middleware no tiene un tipo de propiedad calculada. Los formatos son `longtext, shorttext, number, status, tag, date, file, checkbox, url, email, phone, emoji, object, relations, map` (`pkg/lib/pb/model/models.pb.go`). Agregar `formula` es cambiar el protobuf, y eso toca los dos repos.
- [Probable] Cada cambio de protobuf va a chocar con upstream en cada actualización. Mantener el fork al día es un costo fijo que no se termina: hay que decidir cada cuánto se rebasea.
- [Probable] Las apps móviles oficiales no van a entender el formato nuevo. El fork es solo de escritorio.

## Licencia (resuelto)

- [Cierto] `anytype-ts` y `anytype-heart` están bajo la Any Source Available License 1.0. "Non-Commercial Use" incluye uso personal. Se puede modificar y redistribuir bajo la misma licencia, incluyendo una copia de ella.
- [Cierto] Hay modo de red `LocalOnly` (`RpcAccount_LocalOnly` en `pb/commands.pb.go`), así que la app corre sin conectarse a ninguna red de sync. Calza con "sin sync por ahora".
- [Suposición] Lo que escribamos nosotros puede tener otra licencia, pero el conjunto sigue atado a la ASAL. Si un día se quiere vender, hay que reescribir.

## Qué ya viene hecho en Anytype

| pedido | estado en Anytype | fuente |
| --- | --- | --- |
| Objetos, Tipos, propiedades | hecho | [Cierto] docs de Types |
| Tablas genéricas (queries y colecciones con vistas) | hecho | [Cierto] docs de Collections |
| Relacionar tablas (propiedad que apunta a objetos de otro tipo) | hecho, formato `object` | [Cierto] `RelationFormat_object` |
| Backlinks y grafo | hecho | [Probable] uso conocido de la app |
| Editor de bloques | hecho | [Cierto] `core/block/editor` |
| Fórmulas por fila | **falta** | [Cierto] no existe el formato |
| Rollups (leer valores de objetos relacionados) | **falta** | [Probable] solo hay agregados de pie de columna |
| Audio | después | decisión 2026-10-25 |

## Diseño propuesto

### Dónde se calcula

Recomendación: calcular en el **middleware (Go)** y guardar el resultado como un detalle más del objeto.

- [Probable] Ordenar, filtrar y agrupar lo hace el middleware (`pkg/lib/database`). Si la fórmula se calcula solo en el cliente, se ve en la tabla pero no se puede ordenar ni filtrar por ella.
- Alternativa más barata: calcular en el cliente (TypeScript), solo para mostrar. Sirve como prototipo de una semana para validar la idea, pero no es el destino.

### Cambios

1. **Protobuf** (`anytype-heart`): nuevo `RelationFormat_formula` y un detalle en el objeto-relación con el texto de la fórmula (por ejemplo `relationFormula`). [Suposición] El número del enum va lejos de los usados (por ejemplo 200) para no chocar con upstream.
2. **Motor de fórmulas** (Go): parsear y evaluar. [Suposición] Usar una librería de expresiones existente (por ejemplo `expr-lang/expr`, MIT) en vez de escribir el parser; hay que confirmar que permite registrar `prop()` y funciones propias.
3. **Recalcular**: cuando cambia un detalle de un objeto, recalcular sus fórmulas, y también las de los objetos que lo referencian (rollups). [Probable] Se engancha donde el indexer o las suscripciones ya reaccionan a cambios de detalles (`core/indexer`, `docs/Subscriptions.md`).
4. **Rollups** como funciones de fórmula: `sum(prop("Tareas"), "Horas")`, `count(prop("Tareas"))`, `min`, `max`, `avg` sobre los objetos de una propiedad `object`. Sin `map`/`filter` generales en v1.
5. **Cliente** (`anytype-ts`): el tipo "Fórmula" en el editor de propiedades (`component/menu/dataview/relation/edit.tsx`), un editor de fórmula con error visible, y la celda de solo lectura en la tabla.

### Lenguaje v1

- Operadores `+ - * / % == != < <= > >= and or not`.
- `prop("Nombre")`, `if(cond, a, b)`, `concat`, `length`, `lower`, `upper`, `round`, `abs`, `now()`, `dateAdd`, `dateBetween`, `formatDate`, `empty`.
- Rollups: `sum`, `count`, `min`, `max`, `avg` sobre propiedades `object`.
- Errores: referencia a propiedad inexistente, tipo incorrecto, división por cero, ciclo entre fórmulas.

## Estructura de repos

- [Suposición] Dos forks en GitHub: `chemworks/anytype-heart` y `chemworks/anytype-ts`. `chemworks/obj-notes` queda para el plan, las decisiones y un script que arma la versión local (clona los dos, compila el middleware con `make install-dev-js`, corre el cliente con `bun install` y `bun run electron`, según el README de anytype-ts).
- Cada cambio en una rama propia sobre el último release de upstream, para que rebasear sea predecible.

## Etapas

| etapa | qué | tamaño [Suposición] |
| --- | --- | --- |
| 0 | Compilar y correr los dos repos sin cambios, en modo LocalOnly | 1 a 3 días (el build de Go + Electron es lo que suele trabarse) |
| 1 | Prototipo de fórmula en el cliente, solo para mostrar | 1 semana |
| 2 | Formato `formula` en protobuf + motor en Go + recalcular por objeto | 2 a 3 semanas |
| 3 | Rollups sobre propiedades `object` + recalcular dependientes | 1 a 2 semanas |
| 4 | Pulido: editor con errores, ordenar y filtrar por fórmulas, tests | 1 semana |
| después | Audio (bloque de grabación), luego transcripción | sin estimar |

Total hasta etapa 4: unas 5 a 8 semanas para una persona con ayuda de IA [Suposición]. El riesgo mayor no es la fórmula sino aprender las dos bases de código.

## Fork vs proponer la mejora a Anytype (evaluado 2026-10-25)

- [Cierto] Aceptan PRs externos contra la rama `develop`. Las revisa el equipo y hay que firmar un CLA, que corre como bot en los dos repos (`.github/workflows/cla.yml`). Las ideas de features se piden como issue o en el foro. Fuente: https://github.com/anyproto/.github/blob/main/docs/CONTRIBUTING.md
- [Cierto] El pedido de fórmulas existe hace años en el foro ("Derived Relations (Formulas)", https://community.anytype.io/t/derived-relations-formulas/1554) y todavía no se implementó. [Probable] No encontré respuesta del equipo con fecha ni plan en la página revisada.
- [Probable] Una propiedad nueva cambia el protobuf y obliga a que la soporten también iOS, Android y la sync. Es una decisión de producto del equipo de Anytype, no un PR que se mergea solo. Un PR grande sin acuerdo previo tiene pocas chances.
- [Probable] Si lo aceptan, se ahorra el costo de mantener el fork y funciona en móvil y con sync. Si no, no se ahorra nada y el control del cronograma queda en manos de ellos.
- [Cierto] Con el CLA, el código que aportes queda bajo las condiciones que defina Any.

Recomendación: **híbrido**. Construir en el fork, con el cambio aislado y en ramas sobre `develop`. En paralelo, abrir una discusión con el diseño (formato, lenguaje, dónde se calcula) para ver si lo quieren. Si dicen que sí, se manda como PR y el fork queda para lo que no acepten. No conviene esperar a que ellos lo construyan.

## Decisiones abiertas

- Si la etapa 1 (prototipo en cliente) vale la pena o se va directo a la etapa 2.
- Cada cuánto se actualiza el fork contra upstream.
