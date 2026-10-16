Progreso del proyecto de arquitectura

Bitácora de sesiones. Claude agrega una entrada al cerrar cada sesión, la más reciente arriba.

Formato de cada entrada:

markdown

## AAAA-MM-DD — Título corto de la sesión

- **Se decidió o produjo:** ...
- **Quedó abierto:** ...
- **Siguiente paso:** ...

---

## Estado al 2026-10-15

**Las 13 decisiones están aprobadas.** D-01 el 15 de octubre tras ADR-001; D-02 a D-13 el mismo día. El entregable en curso es `docs/architecture/arquitectura.md`, con 2 de 12 secciones escritas.

| ADR | Decisión | Qué se eligió |
| --- | --- | --- |
| [001](adr/001-modelo-conexion-whatsapp.md) | D-01 Conexión de WhatsApp | Tech Provider como destino; piloto con números del portafolio de Numi |
| [002](adr/002-alcance-consola.md) | D-02 Alcance de la consola | Completa: 47 requisitos P0, 11 vistas |
| [003](adr/003-runtime-backend.md) | D-04 Runtime del backend | Hono en Cloudflare Workers |
| [004](adr/004-multimedia-mvp.md) | D-03 Multimedia | Se descargan, se guardan y se muestran |
| [005](adr/005-procesamiento-webhook.md) | D-05 Webhook | Webhook mínimo más Cloudflare Queues |
| [006](adr/006-aislamiento-multiempresa.md) | D-06 Aislamiento | RLS más filtrado explícito; tres roles de base de datos |
| [007](adr/007-identidad-y-roles.md) | D-08 Identidad | Supabase Auth solo como identidad; autorización en el backend |
| [008](adr/008-tiempo-real.md) | D-07 Tiempo real | Broadcast desde el servidor, con red de seguridad de 60 s |
| [009](adr/009-arquitectura-frontend.md) | D-09 Frontend | Aplicación de página única con Vite, servida desde Workers |
| [010](adr/010-archivos-r2.md) | D-10 Archivos | R2, servido por el Worker; sin URL firmadas |
| [011](adr/011-secretos-entornos-despliegue.md) | D-11 Secretos y despliegue | AES-GCM, Resend, dos entornos, Supabase Pro |
| [012](adr/012-envio-de-mensajes.md) | D-12 Envío de mensajes | Síncrono, con persistencia previa y reintento manual |
| [013](adr/013-metricas.md) | D-13 Métricas | Consultas SQL directas con `security_invoker` |

**Costo fijo mensual:** 30 USD (Workers 5 + Supabase Pro 25), más los mensajes de Meta y el dominio.

### Lo que bloquea o urge

| Qué | Por qué urge | Dónde |
| --- | --- | --- |
| **Verificación de negocio ante Meta** | Ruta crítica del proyecto y no la controla el equipo. Sin ella el piloto queda limitado a 2 clientes | ADR-001 |
| **Configurar Resend (SPF, DKIM, DMARC)** | Sin correo no hay invitación y **no hay alta de organizaciones**. La propagación de DNS no depende del equipo | ADR-007, ADR-011 |
| **Elegir la base de componentes accesibles** | Es lo que decide cuánto de las 31 vistas no hay que escribir a mano, con `RF-TRV-06` exigiendo accesibilidad real. Desbloqueado al cerrarse el framework el 2026-10-16 | ADR-009 |

**La etapa de arquitectura está cerrada (2026-10-16).** Las 14 decisiones aprobadas, el documento de arquitectura terminado, la revisión independiente de `CLAUDE.md` §11 hecha y sus siete hallazgos corregidos, los supuestos que importaban confirmados y los documentos fuente sincronizados. **Lo que queda no es trabajo de diseño.**

### Puntos abiertos

- **Base de componentes accesibles:** sin elegir. Es lo primero antes de construir vistas.
- **Riesgo de calendario:** 111 requisitos P0 y 31 vistas en siete semanas con dos personas a tiempo parcial. Cada ADR tiene su orden de repliegue, pero **nadie sumó el alcance total ni priorizó entre las listas de ADR-002 y ADR-004** — esa comparación es la que hará falta el 16 de noviembre. Aplazado por decisión de Franco el 2026-10-16.
- **Concentración en Supabase:** Postgres, Auth y Realtime en un proveedor con antecedentes de caídas. Asumido conscientemente; tabla de modos de falla en ADR-011.
- **Antes del piloto:** la lista de puesta en marcha de ADR-011, probar una restauración de copia de seguridad, y medir V30 con 6.000 conversaciones y 60.000 mensajes (ADR-013).

**Siguiente paso:** elegir la base de componentes accesibles y empezar el desarrollo.

---

## 2026-10-16 — Etapa de arquitectura cerrada

- **Se decidió o produjo:** confirmados los supuestos que faltaban y **cerrada la etapa de arquitectura** según los criterios de `CLAUDE.md` §7.

  **Franco confirma los dos supuestos que podían invalidar la sección 11:**
  - **20 mensajes por conversación.** Era el más influyente porque multiplica todos los volúmenes del sistema. Confirmado, los números de la sección 11 quedan intactos: 200.000 mensajes al mes con 20 negocios, 0,95 por segundo en el pico, 84× de margen contra el límite de Meta.
  - **3 a 10 usuarios por negocio.** Da ~60 asesores concurrentes contra un techo de 200 conexiones de Realtime, 3,3× de margen.

  **Los otros tres "supuestos" no eran tales**, y se reclasificaron en §1.8: los 20 negocios a 12 meses son el **tope duro del portafolio de Meta**, no una proyección; el umbral de 2 s de la vista de métricas se confirma **midiendo** (ADR-013 ya lo exige); y el techo de costo es un **criterio de juicio** para decidir si una pieza nueva se justifica, no un número de diseño.

  **Los cuatro pendientes del documento de arquitectura quedan cerrados:** revisión independiente hecha, supuestos confirmados, framework decidido y documentos fuente actualizados. El estado del documento pasa de *"Completo, pendiente de revisión independiente"* a **Terminado**.

- **Quedó abierto:**
  - **Elegir la base de componentes accesibles.** Es lo primero antes de construir vistas y lo que decide cuánto trabajo de interfaz hay que hacer a mano.
  - **El riesgo de calendario, aplazado por decisión de Franco.** Siguen siendo 111 requisitos P0 y 31 vistas en siete semanas con dos personas a tiempo parcial, y **nadie sumó el alcance total ni priorizó una sola lista de repliegue** entre ADR-002 y ADR-004. Es la comparación que hará falta en la revisión del 16 de noviembre.
  - **La lista de puesta en marcha de ADR-011** y la verificación de negocio ante Meta, que es ruta crítica y no la controla el equipo.

- **Siguiente paso:** elegir la base de componentes y empezar el desarrollo.

---

## 2026-10-16 — ADR-009 cerrado: React con Vite

- **Se decidió o produjo:** **Franco confirma React con Vite** para el frontend. Era el último punto abierto de las 14 decisiones: ADR-009 se había aprobado el 15 de octubre comparando arquitecturas sin comparar frameworks, y la revisión independiente de esta misma fecha lo marcó como inconsistencia porque `CLAUDE.md` §9 ya afirmaba React mientras el ADR lo dejaba sin resolver. **Las dos fuentes vuelven a coincidir.**

  El razonamiento que ya estaba escrito en el ADR y que la decisión confirma: con 31 vistas, siete semanas y `RF-TRV-06` exigiendo accesibilidad real, lo que pesa no es el framework sino **cuántos componentes se pueden no escribir**, y React es donde más oferta hay de bases de componentes accesibles ya resueltas.

  Actualizados `docs/adr/009-arquitectura-frontend.md` (el punto abierto se reemplaza por la decisión), `arquitectura.md` y las referencias vigentes de esta bitácora.

- **Quedó abierto:** **elegir la base de componentes accesibles**, que pasa a ser lo primero antes de construir las vistas. Es ahora lo que decide cuánto trabajo de interfaz hay que hacer a mano.

- **Siguiente paso:** elegir la base de componentes y empezar el desarrollo.

---

## 2026-10-16 — Revisión independiente y corrección de siete huecos en los ADR

- **Se decidió o produjo:** la **revisión independiente que pedía `CLAUDE.md` §11**, hecha en una sesión que no participó en el diseño, sobre los 14 ADR, `arquitectura.md` y el diagrama de Archify. Se revisaron los ADR en tres bloques por revisores separados, y cada hallazgo grave se verificó contra el texto de los documentos antes de darlo por válido.

  **Veredicto: la arquitectura es sólida y escalable. No hay rediseño pendiente.** Los hallazgos son huecos de especificación —cosas que el ADR no decía, no cosas que decidiera mal—. **Ninguna decisión se cambió en esta sesión.**

  **Las siete correcciones aplicadas:**

  1. **ADR-005, pérdida de mensajes (`RF-TRV-02`).** El webhook descartaba con 200 los eventos de un número no asociado a ninguna organización. En el modo transitorio de ADR-001 los números se conectan a mano, así que existe una ventana entre que el número recibe mensajes en Meta y que el operador lo conecta: **un 200 le dice a Meta que no reintente, y esos mensajes se perdían**. Ahora el evento se guarda con la organización sin resolver y se reprocesa al conectar el número. Regla nueva: **el webhook nunca descarta un evento cuya firma es válida.**
  2. **ADR-006, política de RLS de cada rol.** El ADR decía que `app_worker` filtra "por organización del evento" sin decir cómo llega esa organización a la política; escrito así, la implementación natural es `USING (true)`, que lo vuelve equivalente a `bypassrls` **en el camino por el que pasan todos los mensajes entrantes**. Se especifica que los tres roles usan la misma variable de contexto y que la diferencia es la fuente del valor, que la política falla cerrado, y que las tablas llevan `FORCE ROW LEVEL SECURITY`. Se agregaron tres pruebas. **Era el hallazgo que no se podía retrofitear**: es esquema.
  3. **ADR-014, agente ante entrada adversaria.** El ADR aislaba bien el agente pero no decía nada sobre su comportamiento ante entrada hostil, y el agente es la única pieza que **actúa sobre texto que escribe un desconocido no autenticado**. Se agrega el punto 6a: el texto del contacto es dato y nunca instrucción, el agente no tiene herramientas, tope de 10 turnos consecutivos (antes un intercambio con otro automatismo no terminaba nunca), tope de gasto por organización (antes nada separaba los 6 USD estimados de una factura abierta), y la verificación de suspensión **antes** de llamar al modelo, no al enviar — si no, una organización suspendida ya habría mandado su contenido a un tercero.
  4. **ADR-012, mensajes atascados en `enviando`.** Si el Worker moría entre guardar el estado y actualizarlo, el mensaje quedaba en `enviando` para siempre: indicador infinito y, como no es un estado de error, **el asesor ni siquiera podía reintentar**. Se fija que `enviando` es transitorio con plazo de 2 minutos y un barrido lo pasa a error de resultado incierto.
  5. **ADR-010, límite de memoria de Workers.** Era el único punto de la arquitectura que asumía comportamiento de un proveedor **sin cita**, contra `CLAUDE.md` §4 regla 4. Verificado: **128 MB por isolate, no por invocación**, así que un documento de 100 MB cargado en memoria puede tumbar invocaciones vecinas. Se fija que todo archivo se mueve en flujo y que un `arrayBuffer()` en ese camino es un error de corrección.
  6. **ADR-005, reintentos de archivos.** "Se reintenta dentro de los 7 días" no salía de los 100 reintentos: con el retroceso por defecto se agotan en horas y el archivo se perdía con seis días de margen sin usar. Se fija retención de 7 días y retroceso con tope. Se agregaron tres límites verificados de Queues (espera de lote 60 s, 250 invocaciones concurrentes, 15 min de duración).
  7. **ADR-014, contradicción interna.** La lista de actualizaciones pedía registrar "bajo ZDR" mientras la investigación del mismo ADR concluía que el arreglo correcto es **preparación para HIPAA, no ZDR**. Corregido para que no se implemente el arreglo equivocado.

  **Pieza transversal que apareció:** las correcciones 4 y 6, más la retención de ADR-005 y el borrado de archivos de ADR-004, necesitan todas un **trabajo programado (Cron Trigger)**, que ningún ADR mencionaba — Workers no tiene procesos largos. **Es uno solo para los cuatro casos**, no una pieza por cada uno.

- **Quedó abierto:**
  - **La elección de framework de ADR-009** fue la única corrección que la revisión no pudo aplicar sola, porque dependía de un dato que no está en el repositorio. **Franco la cerró el mismo día: React con Vite** (ver la entrada siguiente). El ADR queda cerrado y `CLAUDE.md` §9 deja de ir por delante de su decisión.
  - **El riesgo de calendario sigue sin cuantificar:** ningún ADR suma el alcance total ni prioriza entre las listas de repliegue de ADR-002 y ADR-004. Son 111 requisitos P0 con dos personas a tiempo parcial, y esa comparación es justo la que hará falta el 16 de noviembre.
  - Las correcciones de esta sesión agregan trabajo a `docs/01` a `docs/03` (estado transitorio de eventos sin organización, `enviando` con plazo, topes del agente). No aplicadas.
  - Sin cambios en lo demás: los cinco supuestos de §1.8, la corrección de `max_size`, el BAA de Anthropic y la verificación de negocio ante Meta.

- **Siguiente paso:** elegir la base de componentes accesibles y empezar el desarrollo.

---

## 2026-10-15 — Documentos fuente actualizados con las 14 decisiones

- **Se decidió o produjo:** aplicadas las **~35 actualizaciones** que los 14 ADR listaban en sus consecuencias y que no se habían tocado desde el 14 de octubre. Era el pendiente más grande del repositorio: los documentos fuente seguían describiendo un sistema que las decisiones ya habían cambiado.

  **`docs/01_definicion_mvp_v2.2.md`:**
  - **§3, segmento:** pasa de "clínicas estéticas" a **inmobiliarias, clínicas estéticas y odontologías con el mismo producto** (ADR-001), con la consecuencia explícita de que dos de los tres manejan datos de salud.
  - **§6, Fuera del MVP:** **quitados los agentes de IA y la visualización de multimedia**, que entraron por ADR-014 y ADR-004. Se deja registrado que el agente contradice la exclusión anterior y por decisión de quién. Agregada la consecuencia de que, sin plantillas, una conversación con la ventana vencida no se puede retomar.
  - **§8.2, stack:** reescrita la tabla con lo decidido —Hono, Vite, tres colas, Realtime, Resend, Anthropic— y **los cuatro puntos técnicos del brief marcados como resueltos**, cada uno con su ADR y su respuesta.
  - **§8.3:** agregados los tres roles de base y la precisión de que la consola no accede al contenido **por permisos, no por disciplina**.
  - **§8.4, retención:** adjuntos deja de ser condicional, se agrega la tabla de eventos recibidos con 30 días, y se precisa el cifrado de credenciales con clave fuera de la base.
  - **§12, riesgos:** cerrados tres (modelo de conexión, complejidad del stack, multimedia) y agregados cuatro nuevos: configuración de Resend, calendario con el agente, concentración en Supabase y contenido hacia un tercero.

  **`docs/02_documento_maestro_v1.1.md`:** reescrito `RF-CNV-10` —ahora cubre solo video, sticker, ubicación y contacto—, agregados `RF-CNV-10a` a `10e` (adjuntos), precisados `RF-CNV-09` (progresión de estados y reintento manual), `RF-CNV-15` (el aviso no transporta contenido) y `RF-MET-02` (con el agente mide cuánto tardó una persona en entrar), y agregados `RF-MET-09` y `RF-CFG-04` a `RF-CFG-08` para el agente. **El CRM pasa de 63 a 73 requisitos.**

  **`docs/03_consola_plataforma_v1.1.md`:** anotado que **P23 es vista de soporte** y que la vía normal de alta de asesores es el CRM —lo que vuelve `RF-USR-*` irrecortable—, que el doble factor se resuelve con el TOTP del proveedor (confirmado disponible y gratuito), y que `RF-PLT-WA-08` sube a P0 al obtener el rol de Tech Provider, con la nota de que la v2 del registro integrado quedó obsoleta.

  **`CLAUDE.md`:** §9 con el stack decidido y su ADR por fila; §8 con la excepción del agente a la regla de datos sensibles.

- **Quedó abierto:**
  1. **Revisión independiente en sesión nueva** (`CLAUDE.md` §11) — ahora cubre también estos documentos actualizados.
  2. Confirmar los cinco supuestos de `arquitectura.md` §1.8.
  3. Elección de framework del frontend (ADR-009).
  4. El diagrama de Archify, con el candidato validado.
  5. **Las vistas V11 y V50 de `docs/02` §6 no se reescribieron en detalle**: se agregaron los requisitos, pero la descripción de cada vista todavía no refleja las variantes de mensaje con adjunto ni la configuración del agente. Es trabajo de detalle de interfaz, no de arquitectura.

- **Siguiente paso:** revisión independiente en sesión nueva.

---

## 2026-10-15 — Sección 12: trazabilidad. Documento de arquitectura completo

- **Se decidió o produjo:** escrita la **sección 12**, última pendiente. **`docs/architecture/arquitectura.md` tiene sus 12 secciones**, sobre ADR-001 a ADR-014, todas aprobadas.

  **Cobertura verificada:** los 63 requisitos del CRM, los 47 P0 de la consola y el nuevo `RF-MET-09` tienen componente asignado. Ningún requisito quedó sin dónde cumplirse.

  **Corrección de hecho encontrada al verificar los totales contra `docs/03`:** ADR-002 decía *"44 requisitos en P0 y 8 en P1"*; el documento fuente dice **47 P0 y 5 P1**. ADR-002 contaba las dos vistas P1 (P10 y P30) como si fueran requisitos. La decisión no cambia —consola completa— pero el número sí: **el alcance de la consola es 3 requisitos mayor de lo que decía el ADR**, lo que refuerza su propia advertencia de calendario. Corregido en ADR-002 y en la tabla de estado de esta bitácora; las entradas históricas se dejan como registro de lo que se dijo ese día.

  **La pregunta inversa de `CLAUDE.md` §11 —qué se diseñó sin requisito que lo pida— se respondió explícitamente**: siete piezas sin requisito directo (las tres colas, Hyperdrive, la tabla de eventos crudos, los tres roles y la consulta de respaldo de 60 s), y las siete se justifican. Ninguna sobra. Las tres colas eran el caso que más merecía revisión porque parecen complejidad; existen porque tienen tolerancias de fallo incompatibles.

  **Observación sobre los nueve requisitos innegociables:** los números 1 (aislamiento), 2 (sin duplicados) y 7 (datos sensibles fuera de la consola) son **estructurales** — dependen de restricciones y permisos de la base, no de disciplina. Son los tres que un error de código no puede romper.

  **Tres requisitos cambiaron de significado** por decisiones posteriores y los documentos fuente deben actualizarse: `RF-CNV-10` (ADR-004), `RF-MET-02` (con el agente en primera línea mide "cuánto tardó una persona en entrar", no "cuánto esperó el contacto") y `RF-CFG-*`. Más los nuevos `RF-CNV-10a` a `10e` y `RF-MET-09`.

- **Quedó abierto** — lo que falta para dar el documento por terminado según `CLAUDE.md` §7:
  1. **Revisión independiente en sesión nueva** (`CLAUDE.md` §11). La sesión que escribió el documento no puede hacerla.
  2. **Confirmar los cinco supuestos de §1.8**; el más influyente son los 20 mensajes por conversación.
  3. **Elección de framework del frontend** (ADR-009), que depende de qué conoce Anthony.
  4. **Actualizar `docs/01` a `docs/03`** con lo que cada ADR lista en sus consecuencias. Ninguna aplicada todavía.
  5. **El diagrama de Archify**, con el candidato ya validado y pendiente solo de la escritura del HTML.

- **Siguiente paso:** revisión independiente del documento en una sesión nueva, con la instrucción de `CLAUDE.md` §11.

---

## 2026-10-15 — Sección 10: observabilidad

- **Se decidió o produjo:** escrita la **sección 10 de `arquitectura.md`**: qué se registra, qué se vigila y qué avisa, con el objetivo declarado de operar un piloto de una semana con dos personas, no de construir un sistema de monitoreo.

  **Lo que la sección agrega sobre ADR-011**, que se escribió antes del agente: qué vigilar de esa pieza. Dos señales —cola acumulada y tasa de escaladas—, **ninguna como alerta**. El agente es la única pieza del sistema cuyo mal funcionamiento no es una urgencia, y eso es consecuencia directa del diseño de ADR-014: si falla dos veces escala a una persona, de modo que su modo degradado es exactamente el comportamiento del sistema sin agente. Lo que sí hay que vigilar del agente es el costo, que es la única partida variable.

  **Precisión sobre el umbral de la alerta de webhook caído.** Las 2 horas no son arbitrarias: con 500 conversaciones al mes por negocio, dos horas sin eventos es anómalo pero no imposible un martes tranquilo. Un umbral más corto produciría falsas alarmas, y **una alerta que suena sin motivo se empieza a ignorar** — que es la peor forma de perder la única alerta que protege algo irreversible.

  La sección insiste en que la regla de no registrar contenido tiene que ser estructural —las funciones de registro aceptan identificadores, no objetos— porque el fallo no ocurre por mala fe: ocurre a las dos de la mañana, depurando, con un registro del evento entero.

- **Quedó abierto:** sin cambios respecto de la entrada anterior.

- **Siguiente paso:** sección 12, trazabilidad: cada requisito P0 enlazado con el componente que lo cumple. Es la última sección pendiente y la que cierra el criterio de terminado de `CLAUDE.md` §7.

---

## 2026-10-15 — Sección 9: despliegue

- **Se decidió o produjo:** escrita la **sección 9 de `arquitectura.md`**: dos entornos, cómo se despliega, cómo se revierte y qué configuración no vive en el código.

  **Precisión que la sección aporta y que ADR-011 no fijaba: el orden de despliegue cuando la API y la aplicación cambian juntas.** Migraciones primero, luego la API, luego la aplicación. La razón: una aplicación nueva pidiendo un endpoint que todavía no existe da errores inmediatos y visibles; una API nueva sirviendo a una aplicación vieja no, porque los cambios son aditivos. Se despliega en el orden que deja la ventana de incompatibilidad del lado seguro.

  **La asimetría que explica la disciplina de migraciones:** la reversión de Cloudflare cubre el código, no el esquema. Como las migraciones no se revierten, se diseñan para no necesitarlo —solo aditivas—, y eso es justamente lo que vuelve segura la reversión del código: un código anterior funciona sobre un esquema con columnas de más, pero no sobre uno al que le falta algo.

  La sección consolida en una sola tabla **la configuración que no está en el código** —siete verificaciones por entorno—, incluida la corrección de `max_size` de Hyperdrive que encontró la auditoría de escalabilidad. Las tres primeras son las graves y comparten una propiedad: **su fallo es silencioso**. Un canal público de Realtime no da error, simplemente deja de aislar.

  Queda también el **orden de puesta en marcha** por dependencias externas que el equipo no controla: Resend primero (la propagación de DNS no depende de nadie del equipo y sin correo no hay alta de organizaciones), luego la verificación de Meta, después el BAA de Anthropic, y las políticas de RLS **desde la primera migración** — ADR-006 advirtió que si se dejan para el final habrá consultas escritas suponiendo que no existen.

- **Quedó abierto:** sin cambios respecto de la entrada anterior.

- **Siguiente paso:** sección 10, observabilidad.

---

## 2026-10-15 — Sección 8: modos de falla

- **Se decidió o produjo:** escrita la **sección 8 de `arquitectura.md`**. El criterio que la ordena es distinguir lo que **se degrada** —molesta y se recupera solo— de lo que **se pierde** —no vuelve—. Casi todo en este sistema es lo primero; la sección se concentra en identificar dónde está lo segundo.

  **El hallazgo central, que ADR-005 ya había anotado y aquí queda con diagrama propio: los reintentos de Meta duran 7 días y la vigencia del identificador de archivo también.** No es una coincidencia útil. Si el sistema está caído varios días, los mensajes se recuperan cuando Meta reintenta, pero **los archivos de los primeros días ya habrán caducado**. Una caída larga pierde archivos aunque no pierda mensajes, y es el único camino del sistema con pérdida irreversible. De ahí sale la prioridad de la alerta de webhook caído: tiene un reloj de 7 días, no es un aviso de cortesía.

  **Los fallos parciales son los difíciles**, y la sección los trata aparte: un servicio caído se nota, uno que funciona a medias no. El caso delicado es el tiempo límite al enviar —el sistema no sabe si el mensaje salió—, que ADR-012 resolvió dejando la decisión al asesor informado. El agente no tiene esa salida porque su cola reintenta sola, de ahí la clave de idempotencia de ADR-014.

  **Señalado lo contraintuitivo: el fallo más disruptivo no es el más técnico.** Si Resend falla, todo el sistema funciona perfectamente y el negocio se detiene, porque no entran clientes nuevos. Es la pieza más modesta de la arquitectura y la más crítica para la operación comercial; su mitigación —el enlace manual de `RF-PLT-USR-04`— ya estaba en los requisitos por otra razón.

  La sección cierra con el orden de diagnóstico en un incidente —primero el webhook, porque es el único reloj que corre— y con lo que el diseño deliberadamente no cubre.

- **Quedó abierto:** sin cambios respecto de la entrada anterior.

- **Siguiente paso:** sección 9, despliegue.

---

## 2026-10-15 — Secciones 6 y 7: flujos críticos, multiempresa y seguridad

- **Se decidió o produjo:** escritas las **secciones 6 y 7 de `arquitectura.md`**.

  **Sección 6, flujos críticos.** Los cinco diagramas de secuencia que exige `CLAUDE.md` §8, cada uno con qué puede fallar y qué requisito se juega. Recorrerlos de punta a punta confirmó tres cosas —los tres escritores aparecen en los flujos y no hay un cuarto camino oculto; la asignación se dispara desde tres flujos y en los tres pasa por el mismo módulo; ningún flujo lleva contenido a la consola, los registros o el tiempo real— y **encontró un hueco**: el flujo de mensaje entrante crea contacto, conversación y mensaje, y puede asignar, y **todo eso debe ocurrir en una sola transacción**. Si se guardara el mensaje y fallara la asignación, quedaría una conversación en un estado que ningún flujo contempla. Queda explícito en §6.6.

  **Sección 7, multiempresa y seguridad.** Las cinco capas de aislamiento, con una observación que vale la pena retener: **las capas 2 y 3 (filtro explícito y RLS) se cubren mutuamente, pero las capas 4 y 5 (permisos de rol y políticas del canal) no tienen respaldo** — son las más delicadas y hay que verificarlas por separado.

  La decisión más importante de la sección es la que ya tomó ADR-006 y aquí queda dicha con todas las letras: **`app_platform` no obtiene su poder saltando RLS, sino careciendo de permiso sobre las tablas de mensajes.** Una consulta de la consola contra mensajes **falla con un error de permisos**; no devuelve datos filtrados ni un conjunto vacío. Eso convierte `RF-TRV-07` de promesa en imposibilidad.

  La sección documenta además el recorrido completo de un dato sensible —una foto clínica, de Meta a R2 y al navegador—, dónde vive cada secreto, las siete pruebas automáticas de `RF-TRV-01`, y **dos advertencias de configuración que no viven en el código**: el "Allow public access" de Realtime y los roles sin `bypassrls`. Un canal público no da error, simplemente deja de aislar.

  Registrada también la excepción de ADR-014: el agente de IA rompe por primera vez la propiedad de que el contenido no sale del sistema, con el detalle de qué viaja y qué no (el identificador del contacto no viaja) y la condición de la preparación para HIPAA antes de conectar un cliente real.

- **Quedó abierto:**
  - **El diagrama de Archify no se pudo generar en esta máquina.** El candidato está escrito y **pasa la validación sin errores de contenido ni de layout** (15 componentes, 17 conexiones, 2 límites, 4 fichas), pero la entrega del HTML falla con `delivery/ownership-lost` y `finalize/receipt-publication`. Archify escribe de forma atómica y verifica que nadie toque sus archivos entre escribir y leer; esa comprobación falla siempre aquí y el archivo de bloqueo queda en 0 bytes. Se reprodujo en el repositorio y en `%TEMP%`, con `finalize`, `validate` y `deliver`, así que no depende de la carpeta de destino. Causa más probable: el escáner en tiempo real de Windows Defender. `archify doctor` reporta la instalación sana. Todo queda en `.archify/architecture-numi-crm-20261015-1200/` con un README que trae el comando exacto y el diagnóstico. **Por decisión de Franco se deja el diagrama para el final.**
  - Sin cambios en lo demás: los cinco supuestos de §1.8, la corrección de `max_size`, el BAA, la elección de framework del frontend y las actualizaciones pendientes en `docs/01` a `docs/03`.

- **Siguiente paso:** sección 8, modos de falla.

---

## 2026-10-15 — Sección 5: componentes del frontend

- **Se decidió o produjo:** escrita la **sección 5 de `arquitectura.md`**, componentes del frontend (C4 nivel 3), con diagrama Mermaid. Cubre cómo se organizan las 31 vistas —18 del CRM y 13 de la consola, verificadas contra `docs/02` y `docs/03`— en una sola aplicación con dos áreas, y cómo se resuelven sesión, datos, estado y tiempo real.

  **La regla que ordena la sección: la interfaz no decide permisos, los refleja.** De ahí se sigue el punto que conviene decir sin rodeos porque es un error frecuente: **que el código de la consola llegue al navegador no otorga ningún permiso.** La carga diferida del área de plataforma es rendimiento, no seguridad; la protección real es que el servidor verifica el rol en cada petición.

  **Lo que no es obvio y hay que construir bien:** sesión expirada, usuario desactivado y organización suspendida pueden ocurrir **a mitad de sesión**, no solo al entrar, porque ADR-007 verifica en cada petición. Consecuencia de diseño: ese manejo vive en el cliente de la API, **no en cada vista** — si cada vista lo resolviera, habría 31 lugares donde olvidarlo, y el olvido se vería como una pantalla congelada sin explicación.

  La sección rescata además una propiedad de ADR-008 que vale la pena no perder: **el tiempo real y la red de seguridad usan el mismo camino** —invalidar y recargar—. No hay dos rutas por las que los datos lleguen a la pantalla, así que no pueden contradecirse.

  Registradas las dos consecuencias del agente en la interfaz: la configuración dentro de V50 y la distinción visual del autor del mensaje en V11 (ADR-014), más `RF-MET-09` como indicador en V30.

- **Quedó abierto:**
  - **La elección de framework del frontend sigue sin justificar**, como ya registraba ADR-009: depende de qué conoce Anthony, que tiene asignada la interfaz. Todo lo de la sección 5 se mantiene igual con Vue o Svelte. Debe resolverse antes de empezar la interfaz.
  - Sin cambios en lo demás: los cinco supuestos de §1.8, la corrección de `max_size`, el BAA y las actualizaciones pendientes en `docs/01` a `docs/03`.

- **Siguiente paso:** sección 6, flujos críticos: los cinco diagramas de secuencia que pide `CLAUDE.md` §8.

---

## 2026-10-15 — Sección 4: componentes del backend

- **Se decidió o produjo:** escrita la **sección 4 de `arquitectura.md`**, componentes del backend (C4 nivel 3), con diagrama Mermaid. Ocho módulos de negocio dentro de un mismo despliegue —ADR-003 fijó "backend modular, no microservicios"—, más dos piezas compartidas de infraestructura.

  **La regla que ordena la sección: cada dato tiene un dueño.** Un módulo es el único que escribe sus tablas; los demás leen a través de él. No es formalismo: con dos personas en paralelo sobre 63 requisitos, es lo que evita que la ventana de 24 horas quede escrita en tres lugares y uno se olvide al cambiarla.

  **Dos módulos que rompen el molde y por qué.** Asignación **no es dueña de ninguna tabla** y aun así es un módulo, porque la regla de `RF-CNV-12` se dispara desde tres lugares distintos —mensaje entrante, autoasignación al responder (`RF-CNV-11`, ADR-012) y escalada del agente (ADR-014)—; si viviera en cada camino serían tres implementaciones que se desincronizan. Métricas **no escribe nada**: ADR-013 eligió vistas SQL, así que no hay resúmenes que puedan desincronizarse.

  **Excepción documentada antes de que parezca un error:** el consumidor escribe mensajes y conversaciones, que son propiedad del módulo de Conversaciones, pero con el rol `app_worker` y la organización del evento en vez de la de una sesión. Es el mismo código con dos contextos de ejecución. La regla que lo mantiene en un solo camino: el módulo recibe el contexto como parámetro y nunca lo deduce. **Señal de desvío: si aparece un `guardarMensajeDesdeWebhook` distinto de `guardarMensaje`, hay dos caminos donde debería haber uno.**

  La sección fija además el criterio para saber si la interfaz de canal es real o decorativa: **agregar Instagram debería tocar el módulo de canal y ninguno más.** Y mapea los nueve requisitos innegociables de `CLAUDE.md` §8 al módulo que responde por cada uno, como insumo de la sección 12.

- **Quedó abierto:** sin cambios respecto de la entrada anterior —los cinco supuestos de §1.8, la corrección de `max_size`, el BAA y las actualizaciones pendientes en `docs/01` a `docs/03`.

- **Siguiente paso:** sección 5, componentes del frontend (C4 nivel 3).

---

## 2026-10-15 — ADR-014 aprobada; sección 3 de la arquitectura

- **Se decidió o produjo:** Franco **aprueba ADR-014**. Las 14 decisiones del tablero quedan Aprobadas. Escrita la **sección 3 de `arquitectura.md`**, contenedores C4 nivel 2, con diagrama Mermaid.

  **Lo que la sección resuelve de entrada** es una confusión que apareció en la sesión al explicar las colas: mirando el diagrama es fácil contar piezas y creer que son servicios. La sección abre con una tabla de qué se despliega y qué no. **Son dos despliegues —el Worker de la aplicación y el de la API— y cero servidores que mantener vivos.** Las tres colas son recursos gestionados; los tres consumidores son funciones exportadas del mismo Worker de la API, no procesos aparte.

  Precisión que corrige una simplificación hecha antes en la conversación: **son dos Workers, no uno.** Lo fija ADR-009 punto 2, y la razón importa: desplegar un cambio de interfaz no arriesga el webhook, que es la pieza con el requisito duro de `RF-TRV-02`.

  La sección deja explícitos **los tres escritores del sistema** —consumidor de eventos, API en el camino de salida, consumidor del agente—, que es donde se concentran los errores de concurrencia, y recuerda que los tres publican avisos con la misma función (ADR-008, regla 4). También documenta por qué las colas son tres y no una: tienen tolerancias incompatibles —100 reintentos, 7 días, dos intentos— y separarlas no agrega operación sino **aislamiento de fallos**.

  Dos observaciones que salen de la tabla de fallos por contenedor: solo una caída larga de Postgres causa pérdida irreversible, por la caducidad a 7 días de los identificadores de archivo de Meta; y **el fallo más disruptivo para el negocio no es el más técnico** —sin Resend no entran clientes nuevos, aunque todo lo demás funcione—.

- **Quedó abierto:**
  - Los cinco supuestos de §1.8 siguen sin confirmar.
  - Aplicar la corrección de `max_size` de Hyperdrive al configurar el entorno.
  - Decidir y firmar el BAA de Anthropic (permanente).
  - Actualizaciones pendientes en `docs/01` a `docs/03`, listadas en cada ADR; ninguna aplicada todavía.

- **Siguiente paso:** sección 4, componentes del backend (C4 nivel 3): los módulos de identidad, canal, conversaciones, contactos, asignación, métricas, consola, auditoría y agente, con qué datos es dueño cada uno.

---

## 2026-10-15 — Auditoría de escalabilidad; sección 11 escrita

- **Se decidió o produjo:** auditoría de las 14 decisiones contra límites reales, verificados en documentación oficial, y escrita la **sección 11 de `arquitectura.md`** (adelantada en el orden del índice porque responde a la pregunta directa de si el diseño escala). El método: tomar el límite duro de cada pieza y compararlo con el volumen proyectado de §1.1, en vez de confiar en que "Cloudflare escala".

  **Hallazgo 1, y es un defecto real que ningún ADR registraba: Hyperdrive puede abrir más conexiones de las que Supabase acepta.** Hyperdrive en plan de pago admite hasta ~100 conexiones al origen por configuración —y su documentación advierte que el valor es un límite blando que puede excederse—, mientras que **Supabase Micro, el cómputo que incluye el plan Pro que fijó ADR-011, acepta 60 conexiones directas**. ADR-003 fijó usar la cadena Direct. Con suficiente carga, las conexiones que sobran fallan. No pasa en el piloto —0,95 mensajes por segundo en pico no se acerca a 60 conexiones— pero aparecería bajo carga, que es cuando peor se depura. **Corrección: fijar `max_size` en 40**, agregada a la lista de puesta en marcha de ADR-011.

  **Hallazgo 2: el techo de 20 negocios no tiene margen, y es el primer límite del sistema por mucho.** Las conexiones de Postgres ceden hacia las 50 organizaciones, Realtime hacia los 200 asesores, Queues más allá del millón de operaciones. El tope de números por portafolio de Meta se alcanza en 20. Consecuencia de diseño que conviene tener presente: **no tiene sentido optimizar para más de 20 negocios mientras el rol de Tech Provider no exista**, porque es un techo que no se levanta por la vía técnica.

  **Lo que sí sostiene el diseño**, revisadas las 14 decisiones: sin estado en memoria (ADR-003), unicidad en la base y no en el transporte (ADR-005), orden por marca de tiempo de Meta (ADR-005), aislamiento por permisos (ADR-006), el aviso no es el dato (ADR-008) y tres colas con tolerancias distintas (ADR-005, ADR-014). La propiedad más valiosa es que **`organization_id` es la primera columna de cada índice**: el costo de una consulta depende del tamaño de una organización, no del total, así que 100 negocios no consultan más lento que 10.

  **Los cinco puntos donde el sistema cedería se resuelven con configuración o dinero** —subir el cómputo de Supabase cuesta ~15 USD al mes y menos de 2 minutos de interrupción—, ninguno exige rediseño. Eso es lo que distingue un diseño escalable: no que aguante todo, sino que lo que cede se arregle sin reescribir.

- **Quedó abierto:**
  - Aplicar la corrección de `max_size` al configurar Hyperdrive.
  - Medir V30 con datos realistas antes del piloto (ya lo exigía ADR-013).
  - Los cinco supuestos de §1.8 siguen sin confirmar; la sección 11 razona sobre ellos.
  - Aprobación de ADR-014.

- **Siguiente paso:** sección 3 de `arquitectura.md` (contenedores, C4 nivel 2).

---

## 2026-10-15 — D-14: agente de IA al MVP, con Anthropic como proveedor

- **Se decidió o produjo:** Franco incorpora al MVP un **agente de IA configurable por negocio**: responde en primera línea, escala a un asesor y cierra conversaciones. Las integraciones quedan para después. Escrito `docs/adr/014-agente-ia.md` en estado Propuesta.

  **Es un cambio de alcance, no un hueco pendiente.** `docs/01_definicion_mvp_v2.2.md` §6 excluía explícitamente los agentes de IA del MVP, diciendo que "la arquitectura se diseña para incorporarlos después". Se ofrecieron tres caminos —entrar recortando alcance, diseñar ahora y construir después, o entrar sin recortar— y **Franco eligió entrar sin recortar nada**, asumiendo el riesgo de calendario. La objeción quedó registrada en el ADR junto con un **orden de repliegue de cuatro escalones** para la revisión del 16 de noviembre, de modo que un recorte en noviembre sea una decisión preparada.

  **Decisión técnica: el agente es un consumidor propio detrás de su propia cola**, no dentro del consumidor de ADR-005. La razón es que `RF-TRV-02` —no perder mensajes entrantes— es una invariante absoluta y el proveedor de IA es la pieza más lenta e impredecible del sistema. Acoplarlos pondría el requisito más duro a merced de un tercero. Esta separación es además lo que permite apagar el agente dejando de consumir una cola, sin tocar el camino crítico.

  **Proveedor: Anthropic**, por decisión de Franco tras una primera versión escrita sobre OpenAI. Modelos y precios verificados en documentación oficial: Claude Haiku 5.5 (0,10/0,50 USD por millón de tokens), Claude Sonnet 5.5 (2/10), Claude Opus 5.5 (4/20). Recomendación del ADR: **empezar con Haiku** y medir la tasa de escaladas antes de subir de modelo.

  **Hallazgo que mejoró el plan.** Se verificó en la documentación oficial de Anthropic que, para datos de salud, el arreglo correcto no es la retención cero sino la **preparación para HIPAA**, y que esta **se habilita por autoservicio desde la consola con efecto inmediato** (descargar y aceptar el BAA). Esto elimina lo que con otro proveedor habría sido una dependencia externa de calendario con espera desconocida. Dos advertencias que el diseño ya cumple: solo se usa el endpoint de mensajes, que es elegible, y ningún dato del contacto va en esquemas JSON. Una que exige decisión: **la habilitación es permanente** y no se puede revertir desde la consola.

  Decisiones de producto que fija el ADR: configuración por negocio (información en texto libre, nombre y tono, reglas de escalada, preguntas y respuestas fijas que se consultan **antes** que el modelo); apagado por defecto; escalada por cuatro causas, incluida **el fallo del modelo dos veces** —lo que evita que una caída del proveedor deje conversaciones en silencio—; idempotencia por `(organization_id, conversation_id, inbound_message_id)`, porque a diferencia de una persona el agente reintenta solo; y un indicador nuevo `RF-MET-09`, porcentaje de conversaciones resueltas sin escalar, sin el cual el piloto no puede evaluar esta función.

- **Quedó abierto:**
  - **Aprobación de ADR-014** por Franco y Anthony.
  - **Decidir y aceptar el BAA de Anthropic.** Es permanente; lo firma un representante legal de la empresa.
  - **El riesgo de calendario**, que sube de forma material. El orden de repliegue del ADR es la mitigación.
  - Actualizaciones pendientes en los documentos fuente, listadas en el ADR: §6 y §8.1 de la definición del MVP, requisitos nuevos en el documento maestro, y §8 de CLAUDE.md.
  - Los cinco supuestos de `arquitectura.md` §1.8 siguen sin confirmar.

- **Siguiente paso:** sección 3 de `arquitectura.md` (contenedores, C4 nivel 2), que ahora debe incluir la cola y el consumidor del agente.

---

## 2026-10-15 — Las 13 ADR aprobadas; arranca el documento de arquitectura

- **Se decidió o produjo:** **Franco aprueba ADR-002 a ADR-013.** Las 13 decisiones del tablero quedan en estado Aprobada, y el tablero de `CLAUDE.md` §5 lo refleja. Con eso se cumple la condición de `CLAUDE.md` §7 y arranca el entregable principal.

  Creado `docs/architecture/arquitectura.md` con las **secciones 1 y 2 de las 12**, más un índice que marca el estado de cada una.

  **Sección 1, requisitos no funcionales.** Los números con los que se juzga el diseño, separados en tres categorías de peso distinto: exigido (está en un requisito), derivado (se calcula, y el cálculo se muestra) y supuesto (lo propone el documento porque ninguna fuente lo da). El resultado que gobierna el resto: **menos de 1 mensaje por segundo en el pico con 20 negocios**, contra un throughput de Meta de 80 mps — un margen de 84×. El sistema está sobredimensionado por decisión, no por accidente: las piezas se eligieron por corrección y costo de operación, nunca por capacidad.

  Dos hallazgos de esta sección que no estaban escritos en ningún ADR: **el primer límite de crecimiento de Numi CRM no es técnico, es el tope de 20 números por portafolio de Meta** (ADR-001) mientras no haya rol de Tech Provider; y **el riesgo de latencia real no es la cadena webhook→bandeja —estimada en 1,6 s contra un objetivo de 5— sino la consulta de respaldo de 60 s de ADR-008** si el Broadcast falla, que es doce veces el objetivo.

  **Sección 2, contexto C4 nivel 1.** Diagrama Mermaid con los tres actores, el contacto —que origina el tráfico pero no es usuario del sistema, porque su interfaz es WhatsApp— y los ocho sistemas externos, cada uno con qué pasa si falla y qué ADR lo decide. Incluye lo que queda fuera del sistema, para que el diseño no crezca por inercia.

- **Quedó abierto:**
  - **Cinco supuestos esperan confirmación de Franco y Anthony**, listados en §1.8: mensajes por conversación (20, el más influyente porque multiplica todo lo demás), usuarios por negocio (3 a 10), negocios a 12 meses (20), umbral de la vista de métricas (2 s) y techo de costo mensual (unos 100 USD). Las secciones 3 y 11 razonan sobre ellos.
  - Secciones 3 a 12 del documento de arquitectura.
  - La revisión independiente de `CLAUDE.md` §11, que debe hacerse en sesión nueva.
  - Reconfirmar los datos de proveedores antes de construir.

- **Siguiente paso:** sección 3, contenedores (C4 nivel 2): la aplicación del CRM, la consola, la API, el consumidor de webhooks, la cola, la base, el tiempo real, el almacenamiento de archivos y el correo, con cómo se comunican.

---

## 2026-10-15 — Auditoría de coherencia entre ADR; hueco de notificación cerrado

- **Se decidió o produjo:** auditoría de los 13 ADR como conjunto, no uno por uno. El grafo de dependencias es sano: sin ciclos ni referencias a decisiones posteriores. Las invariantes transversales se sostienen —"la API es el único camino hacia los datos" se mantiene intacta en ADR-007, ADR-008, ADR-010 y ADR-013, y `RF-TRV-07` queda garantizado por permisos de base de datos (ADR-006) y no por disciplina—.

  **Hueco corregido:** ADR-008 describía el Broadcast como si lo publicara únicamente el consumidor de la cola, pero el envío de ADR-012 es síncrono y escribe desde la API. Un mensaje enviado por un asesor, y la autoasignación que lo acompaña (`RF-CNV-11`), no llegaban a las pantallas de los demás usuarios hasta la consulta de respaldo de 60 segundos: cuatro veces el objetivo de `RF-TRV-03` y un incumplimiento de `RF-CNV-15`, que no distingue el origen del mensaje.

  Se corrigieron los dos lados: ADR-008 gana la sección **"Quién publica el aviso"**, con los dos escritores del sistema, y la regla de que publicar vive en una única función que ambos llaman. ADR-012 gana los avisos en los pasos 6 y 8 del camino de salida, con la aclaración de que son para los demás usuarios —quien envía ya tiene su respuesta HTTP— y que un fallo al publicar no falla el envío. Las dependencias cruzadas quedaron declaradas: ADR-012 depende de ADR-008.

  **Tablero corregido** (`CLAUDE.md` §5): la columna "Depende de" estaba desactualizada respecto a los encabezados de los ADR en cinco filas —D-10 decía `D-03` cuando el ADR declara cuatro dependencias, D-11 decía `D-04, D-09` cuando declara siete—. Ahora refleja lo que declara cada ADR, con una nota de que se actualiza con ellos. En §7 se corrigió el criterio de terminado, que seguía diciendo "D-01 a D-11" y "las 11 decisiones" desde antes de agregar D-12 y D-13, y una referencia cruzada equivocada a la sección 9 en lugar de la 11.

- **Quedó abierto:**
  - **Aprobación de ADR-002 a ADR-013.** Siguen en Propuesta. Es lo que bloquea el documento de arquitectura: `CLAUDE.md` §7 lo condiciona a decisiones aprobadas, y escribirlo sobre doce propuestas significa reescribirlo si alguna cambia.
  - **Reconfirmar datos de proveedores antes de construir.** Los ADR citan la fecha de compatibilidad de Workers (2026-08-04), el plan gratuito de Resend (3.000 al mes, 100 al día) y Supabase Pro a 25 USD. Se verificaron al escribirlos, pero ADR-001 registra que Embedded Signup v2 quedó obsoleta el 15 de octubre de 2026 —hoy mismo—, lo que muestra que estos datos se mueven.
  - La revisión independiente de `CLAUDE.md` §11 sigue pendiente: esta auditoría la hizo una sesión con el contexto de los ADR a la vista, así que no la sustituye.

- **Siguiente paso:** aprobación de los 12 ADR en Propuesta. Si Franco y Anthony prefieren avanzar antes, se pueden escribir las secciones 1 y 2 de `arquitectura.md` —requisitos no funcionales y contexto C4 nivel 1—, que no dependen de ninguna decisión pendiente.

---

## 2026-10-15 — Auditoría del tablero: dos huecos cubiertos (D-12 y D-13)

- **Se decidió o produjo:** auditoría de las once decisiones a pedido de Franco. Resultado: **115 requisitos en los documentos, 60 citados en los ADR**. La mayoría de los 55 restantes son requisitos de producto sin decisión arquitectónica pendiente, pero **dos grupos sí planteaban decisiones reales y no estaban cubiertos**. Se escribieron dos ADR nuevos y se agregaron D-12 y D-13 al tablero de CLAUDE.md §5.

  **`docs/adr/012-envio-de-mensajes.md`** — ADR-005 resolvió en detalle cómo **entran** los mensajes; cómo **salen** quedó implícito, pese a tocar dos no-negociables (`RF-CNV-08` ventana de 24 h, `RF-CNV-09` estados y reintento). Decisión: **síncrono, con persistencia previa y reintento manual**. El mensaje se guarda con estado `enviando` **antes** de llamar a Meta, así no se pierde lo que el asesor escribió; la llamada ocurre dentro de la petición para que el asesor vea el resultado real; el reintento es del asesor, no automático, **porque en el camino de salida no hay identificador de Meta antes de enviar y un reintento automático duplicaría el mensaje al contacto**.

  Va en dirección contraria a ADR-005 a propósito: en el webhook espera Meta, que castiga la demora; aquí espera una persona mirando la pantalla.

  Datos verificados: error **131047** cuando la ventana venció ("More than 24 hours have passed..."), throughput **80 mensajes/s** por defecto (contra menos de 1 que necesita el piloto), 20 mps fijos en coexistencia. Hallazgo de diseño: **la ventana de 24 h no se puede verificar con certeza localmente** —el reloj lo lleva Meta—, así que la verificación local es comodidad y la respuesta de Meta es la verdad. **Las plantillas quedan fuera del MVP**, de modo que una conversación con la ventana vencida no se puede retomar: es una limitación real que hay que explicar a los negocios del piloto.

  **`docs/adr/013-metricas.md`** — `RF-MET-01` a `RF-MET-08` no aparecían en ningún ADR. Decisión: **consultas SQL directas al abrir la vista**, con vistas `security_invoker` (ADR-006) e índices pensados para ellas. Se descartó precalcular porque **la mediana de `RF-MET-02` no es combinable por día** (la mediana de las medianas no es la mediana) y `RF-MET-04` depende del estado **actual**, así que dos de los cinco indicadores consultarían las tablas igual. Además, una métrica desincronizada es peor que una lenta: nadie la nota.

  El ADR fija ocho definiciones que los requisitos dejaban ambiguas (qué cuenta como espera de respuesta, qué asesor se atribuye un contacto, que los límites del período van en hora de Bogotá y no en UTC) y pide **medir con 6.000 conversaciones y 60.000 mensajes antes del piloto**.

- **Quedó abierto:**
  - Aprobación de ADR-002 a ADR-013 (solo ADR-001 está Aprobada).
  - Otros hallazgos de la auditoría: la actualización pendiente de `RF-PLT-USR-02` en `docs/03` (P23 como vista de soporte, ADR-002) y el punto abierto de ADR-009 sobre la elección de framework.
  - **Advertencia sobre esta auditoría:** la hizo la misma sesión que escribió los trece ADR. `CLAUDE.md` §11 pide revisión en sesión nueva, que sigue pendiente y no queda cubierta por esto.

- **Siguiente paso:** revisión independiente en sesión nueva sobre los trece ADR, y después el documento de arquitectura.

## 2026-10-15 — D-11, secretos, correo, entornos, despliegue y observabilidad

- **Se decidió o produjo:** `docs/adr/011-secretos-entornos-despliegue.md` en estado Propuesta. **Última decisión del tablero: las 11 están resueltas** (D-01 Aprobada, D-02 a D-11 en Propuesta).

  Resoluciones: tokens de WhatsApp cifrados con **AES-GCM por Web Crypto** —parte del runtime, sin biblioteca de npm que verificar—, con la clave como secreto del Worker, **fuera de la base**. Correo con **Resend en plan gratuito** (3.000/mes, dominio propio incluido), con SMTP propio en Supabase Auth. **Dos entornos** sin nada compartido. **Supabase Pro en producción** (25 USD), porque el plan gratuito **pausa el proyecto tras una semana de inactividad** y el piloto empieza el 1 de diciembre. Despliegue desde GitHub con **las pruebas de aislamiento como requisito**; migraciones versionadas y **solo aditivas** (la reversión de Cloudflare cubre el código, no el esquema).

  **Costo fijo total: 30 USD al mes** (Workers 5 + Supabase Pro 25), más los mensajes de Meta y el dominio.

  **Franco aportó un dato de experiencia**: en un proyecto anterior tuvo que migrar de la capa REST de Supabase (PostgREST) a SQL directo por lentitud y caídas. Coincide con lo ya decidido —ADR-003 usa node-postgres por Hyperdrive, ADR-007 usa Supabase solo como identidad—, así que el camino de datos no toca PostgREST. Pero se registró la consecuencia: **Supabase son tres piezas (Postgres, Auth, Realtime) que no fallan juntas**, y concentrarlas en un proveedor con antecedentes de caídas es el mayor riesgo de infraestructura del sistema. Se asume conscientemente y se documentó la tabla de modos de falla.

  Tres alertas mínimas definidas. La primera —**webhook caído a las 2 horas**— es la más urgente porque **tiene un reloj de 7 días**: pasado ese plazo los identificadores de archivo de Meta caducan y los archivos se pierden para siempre. **Es el único fallo del sistema con pérdida irreversible.**

  Lista de **configuración manual por entorno** documentada, incluyendo el "Allow public access" de Realtime que ADR-008 dejó anotado: no está en el código y **un canal público no da error, simplemente deja de aislar**.

- **Quedó abierto:**
  - Aprobación de ADR-002 a ADR-011 por Franco y Anthony (solo ADR-001 está Aprobada).
  - **Acción recomendada para la semana del 19 de octubre: configurar Resend con SPF, DKIM y DMARC.** Es barato, desbloquea el alta de organizaciones y su tiempo de espera (propagación de DNS) no depende del equipo.
  - Reconfirmar las cifras de planes de Supabase antes de contratar: las fuentes difieren en el nombre del tamaño base y en algunos números.
  - Probar una restauración de copia de seguridad antes del piloto. Una copia no probada no es una copia.
  - **Punto abierto de ADR-009:** la elección de React no está justificada en ese ADR; depende de qué conoce Anthony, que tiene asignada la interfaz.

- **Siguiente paso:** con el tablero completo, construir `docs/architecture/arquitectura.md` según la sección 7 de CLAUDE.md (12 secciones, diagramas en Mermaid). Antes conviene la **aprobación de los ADR pendientes** y, según la sección 11, una **revisión independiente en sesión nueva**.

## 2026-10-15 — D-10, archivos y almacenamiento en R2

- **Se decidió o produjo:** `docs/adr/010-archivos-r2.md` en estado Propuesta. Decisión: **el Worker sirve los archivos**, verificando permisos en cada petición. **No se usan URL firmadas.** R2 se usa para **una sola cosa**: los archivos recibidos por WhatsApp.

  El punto real de esta decisión era por dónde viaja el archivo hasta el navegador. Normalmente eso se decide por costo —hacer pasar archivos por un servidor cuesta ancho de banda— pero **en R2 la salida de datos es gratuita** ("does not incur data transfer (egress) charges and is free"). Sin ese argumento, quedaba solo la seguridad, y ahí una URL firmada es un **token al portador**: "should be treated as bearer tokens", funciona para cualquiera que la tenga y **no se puede revocar** hasta que expire. Con fotos clínicas en juego (ADR-004), eso es una fuga que ningún control posterior revierte.

  Es la **tercera vez** que aparece el mismo problema —token que no se puede revocar— después de los tokens de sesión (ADR-007) y el canal de tiempo real (ADR-008). La respuesta es la misma: verificar en cada petición.

  Reglas fijadas: clave `org/{organization_id}/msg/{message_id}/{media_id}`, **nunca expuesta al navegador**; acceso solo por binding, sin credenciales S3 ni dominio público; archivo devuelto **en flujo**; un usuario de A pidiendo un archivo de B recibe **el mismo resultado que si no existiera**, no un error distinto que revelaría su existencia; borrado **archivo antes que fila** (el orden inverso dejaría huérfanos invisibles) más tarea periódica de limpieza; documentos forzados a descarga, no abiertos en el navegador.

  **Costo: 0 USD** al volumen del piloto. Unos 300 MB al mes contra 10 GB gratuitos; el primer costo real aparece alrededor de 25–30 organizaciones con un año de historial.

- **Quedó abierto:**
  - Aprobación de ADR-010 por Franco y Anthony.
  - Vigilar documentos de hasta 100 MB pasando por el Worker. Si dan problema, el ADR fija que las URL firmadas son el **último** recurso, no el primero.

- **Siguiente paso:** D-11, la última: secretos y tokens cifrados, correo propio, entornos, despliegue y observabilidad. Arrastra varios pendientes acumulados, entre ellos el correo que bloquea el alta de organizaciones (ADR-007) y el plan de Supabase (ADR-008).

## 2026-10-15 — D-09, arquitectura del frontend

- **Se decidió o produjo:** `docs/adr/009-arquitectura-frontend.md` en estado Propuesta. Decisión: **aplicación de página única con React y Vite, servida desde Cloudflare Workers**. CRM y consola en **una sola aplicación**, con las rutas de `/plataforma` cargadas de forma diferida.

  Se abandona la propuesta de §8.1 (Next.js con OpenNext). Dos razones, y la primera pesa más que la segunda:

  **1. Next.js resolvería un problema que este sistema ya no tiene.** ADR-007 dejó al frontend como consumidor puro de una API: el navegador nunca lee la base. El renderizado en servidor —lo único que justifica la complejidad de Next.js— no se puede usar, porque el servidor del frontend no tiene acceso a datos. Se pagaría la complejidad sin obtener el beneficio.

  **2. Las dos vías de Next.js en Workers están inmaduras.** Verificado hoy: Cloudflare **cambió su recomendación** y ahora dice "Cloudflare recommends vinext as the default way to run Next.js applications on Cloudflare Workers", pero **"vinext is in beta"**. OpenNext quedó como vía para *mantener* aplicaciones existentes ("cannot yet migrate to vinext because of a compatibility gap"), con huecos conocidos y una cola de revalidación "not fully suitable for production deployments". **Es el mismo patrón que Fastify en D-04**: el dato del brief caducó.

  La ruta de aplicación de página única en Workers está documentada y madura, con complemento oficial de Vite y `assets.not_found_handling` en modo SPA.

  Otras reglas fijadas: dos Workers separados (aplicación y API); el cliente de Supabase en el navegador se usa **solo para autenticar, nunca para consultar datos**; biblioteca de datos de servidor (TanStack Query o similar) y **sin gestor de estado global**, que no tendría trabajo; el aviso de tiempo real **invalida consultas**, nunca escribe en la caché; búsqueda de contactos en el servidor (`RF-TRV-03`); carga diferida de `/plataforma` por rendimiento, **no por seguridad** —la protección real es `RF-PLT-SEG-01` en el backend.

- **Quedó abierto:**
  - Aprobación de ADR-009 por Franco y Anthony.
  - **31 vistas (18 del CRM + 13 de la consola) en siete semanas con dos personas sigue siendo el riesgo central del proyecto.** Esta decisión lo reduce al quitar una capa de adaptación, pero no lo elimina.
  - La elección de base de componentes corresponde a la reunión de arranque del design system; el criterio fijado es usar una base ya resuelta, no construir desde cero.
  - Actualizar §8.1 y §13 de la definición del MVP, y §9 de CLAUDE.md.

- **Siguiente paso:** D-10, archivos (si se usa R2 y para qué). Depende de D-03, ya resuelta; ADR-004 dejó buena parte del terreno fijado.

## 2026-10-15 — D-08, identidad, roles y sesiones

- **Se decidió o produjo:** `docs/adr/007-identidad-y-roles.md` en estado Propuesta. Decisión: **Supabase Auth solo como identidad; toda la autorización en el backend**. En una frase: el proveedor responde "quién eres"; la API responde "qué puedes hacer", y lo consulta en la base **en cada petición**.

  **TOTP confirmado** (pendiente que dejó ADR-002): "TOTP MFA API is free to use and is enabled on all Supabase projects by default". `RF-PLT-ACC-02` y `RF-PLT-ACC-03` se cumplen llamando al proveedor, sin construir nada.

  Dos hechos verificados obligaron al diseño: (1) **cerrar sesión no invalida el token de acceso** ("the access token JWT will still be valid until it expires"), y (2) los límites de sesión se aplican **de forma perezosa**, así que la duración real es el tiempo configurado más la vigencia del token. Si la autorización se apoyara en el token, un usuario desactivado seguiría entrando hasta una hora y `RF-AUT-07` (suspensión cierra sesiones "en la siguiente acción") sería imposible. Por eso **el estado de usuario y organización se verifica contra la base en cada petición**, no contra el token.

  Consecuencia: ni el rol de organización ni el rol de plataforma viajan en el token. `RF-PLT-SEG-01` exige verificación del rol en cada petición, y un claim en un token de una hora no permite revocar a un operador. Los operadores viven en **tabla propia**, separada de los usuarios de negocio, poblada por migración. **`aal2` es obligatorio en toda ruta de plataforma de la API**, no solo en la pantalla.

  Se descartó la Opción A (RLS directo desde el navegador con el cliente de Supabase): contradice `RF-PLT-SEG-01`, dejaría la suspensión en manos de un token irrevocable y no da dónde esconder los tokens de WhatsApp.

  Configuración fijada: `jwt_expiry` 3600 s, `sessions.timebox` 12 h, `sessions.inactivity_timeout` 30 min (`RF-PLT-SEG-03`), rotación de tokens de refresco activa.

- **Quedó abierto:**
  - Aprobación de ADR-007 por Franco y Anthony.
  - **Dependencia de ruta crítica: el correo con remitente propio bloquea el alta de organizaciones.** Sin él no hay invitación (`RF-PLT-ORG-06`) y el primer administrador no entra. Formalmente es D-11, pero conviene resolverlo antes: es barato y desbloquea el flujo principal de la consola.
  - **El tiempo real de Supabase va directo del navegador a la base y no pasa por esta verificación.** D-07 debe resolverlo explícitamente.
  - Actualizar §8.1 y §8.2 punto 2 de la definición del MVP, y la decisión 8 de §2 de la consola.

- **Siguiente paso:** D-07, tiempo real en la bandeja. Depende de D-04 y D-06, ambas resueltas, y arrastra el punto abierto de ADR-007 sobre el canal de tiempo real.

## 2026-10-15 — D-07, tiempo real en la bandeja

- **Se decidió o produjo:** `docs/adr/008-tiempo-real.md` en estado Propuesta. Decisión: **Broadcast desde el servidor**, con consulta periódica de 60 segundos como red de seguridad. El consumidor de la cola notifica por HTTP a un canal privado por organización; el navegador recibe el aviso y **pide los datos a la API**.

  Esto resuelve el punto que ADR-006 y ADR-007 dejaron anotado dos veces. Con Broadcast desde el servidor **el navegador solo escucha**: no consulta la base, no lee filas, no necesita permisos sobre ninguna tabla. El canal deja de ser una vía de acceso a datos y pasa a ser un aviso, así que la regla de ADR-007 —el token identifica, la API autoriza en cada petición— se mantiene sin excepciones.

  Se descartó Postgres Changes (la opción que §8.2 punto 2 de la definición llamaba "la más simple"), por cuatro datos verificados: **Supabase mismo recomienda Broadcast** ("the recommended method for scalability and security", "Postgres Changes does not scale as well"); la autorización es **por suscriptor** (100 suscriptores = 100 lecturas por `INSERT`) y se procesa en **un solo hilo**; los permisos del canal **se cachean durante toda la conexión**, así que una desactivación no surte efecto hasta que expire el token —reintroduciendo por otra puerta el problema que ADR-007 resolvió—; y un fallo de política produce **silencio**, no error ("the single biggest source of silently missing events").

  Dato que lo hace viable desde Workers: existe **endpoint HTTP de Broadcast**, así que un Worker puede notificar sin sostener un WebSocket (ADR-003 lo impide).

  Reglas fijadas: el aviso **no lleva contenido** (solo tipo de evento, identificador de conversación y hora), canales privados obligatorios con "Allow public access" desactivado, solo el servidor publica, la notificación ocurre **fuera** de la transacción de escritura, y el contador de sin responder se **recalcula**, no se incrementa a ciegas. La consola no usa tiempo real.

- **Quedó abierto:**
  - Aprobación de ADR-008 por Franco y Anthony.
  - **Argumento nuevo a favor del plan Pro de Supabase:** los proyectos gratuitos **se pausan tras una semana de inactividad**, lo que es un riesgo real en los días previos al piloto del 1 de diciembre. La decisión del plan es D-11.
  - El aislamiento del canal es un sistema de políticas **aparte** del RLS de las tablas (`realtime.messages`) y necesita su propia prueba automática, que extiende `RF-TRV-01` al tiempo real.
  - "Allow public access" es un interruptor de configuración del que depende el aislamiento y **no está en el código**: hay que recordarlo en el despliegue (D-11).
  - Actualizar §8.2 punto 2 y §8.1 de la definición del MVP, y `RF-CNV-15` del documento maestro.

- **Siguiente paso:** D-09, arquitectura del frontend (framework y hosting, CRM y consola juntas o separadas, sesión, obtención de datos, estado, integración con el tiempo real). Depende de D-04, D-07 y D-08, las tres ya resueltas.

## 2026-10-15 — D-06, aislamiento multiempresa

- **Se decidió o produjo:** `docs/adr/006-aislamiento-multiempresa.md` en estado Propuesta. Decisión: **las dos capas**, RLS en la base **más** filtrado explícito por `organization_id` en el código. El filtrado es el camino normal; RLS es la red que atrapa la consulta olvidada. Esto resuelve el punto abierto de §8.2 punto 3 de la definición del MVP: la respuesta es ambos patrones, no uno.

  Razón: `RF-TRV-01` es la condición que define si el piloto fue exitoso (§11). Un requisito de ese peso no puede depender de que dos personas con prisa no olviden nunca un `WHERE`. Una consulta olvidada no falla: devuelve datos de otro negocio en silencio.

  **Tres roles de base de datos**, ninguno con `bypassrls`: `app_user` (usuarios de un negocio, filas de su organización), `app_platform` (consola, todas las organizaciones pero **sin permiso sobre las tablas de mensajes**) y `app_worker` (consumidor de la cola, escribe aunque la organización esté suspendida). El punto clave: `app_platform` no obtiene su poder saltando RLS sino por no tener permiso, así que **`RF-TRV-07` deja de depender de disciplina y pasa a ser imposible a nivel de permisos**.

  Datos verificados que lo hacen viable: Hyperdrive opera en **modo transacción** y hace `RESET` al devolver la conexión ("the connection is `RESET` such that the `SET` commands will not take effect on subsequent queries"), así que **el contexto no se filtra entre peticiones** —el riesgo clásico del patrón no existe aquí—, pero obliga a fijar el contexto **en cada transacción** con alcance local, encapsulado en una única función. En Supabase, `service_role` y el dueño `postgres` tienen `bypassrls`; de ahí que los roles de aplicación deban ser otros.

  Siete pruebas automáticas definidas. Las más valiosas por esfuerzo son las tres de catálogo: ningún rol de aplicación tiene `bypassrls`, toda tabla con `organization_id` tiene RLS activo, y las vistas usan `security_invoker` (las vistas **saltan RLS por defecto**, y §9 de CLAUDE.md propone métricas con vistas SQL).

- **Quedó abierto:**
  - Aprobación de ADR-006 por Franco y Anthony.
  - **Las políticas y los roles deben existir desde la primera migración**, no en la última semana. Si se dejan para el 26–30 de noviembre habrá consultas escritas suponiendo que no existen.
  - El tiempo real de Supabase tiene políticas propias, aparte del RLS de las tablas; hay que verificar ese aislamiento por separado en D-07.
  - Actualizar §8.2 punto 3 y §8.3 de la definición del MVP.

- **Siguiente paso:** D-08, identidad y roles (Supabase Auth, rol de plataforma, doble factor, sesiones). Depende de D-06, ya resuelta. Nota: el orden de CLAUDE.md §5 pone D-08 antes que D-07.

## 2026-10-15 — D-05, procesamiento del webhook

- **Se decidió o produjo:** `docs/adr/005-procesamiento-webhook.md` en estado Propuesta. Decisión: **webhook mínimo más Cloudflare Queues**, con la corrección resuelta en la base de datos y no en la cola. Se confirma el **plan de pago de Workers** (5 USD al mes), que ADR-003 había dejado abierto.

  Hallazgo que define la decisión, verificado hoy: **ni Meta ni Cloudflare Queues garantizan orden, y los dos pueden duplicar**. Meta: "These retries can result in duplicate webhook notifications", reintentos hasta 7 días, sin garantía de orden documentada. Queues: "at least once delivery", "does not guarantee that messages will be delivered in the same order in which they are published". Por lo tanto `RF-TRV-02` y `RF-CNV-06` **no se pueden resolver en el transporte**: van a la base con unicidad `(organization_id, wamid)` y ordenamiento por la marca de tiempo de Meta con desempate estable (su resolución es de segundos).

  El webhook hace solo cinco cosas: verifica firma, resuelve organización, **inserta el evento crudo**, encola una referencia y responde 200. No interpreta, no descarga, no envía. Razón: lo único que Meta castiga es no recibir un 200; con el evento guardado ya no se puede perder nada.

  Segundo hallazgo: los reintentos de Meta duran 7 días **igual que la vigencia del identificador de archivo** (ADR-004). **Una caída larga pierde archivos aunque no pierda mensajes.** Esto condiciona las alertas de D-11: una alerta de webhook caído tiene un reloj corriendo.

  Detalle que hay que probar explícitamente: Meta calcula la firma `X-Hub-Signature-256` sobre el cuerpo con **unicode escapado y hexadecimal en minúsculas**; hashear los bytes crudos da otro resultado. **Probar con tildes o emoji antes de darlo por bueno** — es el error típico y silencioso.

  Otras reglas fijadas: dos colas separadas (eventos y descargas), confirmación individual y no por lote, consumidor idempotente, cola de mensajes fallidos obligatoria, tabla de eventos crudos con retención de 30 días, y estados de mensaje como **progresión** que tolera desorden (Meta documenta que un mensaje puede producir `delivered` y `failed` a la vez).

- **Quedó abierto:**
  - Aprobación de ADR-005 por Franco y Anthony.
  - Plan de pago de Workers como costo fijo nuevo; actualizar §8.1 de la definición del MVP.
  - Precisar `RF-CNV-09` en el documento maestro: los estados siguen una progresión y un estado anterior no pisa a uno posterior.
  - Agregar la tabla de eventos recibidos a la tabla de retención de §8.

- **Siguiente paso:** D-06, aislamiento multiempresa (cómo se aplica la seguridad por fila desde la API). Depende de D-04, ya resuelta.

## 2026-10-15 — D-03, audios, imágenes y documentos

- **Se decidió o produjo:** `docs/adr/004-multimedia-mvp.md` en estado Propuesta. Decisión: **se descargan, se guardan y se muestran** imágenes y audios; documentos como enlace de descarga. El envío de archivos desde el CRM queda fuera.

  Hallazgo que cambió el marco, verificado hoy contra Meta: el webhook trae **solo el identificador**, no el archivo; **el identificador caduca a los 7 días** y la URL de descarga a los **5 minutos**. Lo que no se descarga en esa ventana **se pierde para siempre**. Eso parte la decisión en dos: **guardar es irreversible, mostrar no lo es**. Los documentos fuente las trataban como una sola porque se escribieron sin ese dato.

  Segundo argumento: ADR-001 eliminó la alternativa. El piloto corre con líneas **nuevas** del portafolio de Numi, así que el negocio no tiene ninguna app donde ver esos archivos. El argumento de §13 de la definición del MVP ("la clínica no tiene otro lugar donde verlos") es literalmente cierto en modo transitorio.

  Límites de Meta verificados: audio 16 MB, imagen 5 MB, documento 100 MB, video 16 MB. Videos y stickers se descargan y guardan igual aunque el MVP no los muestre, por el mismo criterio.

- **Quedó abierto:**
  - Aprobación de ADR-004 por Franco y Anthony.
  - **Numi pasa a custodiar datos sensibles de salud** (fotos clínicas, no solo texto). El acuerdo de encargo y la política de tratamiento que §8 de la definición ya exigía deben revisarse con el abogado sabiendo esto.
  - Alcance nuevo en un calendario ya ajustado por ADR-002. Orden de repliegue definido en el ADR; **la descarga y el guardado no se recortan en ningún escenario**.
  - Requisitos nuevos propuestos (`RF-CNV-10a` a `RF-CNV-10e`) y reescritura de `RF-CNV-10`, a confirmar al actualizar el documento maestro.

- **Siguiente paso:** D-05, procesamiento del webhook (cola, idempotencia, orden y reintentos). Depende de D-01 y D-04, ambas ya resueltas.

## 2026-10-15 — D-04, runtime del backend

- **Se decidió o produjo:** `docs/adr/003-runtime-backend.md` en estado Propuesta. Runtime elegido: **Hono en Cloudflare Workers**.

  Dato que cierra la opción del brief, verificado hoy contra Cloudflare: **Fastify sigue sin soporte en Workers** ("we're hoping to be able to add Fastify support later"), mientras que **Express sí funciona** y tiene tutorial oficial. Además, `nodejs_compat` está activo por defecto desde la compatibility date 2026-08-04. La propuesta de §8.1 de la definición del MVP (Fastify en Workers) queda inviable tal como está escrita.

  Razonamiento: a menos de 1 mensaje por segundo el rendimiento no es criterio. Lo que decide es cuántos proveedores hay que operar con dos personas a tiempo parcial. Salir de Workers (Fastify en un servicio Node) agrega un proveedor más que montar, monitorear y desplegar, y deja el webhook de Meta —único punto con requisito duro de disponibilidad, `RF-TRV-02`— sobre un contenedor que el equipo tiene que mantener vivo. Entre Hono y Express en Workers, Hono gana por ser nativo: mismo costo y mismas primitivas, sin la capa de compatibilidad.

  Otros datos verificados: Hyperdrive con Supabase usa la cadena **Direct**, no las agrupadas, con node-postgres ≥ 8.16.3 y un rol de base de datos dedicado de privilegios mínimos. Cloudflare Queues: gratuito 10.000 operaciones/día con retención de 24 h **no configurable**; de pago 1M/mes incluidas más 0,40 USD por millón, con retención de 4 a 14 días.

  Regla de diseño que fija el ADR: **el framework vive solo en la capa de entrada**; el código de negocio no importa nada de Hono. Sin estado en memoria entre peticiones. Toda biblioteca de npm se verifica en el runtime antes de adoptarse.

- **Quedó abierto:**
  - Aprobación de ADR-003 por Franco y Anthony.
  - **Plan de Workers:** el gratuito alcanza por volumen, pero la retención de 24 h no configurable de Queues es un riesgo para `RF-TRV-02` (un incidente de más de un día perdería mensajes encolados). Se confirma en D-05.
  - Curva de aprendizaje de Hono y del modelo de ejecución de Workers, con el calendario ajustado.
  - Actualizaciones pendientes: §8.1, §8.2 punto 1 y §13 de la definición del MVP; §9 de CLAUDE.md.

- **Siguiente paso:** D-03, visualización de audios, imágenes y documentos en el MVP (siguiente en el orden de CLAUDE.md §5; depende de D-01, ya aprobada).

## 2026-10-15 — D-02, alcance de la consola de plataforma

- **Se decidió o produjo:** `docs/adr/002-alcance-consola.md` en estado Propuesta. Alcance elegido: **consola completa**, los 44 requisitos P0 y las 11 vistas P0 de `docs/03_consola_plataforma_v1.1.md`. Fuera del MVP solo lo ya marcado P1: P10, P30, `RF-PLT-ORG-12`, `RF-PLT-USO-02`, `RF-PLT-USO-03`, `RF-PLT-AUD-04`, `RF-PLT-WA-08`.

  Tres precisiones confirmadas por Franco y Anthony: (1) la consola es herramienta de administración de Numi, no autoservicio; crea e invita solo al **primer administrador**, y los asesores los invita el administrador del negocio desde el CRM (`RF-USR-*`), lo que vuelve esos requisitos P0 irrecortables; P23 queda como vista de soporte. (2) El **doble factor del operador se resuelve con el TOTP del proveedor de identidad** (Supabase Auth), no con un flujo propio; a confirmar en D-08. (3) La **auditoría se escribe desde la primera acción** aunque P25 llegue después, porque un registro con huecos no se reconstruye.

  Se descartó la opción de consola mínima más scripts: rompe `RF-PLT-AUD-01` y `RF-PLT-SEG-02`, y contradice el propósito declarado de la consola (§1 del documento).

- **Quedó abierto:**
  - Aprobación de ADR-002 por Franco y Anthony.
  - **Riesgo de calendario:** 44 requisitos de consola más 63 del CRM en siete semanas con dos personas a tiempo parcial. Corte de emergencia definido para la revisión del 16 de noviembre, en orden: P25, luego `RF-PLT-USO-01`, luego buscador y filtros de P20.
  - Correo propio con entrega confiable (`RF-PLT-SEG-05`) sin resolver; el servidor de Supabase no sirve para producción. Se decide en D-11.
  - Actualizaciones pendientes: §5.4 y §2 (decisión 8) de la consola; §12 de la definición del MVP.

- **Siguiente paso:** D-04, runtime del backend (Hono o Express en Workers, o Fastify en un servicio Node).

## 2026-10-15 — ADR-001 aprobada; arranque en modo transitorio

- **Se decidió o produjo:** Franco y Anthony **aprueban ADR-001**. Punto de partida confirmado: el MVP y el piloto corren con **números del portafolio de Numi** (conexión manual, `RF-PLT-WA-02`), y el **objetivo final es Tech Provider** con Embedded Signup, migrando sin reescribir la integración. D-01 pasa a Aprobada en el tablero de CLAUDE.md §5.

- **Quedó abierto:**
  - Límite del modo transitorio: **2 clientes** mientras el negocio no esté verificado ante Meta, **20** una vez verificado. El piloto del 1 de diciembre con tres nichos necesita la **verificación de negocio** resuelta; Tech Provider no es requisito para arrancar.
  - Actualizaciones pendientes en documentos fuente (§3, §8.1 y §12 de la definición del MVP; §5.5 de la consola).

- **Siguiente paso:** D-02, alcance de la consola (completa o versión mínima).

## 2026-10-14 — D-01, modelo de conexión de WhatsApp

- **Se decidió o produjo:** `docs/adr/001-modelo-conexion-whatsapp.md` en estado Propuesta. Modelo objetivo: **Tech Provider con Embedded Signup**, cada negocio con su propio portafolio y su cuenta de WhatsApp. El piloto puede arrancar en modo transitorio con números de Numi, sobre la misma interfaz de canal. Queda fijado que las credenciales de WhatsApp son **por organización** (`waba_id`, `phone_number_id`, token cifrado), nunca un token global.

  Hallazgo que definió la decisión: los límites de Meta son **por portafolio y compartidos entre todos sus números** (250 destinatarios únicos cada 24 h; 2 números sin verificación, 20 con ella). El plan de alojar los números de todos los clientes en el portafolio de Numi no escala: un cliente consumiría la cuota de los demás y hay un tope duro de 20 clientes.

  Se confirmó además que el segmento inicial pasa a ser inmobiliarias, clínicas estéticas y odontologías, con **el mismo producto** para los tres; los nichos específicos se analizan después de validar la idea.

- **Quedó abierto:**
  - Aprobación de ADR-001 por Franco y Anthony (hoy en estado Propuesta).
  - **Riesgo principal del proyecto:** la app de Numi AI ya está publicada y aprobada, pero la verificación de negocio ante Meta está frenada porque faltan datos, y después queda la habilitación como Tech Provider. Son dos pasos, es la ruta crítica y no la controla el equipo. Si no se resuelve hacia mediados de noviembre, el piloto queda limitado a dos clientes.
  - Por confirmar con Meta: qué permisos tiene aprobados la app (`whatsapp_business_management`, `whatsapp_business_messaging`) y si la sección de Tech Provider está visible o bloqueada hasta verificar el negocio.
  - Actualizaciones pendientes en los documentos fuente: §3 de la definición del MVP (segmento), §8.1 y §12 (modelo Tech Provider y ruta crítica), y §5.5 de la consola (`RF-PLT-WA-08` sube a P0 al obtener el rol).

- **Siguiente paso:** D-02, alcance de la consola (completa o versión mínima). En paralelo y fuera de la arquitectura: destrabar la verificación de negocio con Meta.

## 2026-10-14 — Repositorio creado

- **Se decidió o produjo:** repositorio con CLAUDE.md y los tres documentos de docs/ (definición del MVP v2.2, documento maestro v1.1, consola de plataforma v1.1).
- **Quedó abierto:** las 11 decisiones de la sección 5 de CLAUDE.md.
- **Siguiente paso:** D-01, modelo de conexión de WhatsApp.
