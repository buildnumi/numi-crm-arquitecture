# CLAUDE.md — Numi CRM · Arquitectura

Este archivo le da a Claude el contexto, el objetivo y el método de trabajo de este repositorio. Se lee al inicio de cada sesión. Si algo aquí contradice una instrucción de la conversación, manda la conversación; si contradice un documento de `/docs`, avisa del conflicto antes de seguir.

---

## 0. Cómo arranca cada sesión

Franco o Anthony abren la sesión con algo corto, por ejemplo "hola", "iniciemos el planning de la arquitectura" o "sigamos". **No hace falta más contexto: todo está en este repositorio.** Al recibir ese mensaje, Claude hace esto, en orden:

1. **Lee el estado:** este archivo, `docs/PROGRESO.md`, la carpeta `docs/adr/` y, si existen, los archivos de `docs/architecture/`.
2. **Reporta el estado en un máximo de 6 líneas:** qué decisiones están aprobadas, cuál está en curso, qué entregables existen y cuál es el siguiente paso.
3. **Propone la agenda de la sesión:** el siguiente paso pendiente según el orden de la sección 5, con lo que se va a producir al final. Espera confirmación antes de empezar.
4. **Trabaja el paso** con el método de la sección 6 (decisiones) o de la sección 7 (arquitectura).
5. **Cierra la sesión** actualizando la tabla de la sección 5 y agregando una entrada en `docs/PROGRESO.md`: fecha, qué se decidió o produjo, qué quedó abierto y cuál es el siguiente paso.

Si es la primera sesión (`docs/PROGRESO.md` no existe o solo tiene la entrada de creación del repositorio), Claude primero confirma que encuentra los tres documentos de `docs/`, resume el proyecto en 5 líneas y propone empezar por la decisión D-01.

---

## 1. Objetivo de este repositorio

Este repositorio existe para dos cosas, y termina cuando ambas están hechas:

1. **Resolver todas las decisiones pendientes** del proyecto que afectan la arquitectura (sección 5), cada una documentada en un registro de decisión (ADR).
2. **Construir la arquitectura completa del sistema, sólida y escalable,** que cubra frontend, backend, datos, integraciones, seguridad y despliegue, con sus diagramas (sección 7).

El diseño detallado de la base de datos y el contrato completo de la API son la etapa siguiente. Esta etapa debe dejar definidos sus límites (qué módulos, qué datos son de quién y qué viaja entre las piezas).

## 2. Qué es Numi CRM

**Numi CRM** es una nueva línea de negocio de Numi AI (Medellín, Colombia): un CRM conversacional **multiempresa** para WhatsApp, vendido como servicio (SaaS) a negocios colombianos.

- **Objetivo del MVP:** validar con negocios reales que un equipo puede atender, organizar y medir sus conversaciones de WhatsApp desde un solo lugar.
- **Fecha límite:** MVP terminado el **30 de noviembre de 2026**. Piloto gratuito de una semana con clínicas estéticas a partir del 1 de diciembre.
- **Equipo:** dos socios, **Franco** y **Anthony**. Ambos desarrollan. No hay más personas en el proyecto.
- **Dos aplicaciones de cara al usuario:**
  - el **CRM**, para los asesores y administradores de cada negocio;
  - la **consola de plataforma**, para que el equipo de Numi administre las organizaciones.
- **Segmento inicial:** clínicas estéticas. Después: inmobiliarias, odontologías y veterinarias.

## 3. Fuentes de verdad

| Archivo                              | Qué contiene                                                                              | Para qué se usa                             |
| ------------------------------------ | ----------------------------------------------------------------------------------------- | ------------------------------------------- |
| `docs/01_definicion_mvp_v2.2.md`     | Alcance, stack propuesto, cronograma, riesgos y decisiones pendientes                     | Alcance, plan y restricciones               |
| `docs/02_documento_maestro_v1.1.md`  | 63 requisitos funcionales (RF-AUT, CNV, CON, EST, MET, USR, CFG, TRV) y 18 vistas del CRM | Qué debe hacer el CRM y qué ve cada usuario |
| `docs/03_consola_plataforma_v1.1.md` | 52 requisitos (RF-PLT-\*) y 13 vistas de la consola                                       | Administración de organizaciones            |
| `docs/adr/`                          | Registros de decisión                                                                     | Decisiones tomadas                          |
| `docs/architecture/`                 | Documento de arquitectura y diagramas                                                     | Entregable principal                        |
| `docs/PROGRESO.md`                   | Bitácora de sesiones                                                                      | Continuidad entre sesiones                  |

**Orden de prioridad cuando dos fuentes no coinciden:**

1. Un ADR con estado **Aprobado** manda en lo técnico.
2. El documento maestro y la consola mandan en el detalle funcional.
3. La definición del MVP manda en alcance, fechas y restricciones.

Si encuentras una contradicción entre documentos, **no la resuelvas en silencio**: señálala con las dos citas y propone cómo resolverla.

## 4. Reglas de trabajo para Claude

1. **No inventes.** Si falta un dato para decidir o diseñar, pregunta. Si propones algo que no está en los documentos, márcalo como _propuesta_ y explica por qué.
2. **Cita siempre.** Justifica con el documento y la sección, o con el identificador del requisito (por ejemplo, `RF-CNV-08`, `RF-PLT-WA-02`).
3. **Separa hechos, supuestos y recomendaciones.**
4. **Verifica los datos de proveedores.** Todo dato sobre Meta (WhatsApp Cloud API), Supabase, Cloudflare, Vercel u otro proveedor (límites, precios, compatibilidad, políticas) se verifica en su documentación oficial, con enlace. Tu conocimiento puede estar desactualizado: ya pasó con el soporte de Fastify en Cloudflare Workers.
5. **Opciones antes que respuestas.** En cada decisión, presenta de 2 a 3 opciones con ventajas, desventajas y costo para _este_ contexto (dos personas, siete semanas, poco volumen). Recomienda una y explica por qué. **Franco y Anthony deciden;** nunca marques un ADR como Aprobado sin su confirmación explícita.
6. **Una decisión por sesión,** salvo que pidan avanzar más.
7. **No amplíes el alcance.** Lo que está en "Fuera del MVP" (sección 6 de la definición y sección 12 del maestro) no entra. Si crees que algo debería entrar, propónlo como cambio de alcance con su costo.
8. **No modifiques `docs/01–03` ni ADR aprobados** salvo que se pida. Si una decisión los afecta, lista qué habría que actualizar.
9. **Diseña para el volumen real.** Toda pieza nueva debe justificar qué problema concreto resuelve hoy, y decir cuándo dejaría de bastar.
10. **Escribe en español,** claro y directo, sin relleno.

## 5. Decisiones pendientes

Este es el tablero del proyecto. Claude lo mantiene al día.

La columna **Depende de** refleja lo que declara el encabezado de cada ADR. Si un ADR cambia sus dependencias, esta tabla se actualiza con él.

| ID   | Decisión                                                                                                                                                                        | Tipo               | Depende de (ADR)                     | Estado    |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ | ------------------------------------ | --------- |
| D-01 | Modelo de conexión de WhatsApp: coexistencia o número solo en la API                                                                                                            | Producto y técnica | —                                    | Aprobada (ADR-001) |
| D-02 | Alcance de la consola: completa o versión mínima                                                                                                                                | Producto           | 001                                  | Aprobada (ADR-002) |
| D-03 | Visualización de audios, imágenes y documentos en el MVP                                                                                                                        | Producto           | 001, 003                             | Aprobada (ADR-004) |
| D-04 | Runtime del backend: Hono o Express en Workers, o Fastify en un servicio Node                                                                                                   | Técnica            | 001                                  | Aprobada (ADR-003) |
| D-05 | Procesamiento del webhook: cola, idempotencia, orden y reintentos                                                                                                               | Técnica            | 001, 003, 004                        | Aprobada (ADR-005) |
| D-06 | Aislamiento multiempresa: cómo se aplica la seguridad por fila desde la API                                                                                                     | Técnica            | 003, 005                             | Aprobada (ADR-006) |
| D-07 | Tiempo real en la bandeja                                                                                                                                                       | Técnica            | 003, 005, 006, 007                   | Aprobada (ADR-008) |
| D-08 | Identidad y roles: Supabase Auth, rol de plataforma, doble factor, sesiones                                                                                                     | Técnica            | 002, 003, 006                        | Aprobada (ADR-007) |
| D-09 | Arquitectura del frontend: framework y hosting, CRM y consola en la misma aplicación o separadas, manejo de sesión, obtención de datos y estado, integración con el tiempo real | Técnica            | 002, 003, 007, 008                   | Aprobada (ADR-009) |
| D-10 | Archivos: si se usa R2 y para qué                                                                                                                                               | Técnica            | 003, 004, 006, 007                   | Aprobada (ADR-010) |
| D-11 | Secretos y tokens cifrados, correo propio, entornos, despliegue y observabilidad                                                                                                | Técnica            | 001, 003, 005, 007, 008, 009, 010    | Aprobada (ADR-011) |
| D-12 | Envío de mensajes: ventana de 24 h, estados y reintentos                                                                                                                       | Técnica            | 001, 003, 005, 006, 007, 008         | Aprobada (ADR-012) |
| D-13 | Cálculo y almacenamiento de las métricas                                                                                                                                        | Técnica            | 003, 006                             | Aprobada (ADR-013) |
| D-14 | Agente de IA configurable por negocio: activación, escalada, configuración y tratamiento de datos                                                                               | Producto y técnica | 001, 003, 005, 006, 007, 008, 012    | Aprobada (ADR-014) |

**Orden de trabajo:** D-01 → D-02 → D-04 → D-03 → D-05 → D-06 → D-08 → D-07 → D-09 → D-10 → D-11 → D-12 → D-13. D-12 y D-13 se agregaron el 2026-10-15 al auditar el tablero: cubren el envío de mensajes y las métricas, que ninguna de las once decisiones originales resolvía. D-14 se agregó el 2026-10-15 por decisión de Franco: incorpora al MVP el agente de IA que `docs/01_definicion_mvp_v2.2.md` §6 excluía. Si una decisión depende de otra todavía pendiente, Claude lo dice y propone resolver primero la dependencia.

**Estados posibles:** Pendiente · En curso · Propuesta (ADR escrito, falta aprobación) · Aprobada · Reemplazada.

Cada decisión técnica o de producto que afecte la arquitectura se documenta como ADR en `docs/adr/NNN-titulo.md`, usando el número de la decisión (D-01 → `001-modelo-conexion-whatsapp.md`).

## 6. Método para cada decisión

1. **Contexto:** qué problema resuelve, qué requisitos toca (con sus RF) y qué restricciones aplican.
2. **Investigación:** datos verificados de proveedores, con enlaces.
3. **Opciones:** de 2 a 3, con ventajas, desventajas, esfuerzo, costo y riesgo.
4. **Recomendación** con su razonamiento.
5. **Decisión** de Franco y Anthony.
6. **ADR escrito** con la plantilla de la sección 10, estado _Propuesta_ hasta que lo aprueben.
7. **Impacto:** qué cambia en los documentos de `docs/`, en otras decisiones y en el cronograma.

## 7. Entregable de arquitectura

Cuando las decisiones D-01 a D-14 estén aprobadas, se construye `docs/architecture/arquitectura.md`. Debe cubrir frontend, backend, datos, integraciones, seguridad y despliegue, con diagramas en Mermaid:

| Sección                                  | Contenido                                                                                                                                                                                                              |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Requisitos no funcionales             | Tabla con números: negocios esperados a 12 meses, volumen por negocio, mensajes por segundo derivados, latencia de la bandeja (menos de 5 s), disponibilidad, retención, costo mensual máximo                          |
| 2. Contexto (C4 nivel 1)                 | El sistema, sus usuarios (asesor, administrador, operador de Numi) y los sistemas externos (Meta, correo, proveedores de nube)                                                                                         |
| 3. Contenedores (C4 nivel 2)             | Aplicación del CRM, consola, API, procesador de webhooks, cola, base de datos, tiempo real, almacenamiento de archivos, correo, y cómo se comunican                                                                    |
| 4. Componentes del backend (C4 nivel 3)  | Módulos: identidad y organizaciones, canal (WhatsApp detrás de una interfaz de canal), conversaciones y mensajes, contactos, asignación, métricas, consola, auditoría. Qué datos es dueño cada uno y qué eventos emite |
| 5. Componentes del frontend (C4 nivel 3) | Estructura de la aplicación, rutas del CRM y de la consola, manejo de sesión y roles, obtención de datos, estado, tiempo real, componentes compartidos del design system                                               |
| 6. Flujos críticos                       | Diagramas de secuencia de los 5 flujos de la sección 8                                                                                                                                                                 |
| 7. Multiempresa y seguridad              | Cómo se aísla cada negocio en cada capa; recorrido de los datos sensibles; dónde viven los secretos; qué ve el rol de plataforma                                                                                       |
| 8. Modos de falla                        | Qué pasa si falla cada pieza (Meta, cola, base, tiempo real, correo) y cómo se recupera                                                                                                                                |
| 9. Despliegue                            | Entornos, servicios por entorno, cómo se despliega y cómo se revierte                                                                                                                                                  |
| 10. Observabilidad                       | Registros, errores y alertas mínimas para operar el piloto                                                                                                                                                             |
| 11. Escalabilidad                        | Qué aguanta el diseño, primer cuello de botella esperado y señales para revisarlo                                                                                                                                      |
| 12. Trazabilidad                         | Cada requisito P0 enlazado con el componente que lo cumple                                                                                                                                                             |

**Terminado cuando:** las 14 decisiones están aprobadas, el documento tiene las 12 secciones y una revisión independiente (sección 11) no encuentra problemas abiertos.

> **Los tres criterios se cumplieron el 2026-10-16. Esta etapa está cerrada.** Las 14 decisiones aprobadas, las 12 secciones escritas y la revisión independiente hecha en una sesión que no participó del diseño: encontró siete huecos de especificación —ninguna decisión equivocada—, los siete corregidos en sus ADR. El trabajo que sigue es de desarrollo, no de arquitectura; lo que queda pendiente está en la cabecera de `docs/PROGRESO.md`.

## 8. Restricciones y requisitos que no se negocian

**Restricciones**

| Restricción           | Detalle                                                                                                                                         |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Equipo                | Dos personas desarrollando, con tiempo parcial                                                                                                  |
| Fecha                 | MVP el 30 de noviembre de 2026; revisión de avance el 16 de noviembre                                                                           |
| Canal                 | Solo WhatsApp, por la API oficial de Meta (Cloud API) y sus webhooks                                                                            |
| Plataforma            | Aplicación web para computador; sin versión móvil en el MVP                                                                                     |
| Idioma y región       | Español (Colombia), zona horaria de Bogotá                                                                                                      |
| Volumen de referencia | 500 conversaciones al mes por negocio                                                                                                           |
| Legal                 | Ley 1581 de 2012. Las conversaciones pueden contener **datos sensibles** (salud). El negocio es responsable del tratamiento y Numi es encargado |
| Costo                 | Pocos costos fijos; verificar precios vigentes antes de recomendar planes                                                                       |

**Requisitos que no se negocian** (si una propuesta los pone en riesgo, dilo explícitamente):

- **Aislamiento entre negocios,** verificado con pruebas automáticas (`RF-TRV-01`).
- **Webhook confiable:** firma de Meta validada, sin duplicados ante reintentos (clave única por identificador de mensaje de Meta), sin pérdidas (`RF-TRV-02`).
- **Orden de los mensajes** por la hora de envío, aunque lleguen desordenados (`RF-CNV-06`).
- **Ventana de 24 horas** de Meta aplicada antes de enviar (`RF-CNV-08`).
- **Estados del mensaje:** enviando, enviado, entregado, leído y error (`RF-CNV-09`).
- **Datos sensibles fuera de la consola,** los registros técnicos y las métricas (`RF-TRV-07`, `RF-PLT-USO-04`). **Única excepción, decidida el 2026-10-15 (ADR-014):** el agente de IA envía el texto de la conversación a la API de Anthropic. No viajan el nombre ni el teléfono del contacto, ni los adjuntos. Exige la preparación para HIPAA habilitada antes de conectar un cliente real.
- **Tokens cifrados,** nunca en el navegador ni en los registros (`RF-PLT-SEG-02`).
- **Consola:** acciones en el servidor con verificación del rol en cada petición, y auditoría de cada acción (`RF-PLT-SEG-01`, `RF-PLT-AUD-01`).
- **Organización suspendida:** nadie entra ni envía, pero los mensajes entrantes se guardan (`RF-AUT-07`, `RF-CNV-16`).

**Flujos críticos** (deben tener diagrama de secuencia):

1. Mensaje entrante, desde Meta hasta la bandeja.
2. Envío de un mensaje con control de la ventana de 24 horas y los estados que informa Meta.
3. Asignación automática de una conversación nueva.
4. Alta de una organización desde la consola, con la invitación del primer administrador.
5. Suspensión y reactivación de una organización.

## 9. Stack de partida y hechos verificados

Propuesto en el brief; las decisiones pueden cambiarlo.

**Stack decidido** (actualizado el 2026-10-15; las 14 decisiones están aprobadas).

| Capa                    | Decidido                                                                       | ADR |
| ----------------------- | ------------------------------------------------------------------------------ | --- |
| Lenguaje                | Node.js con TypeScript                                                         | —   |
| Backend                 | **Hono** en Cloudflare Workers                                                 | 003 |
| Frontend                | **Página única con React y Vite**, servida desde Workers                       | 009 |
| Base de datos           | PostgreSQL administrado (Supabase), conectado con Cloudflare Hyperdrive        | 003 |
| Autenticación           | Supabase Auth **solo como identidad**; la autorización vive en el backend      | 007 |
| Archivos                | **Cloudflare R2**, servido por el Worker; sin URL firmadas                     | 004, 010 |
| Procesamiento asíncrono | **Tres colas** de Cloudflare Queues: eventos, archivos y agente                | 005, 014 |
| Tiempo real             | **Supabase Realtime, Broadcast desde el servidor**                             | 008 |
| Correo                  | **Resend**, con dominio propio                                                 | 011 |
| Agente de IA            | **API de Anthropic**, detrás de su propia cola                                 | 014 |
| Código y despliegue     | GitHub, con entornos de desarrollo y producción                                | 011 |

Costos fijos: 30 USD al mes (Workers 5, Supabase Pro 25), más el costo variable del agente y los mensajes de Meta.

**Hechos ya verificados** (reconfírmalos si una decisión depende de ellos):

- Según Cloudflare, Express y Koa funcionan en Workers, pero **Fastify todavía no**.
- El servidor de correo por defecto de Supabase **no es para producción**: solo entrega a miembros del equipo del proyecto y con un límite muy bajo.
- Meta permite la **coexistencia** (app de WhatsApp Business y Cloud API en el mismo número), pero exige ser Tech Provider o Solution Partner y usar el registro integrado de Meta (Embedded Signup). Los números en coexistencia tienen un throughput fijo y el CRM debe mostrar los mensajes enviados desde la app.

**Criterios de diseño** (propuestos; las decisiones pueden ajustarlos):

- Backend modular, no microservicios. WhatsApp detrás de una interfaz de canal.
- Toda tabla con datos de un negocio lleva `organization_id`, y sus relaciones usan claves foráneas compuestas (`organization_id`, `id`).
- El contacto se reconoce por identidades por canal; el responsable del contacto se deriva de su conversación.
- Fechas en `timestamptz`; presentación en hora de Bogotá.
- Métricas con consultas o vistas SQL; no hace falta un almacén de datos a este volumen.
- Código, tablas y endpoints en inglés; documentación en español; diagramas en Mermaid.

## 10. Plantilla de ADR

```markdown
# ADR-NNN: Título

- **Estado:** Propuesta | Aprobada | Reemplazada por ADR-XXX
- **Fecha:** AAAA-MM-DD
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-...
- **Depende de:** ADR-...

## Contexto

Qué problema hay que resolver y por qué ahora. Restricciones que aplican.

## Opciones consideradas

### Opción A: ...

- Ventajas
- Desventajas
- Esfuerzo, costo y riesgo
- Fuentes verificadas (enlaces)

### Opción B: ...

## Decisión

Qué se eligió y por qué.

## Consecuencias

Qué se vuelve más fácil, qué se vuelve más difícil, qué hay que construir y qué documentos de `docs/` hay que actualizar.

## Qué revisar cuando el sistema crezca

Señales concretas que indicarían que hay que reconsiderar esta decisión.
```

## 11. Revisión independiente

Cada ADR y el documento de arquitectura se revisan en una **sesión nueva** de Claude, que no participó en su diseño. Instrucción para esa sesión: "Actúa como revisor. Revisa `<archivo>` contra `docs/` y la lista de la sección 11 de CLAUDE.md. Reporta solo problemas concretos, con cita".

- [ ] ¿Algún flujo o consulta puede cruzar datos entre negocios?
- [ ] ¿Algún mensaje entrante se puede perder o duplicar? ¿Qué pasa si Meta reintenta o llegan desordenados?
- [ ] ¿Qué pasa si falla cada pieza (cola, base, tiempo real, Meta, correo)?
- [ ] ¿El contenido de las conversaciones puede llegar a la consola, los registros o las métricas?
- [ ] ¿Los tokens y secretos pueden llegar al navegador o a los registros?
- [ ] ¿Cada requisito P0 relacionado está cubierto? ¿Hay algo diseñado que no corresponde a ningún requisito?
- [ ] ¿El frontend y el backend coinciden en sesión, roles, tiempo real y manejo de errores?
- [ ] ¿Hay piezas que no se justifican con el volumen real?
- [ ] ¿Los datos de proveedores están verificados con su documentación oficial?
- [ ] ¿Cabe en el tiempo y el equipo disponibles?

## 12. Estructura del repositorio

```
numi-crm/
├── CLAUDE.md
└── docs/
    ├── 01_definicion_mvp_v2.2.md
    ├── 02_documento_maestro_v1.1.md
    ├── 03_consola_plataforma_v1.1.md
    ├── PROGRESO.md
    ├── adr/
    └── architecture/
        └── arquitectura.md
```

## 13. Glosario

| Término                | Significado                                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------------------------- |
| Organización o negocio | Cliente de Numi CRM; límite de aislamiento de los datos                                                         |
| Operador de plataforma | Persona del equipo de Numi que usa la consola; no pertenece a ninguna organización                              |
| Asesor / Administrador | Roles de los usuarios de un negocio en el CRM                                                                   |
| Contacto               | Persona que escribe al negocio; se reconoce por su identidad en cada canal                                      |
| Conversación           | Hilo continuo de mensajes con un contacto; puede estar abierta o cerrada                                        |
| Responsable            | Usuario asignado a una conversación                                                                             |
| Ventana de 24 horas    | Regla de Meta: solo se puede enviar texto libre dentro de las 24 horas siguientes al último mensaje del cliente |
| Coexistencia           | Uso simultáneo de la app de WhatsApp Business y la Cloud API en el mismo número                                 |
| P0 / P1                | Indispensable para el MVP / puede llegar después                                                                |
