# ADR-013: Cálculo y almacenamiento de las métricas

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-MET-01 a RF-MET-08, RF-EST-01, RF-EST-04, RF-TRV-01, RF-TRV-03, RF-TRV-07
- **Depende de:** ADR-003, ADR-006
- **Condiciona a:** —

## Contexto

Este ADR cubre el segundo hueco detectado al auditar el tablero: **`RF-MET-01` a `RF-MET-08` no aparecen en ninguna de las once decisiones anteriores**, pese a plantear problemas de diseño reales.

`CLAUDE.md` §9 propone como criterio *"métricas con consultas o vistas SQL; no hace falta un almacén de datos a este volumen"*, y ADR-006 anotó que las vistas necesitan `security_invoker` para no saltarse RLS. Pero **nadie verificó que ese criterio aguante los requisitos escritos**, y hay dos que no son triviales.

### Los cinco indicadores

| Requisito | Qué calcula |
| --- | --- |
| `RF-MET-01` | Conversaciones por mes: con al menos un mensaje en el período |
| `RF-MET-02` | **Tiempo de atención: mediana** (y promedio como dato secundario) de los minutos entre un mensaje entrante que espera respuesta y la primera respuesta **de una persona**. Las respuestas automáticas no cuentan |
| `RF-MET-03` | Nuevos leads: contactos creados en el período |
| `RF-MET-04` | **Tasa de conversión:** de los contactos creados en el período, el porcentaje que **está en estado Cliente al consultar** |
| `RF-MET-05` | Cantidad de contactos: total y desglose por estado |

Más: `RF-MET-06` (todo por negocio **y por asesor**), `RF-MET-07` (selector de período, incluido rango personalizado) y `RF-MET-08` (solo administradores).

### Las tres dificultades reales

**Primera: `RF-MET-02` es la métrica difícil, y conviene ver por qué.**

No es un promedio de tiempos de respuesta. Requiere, por cada conversación, identificar **qué mensajes entrantes esperaban respuesta** y emparejarlos con **la primera respuesta humana posterior**. Varios mensajes seguidos del contacto sin respuesta intermedia cuentan como una sola espera, no como varias. Y la mediana, a diferencia del promedio, **no se puede acumular**: hay que tener el conjunto de valores para ordenarlo.

**Segunda: `RF-MET-04` mira dos momentos distintos.** El denominador son los contactos creados **en el período**; el numerador, cuántos de ellos están en Cliente **hoy**. No es una foto del período: es una cohorte evaluada en el presente. Esto importa porque significa que **el mismo período consultado dos veces puede dar resultados distintos**, y eso es correcto, no un error.

Hay un dato que lo hace viable: `RF-EST-04` exige un historial de cambios de estado con quién, cuándo y de qué estado a cuál. Ese historial existe por requisito propio, y es lo que permite responder preguntas sobre estados en el tiempo.

**Tercera: el desglose por asesor necesita una definición que los requisitos no dan.** `RF-MET-06` pide todas las métricas por asesor, pero *"contactos creados"* (`RF-MET-03`) no tiene un asesor obvio: un contacto nace cuando escribe por primera vez, posiblemente antes de que alguien lo atienda. Hay que decidirlo, y decirlo.

### Restricciones que aplican

- Volumen: 500 conversaciones al mes por negocio (`CLAUDE.md` §8). A 12 meses de retención, unas **6.000 conversaciones y 60.000 mensajes por organización**.
- `RF-TRV-03`: la búsqueda responde en menos de 1 segundo con 5.000 contactos. No fija un número para las métricas, pero marca la expectativa.
- Las métricas **no pueden mostrar contenido de conversaciones** (`RF-TRV-07`).
- Aislamiento: ADR-006 exige `security_invoker` en las vistas, porque **las vistas saltan RLS por defecto**.
- Dos personas con tiempo parcial.

## Opciones consideradas

### Opción A: Consultas SQL directas, calculadas al abrir la vista

Cada indicador es una consulta sobre las tablas de producción, ejecutada cuando el administrador abre V30.

- **Ventajas:** nada que construir ni mantener más allá de las consultas; **siempre exacto**, sin desfase; un cambio de definición se corrige en un solo lugar y aplica al histórico; `RF-MET-07` con rango personalizado funciona sin esfuerzo, porque la consulta recibe las fechas; ningún dato duplicado que pueda desincronizarse.
- **Desventajas:** cada apertura de la vista ejecuta varias consultas de agregación; `RF-MET-02` recorre los mensajes del período, que al año son decenas de miles por organización; sin índices adecuados puede tardar.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo. Riesgo bajo al volumen descrito, a verificar con datos reales.

### Opción B: Tablas de resumen precalculadas

Una tarea periódica consolida los indicadores por día y asesor; la vista lee el resumen.

- **Ventajas:** la vista responde de inmediato sin importar el histórico; la carga es predecible.
- **Desventajas:** **la mediana no se puede precalcular por día y luego combinar** —la mediana de las medianas no es la mediana—, así que `RF-MET-02` necesitaría guardar los valores individuales, que es casi tanto como consultar las tablas; `RF-MET-04` mira el estado actual, así que un resumen histórico no sirve; el rango personalizado de `RF-MET-07` obliga a recombinar días; hay que construir la tarea, manejar sus fallos y rellenar huecos; **los datos se desincronizan** si la tarea falla, y una métrica incorrecta es peor que una lenta porque nadie la nota; un cambio de definición exige recalcular el histórico.
- **Esfuerzo, costo y riesgo:** esfuerzo alto, riesgo medio.
- **Veredicto:** resuelve un problema de rendimiento que a este volumen **no está demostrado que exista**. Contradice la regla 9 de `CLAUDE.md` §4.

### Opción C: Almacén de datos aparte

Exportar a un sistema analítico.

- **Desventajas:** otro proveedor que montar y operar; duplica datos sensibles de salud fuera de la base principal, con todo lo que eso implica bajo la Ley 1581; `CLAUDE.md` §9 ya dice explícitamente que no hace falta a este volumen.
- **Veredicto:** descartada.

## Decisión

**Opción A: consultas SQL directas, calculadas al abrir la vista**, con índices pensados para ellas y una medición explícita antes del piloto.

Razón:

1. **El volumen no justifica otra cosa.** 60.000 mensajes por organización al año es poco para Postgres con los índices correctos. Precalcular sería optimizar sin medir.
2. **Dos de los cinco indicadores no se pueden precalcular bien.** La mediana de `RF-MET-02` no es combinable por día, y `RF-MET-04` depende del estado actual. Un diseño precalculado terminaría consultando las tablas igual para esos dos, con la complejidad añadida de mantener los otros tres.
3. **Una métrica desincronizada es peor que una lenta.** Un número incorrecto en una pantalla de métricas no se nota, y las decisiones del negocio se toman sobre él.
4. **Es reversible.** Si al medir resulta lento, agregar un resumen después es trabajo acotado. Al revés —quitar un sistema de resúmenes ya construido— no lo es.

### Dónde viven las consultas

- **Vistas SQL** para la lógica de cada indicador, como propone `CLAUDE.md` §9. Dejan la definición en un solo lugar, legible y probable con SQL.
- **Todas con `security_invoker`** (ADR-006). Sin eso, una vista se ejecuta con los permisos de su dueño y **devolvería datos de todas las organizaciones**. Es el error más grave posible en este ADR, y es silencioso.
- **El filtro explícito por organización se mantiene** también aquí, como en el resto del sistema (ADR-006): la vista recibe la organización, y RLS es la red.
- **La API las consulta con el rol `app_user`** y el contexto de organización de la transacción, igual que cualquier otra consulta. No hay camino especial para las métricas.
- **`app_platform` no puede leerlas**, porque no tiene permiso sobre las tablas de mensajes (ADR-006). Coherente con `RF-TRV-07` y `RF-PLT-USO-04`: las métricas de un negocio son del negocio, no de Numi.

### Definiciones que esta decisión fija

Los requisitos dejan ambigüedades que hay que resolver antes de implementar. Estas definiciones deben reflejarse en los textos de ayuda que `RF-MET` pide en la vista (§6.13, "cada indicador tiene un texto que explica cómo se calcula"):

| Punto | Definición |
| --- | --- |
| **Espera de respuesta** (`RF-MET-02`) | Un mensaje entrante inicia una espera **solo si no hay otra espera abierta** en esa conversación. Varios mensajes seguidos del contacto cuentan como una sola |
| **Respuesta que cierra la espera** | El primer mensaje saliente **enviado por una persona** después del entrante. Se excluye lo automático (`RF-MET-02` lo dice; hoy no hay respuestas automáticas en el MVP, pero la definición debe quedar lista) |
| **Esperas sin respuesta** | No entran en el cálculo. Una conversación sin responder no es un tiempo de atención infinito: es otra cosa, y ya se mide aparte en la consola |
| **Asesor de una conversación** (`RF-MET-06`) | El responsable **al momento de la consulta**. Si se reasignó, la conversación cuenta para el responsable actual. Es lo simple y lo que coincide con la vista de la bandeja |
| **Asesor de un contacto creado** (`RF-MET-03` por asesor) | El responsable de su conversación, derivado como indica `CLAUDE.md` §9. Un contacto sin responsable cuenta solo en el total del negocio |
| **Tasa de conversión** (`RF-MET-04`) | Contactos creados en el período que **hoy** están en Cliente, dividido por los creados en el período. El mismo período puede dar valores distintos en fechas distintas: es correcto |
| **Comparación con el período anterior** (§6.13) | El período inmediatamente anterior de la misma duración |
| **Zona horaria** | Los límites del período se calculan en hora de Bogotá, no en UTC (`RF-TRV-05`). Un "mes" termina a medianoche de Bogotá |

El último punto es de los que se pasan por alto y producen números mal por un día entero en los bordes del período.

### Índices necesarios

Las consultas de métricas marcan qué índices hacen falta. Todos empiezan por `organization_id`, como exige el filtro de ADR-006:

- Mensajes por `(organization_id, conversation_id, sent_at)` — es el que sostiene `RF-MET-02` y el orden de la conversación (`RF-CNV-06`).
- Conversaciones por `(organization_id, last_message_at)`.
- Contactos por `(organization_id, created_at)` — `RF-MET-03` y `RF-MET-04`.
- Contactos por `(organization_id, status)` — `RF-MET-05`.

**Hay que medir con datos realistas antes del piloto**: 6.000 conversaciones y 60.000 mensajes en una organización, y cronometrar V30. Es una tarea de una hora que evita descubrir el problema con un cliente delante.

### Qué fija esta decisión para el diseño

1. **Las métricas se calculan al consultar.** Sin tareas periódicas, sin tablas de resumen, sin caché en el MVP.
2. **Toda vista de métricas usa `security_invoker`**, y entra en la prueba de catálogo de ADR-006.
3. **Las métricas nunca devuelven contenido.** Son cantidades, tiempos y porcentajes (`RF-TRV-07`).
4. **Solo administradores** (`RF-MET-08`), verificado en el servidor como cualquier otro permiso (ADR-007). La interfaz oculta la opción; la autoridad está en la API.
5. **El historial de cambios de estado** (`RF-EST-04`) es la fuente para cualquier pregunta sobre estados en el tiempo. No se derivan de un campo sobrescrito.
6. **Si una consulta de métricas tarda, se corrige con índices**, no agregando una capa de resúmenes. Esa es la última opción.

## Consecuencias

### Más fácil

- Los números son siempre exactos y no pueden desincronizarse.
- Un cambio de definición se corrige en la vista y aplica también al histórico.
- El rango personalizado de `RF-MET-07` no necesita nada especial.
- Sin piezas nuevas que operar.

### Más difícil

- `RF-MET-02` es una consulta no trivial que hay que escribir y probar con cuidado.
- Hay que medir el rendimiento con datos realistas, que es un trabajo que suele postergarse.
- Las definiciones de la tabla hay que mantenerlas coherentes entre la consulta y el texto de ayuda de la vista.

### Qué hay que construir

- Vistas SQL para los cinco indicadores, con `security_invoker`.
- La consulta de `RF-MET-02`, con el emparejamiento de esperas y respuestas descrito.
- Desglose por asesor para todas (`RF-MET-06`).
- Los cuatro índices listados.
- Rutas de la API con verificación de rol administrador.
- Textos de ayuda que expliquen cada cálculo, coherentes con las definiciones fijadas.
- Prueba de rendimiento con 6.000 conversaciones y 60.000 mensajes.
- Prueba de aislamiento: un administrador de A no ve números de B, con y sin el filtro explícito.

### Qué documentos hay que actualizar

- `docs/02_documento_maestro_v1.1.md` §5.5: incorporar las definiciones de la tabla como textos de ayuda de cada indicador.
- `docs/02_documento_maestro_v1.1.md` §5.2: precisar en `RF-MET-02` que las esperas sin respuesta no entran en el cálculo.

## Qué revisar cuando el sistema crezca

- **Si V30 tarda más de 2 segundos:** revisar los índices primero, el plan de ejecución después. Solo si ambos están bien, considerar resúmenes.
- **Al superar unas 20.000 conversaciones por organización** —unos tres años al volumen de referencia—: volver a medir. Es el punto donde `RF-MET-02` podría empezar a pesar.
- **Si aparecen respuestas automáticas:** la definición de "respuesta que cierra la espera" ya las excluye, pero hay que marcar el origen del mensaje para poder distinguirlas (el modelo de datos de §8 ya contempla el campo "origen").
- **Si los negocios piden exportar métricas:** es alcance nuevo; las consultas ya existen, falta el formato y el camino de descarga.
- **Si se pide comparar entre organizaciones** (para Numi, no para el negocio): choca con `RF-PLT-USO-04` y con el hecho de que `app_platform` no puede leer mensajes. Requeriría un diseño distinto, con datos agregados sin contenido, y debería decidirse como alcance nuevo.
- **Si un negocio cuestiona un número:** el historial de cambios de estado (`RF-EST-04`) permite reconstruir qué pasó. Conviene no borrarlo antes que los mensajes.
