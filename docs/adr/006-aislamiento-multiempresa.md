# ADR-006: Aislamiento multiempresa

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-TRV-01, RF-TRV-07, RF-PLT-SEG-01, RF-AUT-07, RF-CNV-16, RF-PLT-WA-07
- **Depende de:** ADR-003, ADR-005
- **Condiciona a:** D-07 (tiempo real), D-08 (identidad y roles), D-09 (frontend)

## Contexto

El aislamiento entre negocios es **el primer requisito que no se negocia** (`CLAUDE.md` §8) y la condición de éxito del piloto: *"no hubo pérdida de mensajes ni fallo de aislamiento"* (`docs/01_definicion_mvp_v2.2.md` §11). `RF-TRV-01` lo exige verificado **con pruebas automáticas**.

La definición del MVP ya planteó la pregunta exacta, en §8.2 punto 3:

> "Si la API se conecta a la base con un usuario privilegiado, las políticas de seguridad por fila no la protegen. Hay que elegir un patrón: aplicar la identidad del usuario en cada transacción, o filtrar siempre por organización con pruebas automáticas de aislamiento."

Y en §8.3 fijó el modelo de datos: una base compartida, cada negocio con su `organization_id`, claves foráneas compuestas, y *"el backend obtiene la organización autorizada desde la sesión y la membresía del usuario. **No confía en el `organization_id` que envía el frontend**"*.

Hay que decidir **dónde se aplica el filtro y qué lo garantiza**.

### Restricciones que aplican

- Runtime: Hono en Cloudflare Workers, acceso a Postgres por Hyperdrive con node-postgres y un rol de base de datos dedicado (ADR-003).
- Dos personas con tiempo parcial. Lo que dependa de disciplina humana en cada consulta es lo primero que falla.
- Hay **tres clases de acceso** distintas, y conviene nombrarlas desde el principio porque el diseño tiene que servir a las tres:
  1. **Usuario de un negocio** (asesor o administrador): ve solo su organización.
  2. **Operador de plataforma**: administra todas las organizaciones, pero **nunca ve el contenido de las conversaciones** (`RF-TRV-07`, `RF-PLT-USO-04`, ADR-002).
  3. **El consumidor de la cola** (ADR-005): no tiene usuario; procesa eventos entrantes en nombre del sistema, y debe escribir aunque la organización esté suspendida (`RF-CNV-16`).

### Datos verificados

Verificados el 2026-10-15 contra la documentación oficial.

**Hyperdrive** — [How Hyperdrive works](https://developers.cloudflare.com/hyperdrive/configuration/how-hyperdrive-works/)

| Dato | Valor |
| --- | --- |
| Modo del agrupador | "The Hyperdrive connection pooler operates in **transaction mode**" |
| Reutilización | Las conexiones se comparten entre isolates de Workers |
| Estado de sesión | Al devolver la conexión al grupo, "the connection is `RESET` such that the `SET` commands will not take effect on subsequent queries" |
| Alcance de `SET` | Aplica dentro de una transacción o de la consulta que lo incluye; hay que fijarlo **en cada consulta o transacción** |
| Transacciones largas | Desaconsejadas: "It is not recommended to wrap multiple database operations with a single transaction to maintain the `SET` state" |
| Conexiones por invocación | Una sola invocación de Worker "may obtain multiple connections" |

**Supabase / PostgreSQL** — [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)

| Dato | Valor |
| --- | --- |
| Quién salta RLS | El rol `service_role`, que tiene el atributo `bypassrls`; también el dueño `postgres`, que "has `bypassrls`" |
| Roles personalizados | "You can also create new Postgres Roles which can bypass Row Level Security using the 'bypass RLS' privilege" |
| Vistas | "Views bypass RLS by default because they are usually created with the `postgres` user" |
| Alcance de RLS | "Because RLS is a Postgres primitive, it also protects your data when it is reached through third-party tooling" |

### Los dos hechos que definen la decisión

**Primero: RLS depende del rol de la conexión, no de la aplicación.** Si la API se conecta con `postgres` o con un rol que tenga `bypassrls`, las políticas **no se aplican en absoluto** y el aislamiento queda enteramente en manos del código. Esto es exactamente lo que advierte §8.2 punto 3. La buena noticia es que ADR-003 ya fijó un rol dedicado de privilegios mínimos; aquí se define qué significa eso.

**Segundo: Hyperdrive hace `RESET` al devolver la conexión y opera en modo transacción.** Esto tiene dos consecuencias opuestas:

- *A favor:* **el estado no se filtra entre peticiones.** El riesgo clásico de las conexiones agrupadas —que la organización de un usuario quede pegada a la conexión y la herede el siguiente— **no existe aquí**, porque la conexión se reinicia. Este es el riesgo que haría peligroso el patrón en otros entornos.
- *En contra:* **el contexto hay que fijarlo en cada transacción**, y una invocación puede obtener varias conexiones. No sirve "configurar la sesión al empezar". Cada operación carga su propio contexto.

El segundo punto es lo que descarta la variante ingenua del patrón de RLS y obliga a encapsularlo.

### Lo que realmente está en juego

La tentación es elegir entre RLS y filtrado en la aplicación, como si fueran alternativas. **No lo son, y tratarlas así es el error.**

- El filtrado en la aplicación depende de que **nadie olvide nunca** un `WHERE organization_id = ...`. Con dos personas, siete semanas y prisa, esa apuesta se pierde. Una consulta olvidada no falla: devuelve datos de otro negocio, en silencio.
- RLS sola no alcanza porque hay operaciones que legítimamente no tienen un usuario detrás: el consumidor de la cola (ADR-005) y la consola (ADR-002).

Lo que hay que decidir es **cuál es la última línea de defensa**, y la respuesta debe ser la que no dependa de que una persona se acuerde.

## Opciones consideradas

### Opción A: Filtrado en la aplicación, sin RLS

Todas las consultas filtran por `organization_id`, obtenido de la sesión. La base no tiene políticas.

- **Ventajas:** lo más simple de construir y de entender; sin complejidad de roles ni de políticas; funciona igual para usuarios, consola y consumidor, sin casos especiales; sin costo de rendimiento.
- **Desventajas:** **el aislamiento depende enteramente de la disciplina en cada consulta**; una consulta olvidada no produce un error, produce una fuga silenciosa; las pruebas automáticas de `RF-TRV-01` solo cubren los caminos que alguien pensó en probar; cualquier consulta nueva, escrita con prisa en noviembre, es una oportunidad de fallo; una inyección SQL o un error de lógica expone todo.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo, **riesgo alto** sobre el requisito que no se negocia.
- **Veredicto:** es lo que §8.2 punto 3 plantea como alternativa aceptable, pero deja el requisito más importante apoyado en que nadie se equivoque.

### Opción B: RLS como única defensa, con la identidad del usuario en cada transacción

La API se conecta con un rol sin `bypassrls` y fija el contexto del usuario en cada transacción; las políticas hacen el resto. El código no filtra.

- **Ventajas:** el aislamiento vive en la base, donde no se puede olvidar; una consulta mal escrita devuelve menos datos, nunca datos ajenos; protege también contra inyección SQL, porque la política se aplica igual.
- **Desventajas:** hay que resolver el acceso de la consola y del consumidor, que no tienen usuario; si alguien se conecta con otro rol —una migración, un script, una herramienta— las políticas no aplican y nadie se entera; **con Hyperdrive hay que fijar el contexto en cada transacción**, y una sola omisión deja la consulta sin contexto; depurar "¿por qué esta consulta no devuelve nada?" es incómodo.
- **Esfuerzo, costo y riesgo:** esfuerzo medio-alto. Riesgo medio: el fallo es hacia el lado seguro (no ver datos), que es la dirección correcta.

### Opción C: Las dos, en capas

RLS activo en todas las tablas con datos de negocio, **más** filtrado explícito por `organization_id` en el código. El filtrado es el camino normal; RLS es la red que atrapa lo que se olvidó.

- **Ventajas:** una consulta olvidada falla de forma segura en vez de filtrar datos; el código sigue siendo legible y depurable, porque el filtro está a la vista; las pruebas de `RF-TRV-01` pueden verificar **las dos capas por separado**, incluida la prueba de que RLS sola bastaría; el rendimiento no sufre, porque el filtro explícito deja al planificador usar los índices y la política queda como verificación redundante.
- **Desventajas:** hay que escribir y mantener políticas además del código; duplicación aparente, que puede llevar a que alguien "limpie" el filtro explícito creyéndolo redundante; hay que resolver igual los casos de la consola y del consumidor.
- **Esfuerzo, costo y riesgo:** esfuerzo medio-alto, concentrado al inicio —definir las políticas una vez— y bajo después. Riesgo bajo.

## Decisión

**Opción C: las dos capas.**

Razón: `RF-TRV-01` no es un requisito más. Es la condición que define si el piloto fue exitoso (`docs/01_definicion_mvp_v2.2.md` §11) y lo único que un cliente no perdona. Un requisito de ese peso no puede depender de que dos personas con prisa no olviden nunca una cláusula.

El filtrado explícito es el camino normal porque es legible y rápido. RLS es la red, y existe precisamente para el día en que el filtro falte.

Tres cosas hacen que el costo sea asumible aquí y no en otro contexto:

1. **El modelo de datos ya está preparado.** `CLAUDE.md` §9 fija que toda tabla con datos de negocio lleva `organization_id` y que las relaciones usan claves foráneas compuestas. Las políticas resultan casi todas idénticas y triviales de escribir.
2. **Hyperdrive hace `RESET`**, así que el riesgo clásico de fuga de contexto entre peticiones no existe.
3. **Es más barato ahora que después.** Agregar RLS a un esquema que ya está en producción, con consultas escritas suponiendo que no existe, es un trabajo mucho mayor.

### Los tres roles de base de datos

Esta es la parte que hace funcionar todo lo demás. Se crean **tres roles**, ninguno con `bypassrls`:

| Rol | Quién lo usa | Qué puede |
| --- | --- | --- |
| `app_user` | La API, para peticiones de usuarios de un negocio | Solo filas de la organización fijada en el contexto de la transacción |
| `app_platform` | La API, para acciones de la consola | Todas las organizaciones, **pero ninguna tabla con contenido de conversaciones** |
| `app_worker` | El consumidor de la cola (ADR-005) | Escribir mensajes y conversaciones de cualquier organización; sin lectura más allá de lo necesario para procesar |

Puntos que hacen que esto no sea decorativo:

- **Ninguno tiene `bypassrls`.** Si alguno lo tuviera, su capa de RLS sería inexistente y nadie lo notaría.
- **`app_platform` no obtiene su poder saltando RLS, sino porque no tiene permiso sobre las tablas de mensajes.** Esta es la diferencia clave: `RF-TRV-07` deja de depender de que la consola no consulte lo que no debe, y pasa a ser imposible a nivel de permisos. Una consulta de la consola contra la tabla de mensajes **falla**, no devuelve datos filtrados.
- **`app_worker` existe porque el consumidor no tiene usuario** y debe escribir aunque la organización esté suspendida (`RF-CNV-16`). Su política es por organización del evento, no por sesión.
- El rol `postgres` y cualquier rol con `bypassrls` quedan reservados para migraciones y administración. **La API nunca se conecta con ellos.**

### Cómo se fija el contexto

Por Hyperdrive, en **cada transacción**, con alcance local:

- Se abre la transacción, se fija la organización con alcance local (`set_local`), se ejecutan las consultas y se cierra.
- **No se usa configuración de sesión sin alcance local.** Aunque Hyperdrive hace `RESET`, depender de eso sería apoyar el requisito más importante en un detalle de implementación del proveedor.
- Las transacciones se mantienen **cortas**, como recomienda Hyperdrive: una operación lógica, no un conjunto.
- Este patrón vive **en un único lugar** del código —una función que recibe la organización y la operación a ejecutar—. Ninguna consulta abre transacciones por su cuenta. Es lo que hace que "fijar el contexto en cada transacción" no dependa de que alguien se acuerde.

### Qué fija esta decisión para el diseño

1. **Toda tabla con datos de negocio lleva `organization_id` y tiene RLS activo**, sin excepción. Esto incluye la **tabla de eventos crudos** de ADR-005, que contiene contenido de conversaciones y es fácil de olvidar porque no es una tabla "de producto".
2. **Las políticas se aplican también a `INSERT` y `UPDATE`**, no solo a `SELECT`. Una política de lectura sola permitiría escribir una fila en otra organización.
3. **Las vistas se declaran con `security_invoker`.** Las vistas saltan RLS por defecto al crearse con el dueño; una vista de métricas sin esta precaución sería una fuga. Dado que `CLAUDE.md` §9 propone resolver las métricas con vistas SQL, este punto es directamente relevante.
4. **La organización sale siempre de la sesión y de la membresía del usuario**, nunca de un parámetro del cliente (`docs/01_definicion_mvp_v2.2.md` §8.3). Si una ruta recibe un identificador de organización, se verifica contra la membresía antes de usarlo.
5. **El rol de plataforma no es una membresía de organización** (`docs/03_consola_plataforma_v1.1.md` §2, decisión 2). Un operador no "pertenece a todas"; usa otro rol de base de datos. La forma concreta de representarlo es D-08.
6. **Las claves foráneas son compuestas**, `(organization_id, id)` (`CLAUDE.md` §9). Esto impide a nivel de esquema que una conversación apunte a un contacto de otra organización —un error que RLS no atraparía.
7. **La unicidad del número de WhatsApp es global, no por organización** (`RF-PLT-WA-07`). Es la única restricción que cruza organizaciones a propósito, y por eso debe escribirse explícitamente: la regla general no la cubre.
8. **La suspensión no se implementa con RLS.** Una organización suspendida sigue recibiendo y guardando mensajes (`RF-CNV-16`); lo que se bloquea es el acceso de sus usuarios y el envío. Es una verificación de autorización, no de aislamiento. Mezclarlas haría que el consumidor dejara de poder escribir.

### Cómo se verifica (`RF-TRV-01`)

El requisito exige pruebas automáticas. Deben cubrir, como mínimo:

1. **Aislamiento de lectura:** un usuario de A no ve ninguna fila de B, en cada tabla con datos de negocio.
2. **Aislamiento de escritura:** un usuario de A no puede insertar ni modificar filas en B.
3. **RLS sola basta:** las mismas consultas **sin el filtro explícito** siguen sin devolver datos ajenos. Es la prueba que valida la red de seguridad; sin ella, la segunda capa es una suposición.
4. **Ningún rol de aplicación tiene `bypassrls`.** Se consulta el catálogo del sistema. Es una prueba de una línea que atrapa el error más grave posible.
5. **`app_platform` no puede leer mensajes.** Intentar leerlos debe fallar (`RF-TRV-07`).
6. **Toda tabla con `organization_id` tiene RLS activo.** Se consulta el catálogo y se compara. Esto atrapa la tabla nueva que alguien agregue en noviembre sin acordarse de la política —que es exactamente como ocurren estos fallos.
7. **Las vistas usan `security_invoker`.**

Las pruebas 4, 6 y 7 son las más valiosas por relación entre esfuerzo y protección: son consultas al catálogo del sistema, no escenarios, y detectan la clase de error que no aparece en ninguna prueba funcional.

## Consecuencias

### Más fácil

- Una consulta mal escrita falla de forma segura: devuelve menos, nunca datos ajenos.
- `RF-TRV-07` deja de depender de disciplina y pasa a ser un permiso que no existe.
- Las pruebas de `RF-TRV-01` pueden afirmar algo fuerte y verificable, no "revisamos las consultas".
- Una inyección SQL en una consulta de usuario sigue limitada a su organización.

### Más difícil

- Hay políticas que escribir y mantener, además del código.
- Tres roles de base de datos que crear, con sus permisos, y mantener coherentes entre entornos.
- Depurar una consulta vacía requiere preguntarse si falta contexto de transacción.
- **El filtro explícito parece redundante y alguien podría quitarlo.** Conviene dejarlo escrito en el código: RLS es la red, el filtro es el camino.
- Las migraciones deben acordarse de la política de cada tabla nueva. La prueba 6 es lo que vuelve ese olvido visible.

### Qué hay que construir

- Tres roles de base de datos sin `bypassrls`, con permisos explícitos por tabla.
- Políticas de RLS de lectura y escritura en toda tabla con `organization_id`, incluida la de eventos crudos.
- Función única que abre transacción, fija el contexto de organización con alcance local y ejecuta la operación.
- Claves primarias y foráneas compuestas según `CLAUDE.md` §9.
- Restricción de unicidad global del número de WhatsApp (`RF-PLT-WA-07`).
- Vistas de métricas con `security_invoker`.
- Las siete pruebas automáticas de aislamiento.

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.2 punto 3: el punto abierto queda resuelto por este ADR. La respuesta es **ambos patrones**, no uno.
- `docs/01_definicion_mvp_v2.2.md` §8.3: agregar los tres roles de base de datos y la precisión de que la consola no accede a las tablas de mensajes por permisos, no por disciplina.

### Impacto en el cronograma

El reparto de `docs/01_definicion_mvp_v2.2.md` §9 ya asigna a **Franco** la base de datos y el aislamiento, y §10 reserva la semana del 26 al 30 de noviembre para pruebas de aislamiento. Esta decisión encaja en ese plan, pero **las políticas y los roles deben existir desde la primera migración**, no en la última semana. Si se dejan para el final, habrá consultas escritas suponiendo que no existen.

`docs/01_definicion_mvp_v2.2.md` §10 ya dice que el aislamiento **no se recorta** en la revisión del 16 de noviembre. Esta decisión no cambia eso.

## Qué revisar cuando el sistema crezca

- **Si una consulta de la bandeja se vuelve lenta:** verificar que los índices incluyan `organization_id` como primera columna. Con filtro explícito el planificador debería resolverlo bien; si no, el problema es el índice, no la política.
- **Si aparece una tabla nueva sin `organization_id`:** revisar si de verdad es global. Las tablas globales legítimas son pocas (organizaciones, operadores, catálogos).
- **Si un cliente exige aislamiento dedicado:** §8.3 ya prevé evaluarlo. Con RLS puesto, mover una organización a su propia base es más fácil, porque el límite ya está marcado en cada consulta.
- **Si se agrega un cuarto rol** (por ejemplo, solo lectura para soporte): debe entrar en las pruebas 4 y 5 el mismo día que se crea.
- **Al adoptar el tiempo real de Supabase (D-07):** sus políticas son un sistema aparte del RLS de las tablas. Hay que verificar el aislamiento del canal de tiempo real por separado; que las tablas estén protegidas no implica que el canal lo esté.
- **Si el equipo crece:** la prueba 6 —toda tabla con `organization_id` tiene RLS— pasa de útil a indispensable, porque ya no todos conocen la regla.
