# Spec: fórmulas relacionales en tablas (fork de Anytype)

Fecha: 2026-10-25. Estado: diseño, sin código.
Contexto: `docs/fork-plan.md`. Lo que se construye es una propiedad nueva de tipo **Fórmula**. Calcula un valor por objeto (por fila de la tabla) y puede leer propiedades de objetos de otras tablas a través de relaciones.

Etiquetas: **[Cierto]** verificado en el código o en docs, **[Probable]** inferencia razonable, **[Suposición]** decisión de diseño sin evidencia directa.

## 1. Qué motor usar: JS, Python o Go

| opción | dónde corre | a favor | en contra |
| --- | --- | --- | --- |
| Python | ninguno de los dos repos | sintaxis conocida | [Cierto] ni `anytype-ts` (TypeScript/Electron) ni `anytype-heart` (Go) tienen Python. Habría que meter un intérprete extra en la app solo para esto. **Descartado.** |
| `mathjs` (JS) | cliente Electron | muy completo, fácil de probar | [Cierto] el orden, los filtros y las vistas los resuelve el middleware en Go (`pkg/lib/database`, `core/subscription`). Si el valor se calcula en el cliente, no se puede ordenar ni filtrar por él, y las filas que no están en pantalla no se calculan. |
| `expr-lang/expr` (Go) | middleware | [Cierto] licencia MIT, chequeo de tipos al compilar, funciones propias con `expr.Function`, `map`/`filter`/`all`/`any` incluidos. Corre donde están los datos, así que ordenar y filtrar funcionan solos. | hay que escribir Go. [Probable] `sum`, `mean`, `min`, `max` y `round` vienen incluidos en las versiones recientes; hay que confirmarlo y, si falta alguno, se registra a mano. |

**Recomendación: `expr-lang/expr` dentro de `anytype-heart`.** [Cierto] Hoy ninguno de los dos repos trae una librería de expresiones (revisé `go.mod` y `package.json`), así que no hay nada que reutilizar. `mathjs` solo serviría para un prototipo visual de pocos días.

## 2. Cómo se ve para el usuario

En una tabla de Tipo **Proyecto** que tiene una propiedad **Tareas** (objetos de Tipo **Tarea**):

```
prop("Precio") * prop("Cantidad")
prop("Estado") == "Hecho" ? "OK" : "Pendiente"
dateBetween(now(), prop("Vence"), "days")

len(prop("Tareas"))                                // cuántas tareas linkeadas
sum(rel("Tareas", "Horas"))                        // suma de Horas de esas tareas
avg(rel("Tareas", "Avance"))
max(rel("Tareas", "Vence"))                        // fecha más lejana
count(rel("Tareas", "Estado"), # == "Hecho")
join(rel("Tareas", "Nombre"), ", ")
```

- `prop("X")` lee una propiedad del objeto mismo. Si `X` es una relación a objetos, devuelve la lista de objetos.
- `rel("Relación", "Propiedad")` sigue la relación y devuelve la lista de valores de esa propiedad en los objetos de la otra tabla. Es la función "relacional", equivalente a un rollup de Notion pero escrito como fórmula.
- Se puede leer una fórmula de otra tabla: `sum(rel("Tareas", "Costo"))`, donde `Costo` también es una fórmula en Tarea.

### Funciones v1

- Operadores: `+ - * / % ** == != < <= > >= and or not`, y `??` para valores vacíos.
- Lógica: `c ? a : b` (en `expr`, `if` es palabra reservada, no una función), `empty(x)`.
- Texto: `concat`, `length`, `lower`, `upper`, `contains`, `replace`, `format`.
- Números: `round(x, n)`, `abs`, `floor`, `ceil`, `toNumber`.
- Fechas: `now()`, `dateAdd(d, n, unidad)`, `dateBetween(a, b, unidad)`, `formatDate(d, patrón)`.
- Listas y relaciones: `rel`, `len`, `count(lista, condición)`, `sum`, `avg`, `min`, `max`, `earliest`, `latest`, `first`, `last`, `join`, `filter`, `map`, `uniq`.
- Implementado el 2026-10-25 en `core/formula` (rama `GO-0000-obj-notes-formula-engine` de `chemworks/anytype-heart`), con tests de cada función.

Fuera de v1: más de un salto en una sola llamada (`rel` sobre `rel`), funciones definidas por el usuario, `let`, compatibilidad con fórmulas de Notion.

## 3. Modelo de datos

- [Cierto] Hoy los formatos de propiedad son `longtext … object, relations, map` (`pkg/lib/pb/model/models.pb.go`) y no hay ninguno calculado.
- Nuevo valor `RelationFormat_formula`. [Suposición] Número 200, lejos de los de upstream, para no chocar al rebasear.
- El objeto-relación (en Anytype cada propiedad es un objeto) suma dos detalles:
  - `relationFormula`: el texto de la fórmula, guardado con **claves internas** de relación y no con nombres. Así, renombrar "Horas" no rompe nada. El editor muestra los nombres.
  - `relationFormulaResultFormat`: `number | shorttext | date | checkbox | object`. Lo infiere el chequeo de tipos de `expr` y le dice al middleware cómo ordenar y filtrar, y al cliente cómo mostrar.
- El resultado se guarda en el objeto como un detalle **local**, que no viaja en los cambios sincronizados ni en el historial. [Cierto] El middleware ya hace eso con valores derivados, como backlinks (`core/block/editor/smartblock/links.go`, `updateBackLinks`) y `injectDerivedDetails` (`detailsinject.go`).
- **Spike del 2026-10-25, resultado:** [Cierto] `State.SetLocalDetail` acepta cualquier clave, incluida una de relación creada por el usuario. El valor aparece en `CombinedDetails()`, que es lo que guarda el indexador (`DocInfo.Details` en `smartblock.go`). No genera cambios `DetailsSet`, porque `makeDetailsChanges` solo mira los detalles sincronizados. Test: `core/block/editor/state/formula_spike_test.go` en la rama `GO-0000-obj-notes-formula-spike` de `chemworks/anytype-heart`.
- [Cierto] Al recargar un objeto, `injectLocalDetails` solo recupera del índice las claves del bundle (`bundle.LocalAndDerivedRelationKeys`). Las fórmulas se recalculan al abrir el objeto, así que no hace falta recuperarlas. Si eso resulta lento, se agrega la lista de claves de fórmula a esa recuperación.
- [Probable] Para recalcular objetos dependientes sin abrirlos existe `spaceIndex.UpdatePendingLocalDetails`, que ya usan las colecciones.
- Motivo: un valor derivado no debería generar cambios ni conflictos cuando un día vuelva la sync.

## 4. Cuándo se recalcula

1. **Cambia el objeto A** (cualquier detalle): se recalculan las fórmulas del Tipo de A sobre A.
2. **Dependientes en otras tablas**: [Cierto] el middleware ya sabe quién apunta a A (`spaceIndex.GetInboundLinksById`). Para cada objeto C que linkea a A, si el Tipo de C tiene una fórmula que usa `rel(...)` sobre una relación que contiene a A, se recalcula C.
3. **Cambia el texto de una fórmula**: se recalculan todos los objetos de los Tipos que tienen esa propiedad, en lote y en segundo plano.
4. **Ciclos**: el grafo de dependencias se arma con los textos de las fórmulas (qué propiedades lee cada una, y por qué relación). Un ciclo se rechaza al guardar la fórmula. En tiempo de ejecución hay además un límite de profundidad de 3 saltos.
5. `now()`: las fórmulas que la usan se recalculan al abrir la vista y una vez por día. No es tiempo real.

[Probable] El punto donde engancharse es el aplicado de estado del smartblock, el mismo lugar donde se inyectan los detalles derivados, más un servicio nuevo `core/formula` registrado en `core/anytype/bootstrap.go`, que es como se registran los servicios ([Cierto] `docs/Architecture.md`).

## 5. Errores

| caso | qué ve el usuario |
| --- | --- |
| sintaxis | el editor marca la posición y no deja guardar |
| propiedad que no existe | igual, con el nombre resaltado |
| tipo incorrecto (`"a" * 2`) | igual, lo detecta `expr` al compilar |
| división por cero, fecha inválida, lista vacía en `avg` | la celda muestra `#ERROR`, y el tooltip dice el motivo |
| ciclo entre fórmulas | no deja guardar y nombra las propiedades del ciclo |
| se borra una propiedad usada | la fórmula pasa a error hasta que se edite |

## 6. Cambios por repo

**anytype-heart (Go)**
- Protobuf: formato nuevo, dos detalles nuevos y RPC `RelationFormulaValidate(text, typeId) -> {ok, resultFormat, errors[]}`.
- `core/formula`: compilar con `expr`, caché de programas compilados por relación, grafo de dependencias y evaluador con `prop`/`rel`.
- Enganche de recálculo (sección 4) y lote al cambiar una fórmula.
- Orden y filtro: tratar `formula` según su `relationFormulaResultFormat`.

**anytype-ts (cliente)**
- [Cierto] Hay que agregar "Fórmula" en el menú de crear y editar propiedad (`src/ts/component/menu/dataview/relation/edit.tsx`).
- Editor de fórmula con autocompletado de nombres de propiedades y errores del RPC de validación.
- Celda de solo lectura que formatea según `resultFormat`.
- Sin confundirla con la "fórmula" de pie de columna que ya existe (`I.FormulaType` en `src/ts/interface/block/dataview.ts`). En la UI conviene que una se llame "Fórmula" y la otra "Resumen".

## 7. Casos de prueba de aceptación

1. Proyecto con 3 tareas de 2, 3 y 5 horas: `sum(rel("Tareas","Horas"))` da 10. Cambio una a 1 y el proyecto pasa a 6 sin reabrir nada.
2. Se desvincula una tarea y el total baja.
3. Ordenar la tabla de Proyectos por esa fórmula funciona.
4. Renombrar "Horas" a "Hs" no rompe la fórmula.
5. Fórmula A usa B y B usa A: no deja guardar.
6. 1.000 tareas en 50 proyectos: cambiar una tarea actualiza su proyecto en menos de 200 ms [Suposición: meta, no medido].

## 8. Orden de trabajo y tamaño

| paso | qué | tamaño [Suposición] |
| --- | --- | --- |
| 0 | Compilar y correr los dos repos | **hecho 2026-10-25**, ver `docs/dev-setup.md` |
| 1 | Spike: guardar un detalle local calculado para una relación de usuario | **hecho 2026-10-25, funciona** |
| 2 | Formato y protobuf, `core/formula` con `prop()` y validación | **hecho 2026-10-25**: motor, `rel()`, ciclos, `RelationFormat_formula = 200`, relaciones `relationFormula` y `relationFormulaResultFormat` |
| 3 | `rel()` y agregados, recálculo de dependientes, ciclos | 1 a 2 semanas |
| 4 | UI en el cliente: menú, editor, celda | 1 semana |
| 5 | Orden y filtro por fórmula, pruebas de la sección 7 | 3 a 5 días |

Total restante: unas 3 a 5 semanas [Suposición]. El riesgo del paso 1 ya está despejado; ahora el mayor riesgo es el recálculo de dependientes (paso 3).
