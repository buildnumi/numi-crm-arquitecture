# ADR-015: Plan de Supabase para el MVP

- **Estado:** Aprobada
- **Fecha:** 2026-10-16
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-TRV-02, RF-TRV-03, RF-AUT-07
- **Depende de:** ADR-003, ADR-008, ADR-011
- **Revisa a:** ADR-011, punto 3 ("El plan de Supabase")

## Contexto

ADR-011 fijó **plan Pro en producción, 25 USD al mes**, con una razón decisiva: el plan gratuito pausa el proyecto tras una semana de inactividad, y el piloto empieza el 1 de diciembre después de días de poca actividad.

El 2026-10-16 Franco plantea que el presupuesto del MVP no admite ese costo fijo, y aporta un dato que ADR-011 no tuvo en cuenta: **el equipo va a usar el sistema todos los días durante la construcción**, lo que impide que la pausa llegue a dispararse.

Esta decisión revisa únicamente la elección de plan. Todo lo demás de ADR-011 —secretos, cifrado, Resend, entornos, despliegue y observabilidad— sigue vigente sin cambios.

### Restricciones que aplican

- Pocos costos fijos; verificar precios vigentes (`CLAUDE.md` §8).
- MVP el 30 de noviembre de 2026, piloto el 1 de diciembre con clínicas estéticas.
- Datos sensibles de salud bajo la Ley 1581 de 2012; Numi es **encargado** del tratamiento.
- `RF-TRV-02`: no se pierde ni se duplica ningún mensaje entrante.
- Volumen del piloto: 2 a 3 negocios, 10.000 mensajes por negocio al mes (`arquitectura.md` §1.1).

### Datos verificados

Verificados el 2026-10-16 contra la documentación oficial — [Supabase Pricing](https://supabase.com/pricing), [Project Pausing](https://supabase.com/docs/guides/platform/free-project-pausing), [Realtime Limits](https://supabase.com/docs/guides/realtime/limits).

**Capacidad del plan gratuito contra el volumen del piloto:**

| Límite | Plan gratuito | Uso en el piloto | Margen |
| --- | --- | --- | --- |
| Realtime, conexiones concurrentes | 200 | ~30 asesores | **~7×** |
| Realtime, mensajes al mes | 2.000.000 | ~30.000 avisos | **~65×** |
| Usuarios activos al mes | 50.000 | decenas | enorme |
| **Conexiones directas** | **60** | — | **igual que Micro del plan Pro** |
| Tamaño de la base | 500 MB | ~30.000 mensajes ≈ 30 a 60 MB | **~10×** |
| Egress | 5 GB al mes | los adjuntos van por R2 (ADR-010) | amplio |

**La capacidad no es el problema.** Ningún límite del plan gratuito se acerca al volumen del piloto.

**Dos consecuencias que conviene fijar porque son contraintuitivas:**

1. **Las conexiones directas son 60 en ambos planes.** El hallazgo de `arquitectura.md` §11.2 —fijar `max_size` de Hyperdrive en 40— **no cambia con esta decisión**. El número sigue siendo 40 por la misma razón.
2. **El egress de Supabase casi no se usa**, porque ADR-010 puso los adjuntos en R2. Los 5 GB del plan gratuito no son una restricción real.

**Lo que el plan gratuito no da:**

| Qué | Plan gratuito | Plan Pro |
| --- | --- | --- |
| Pausa por inactividad | **Tras 7 días de poca actividad** | No se pausa |
| Copias de seguridad | **Ninguna** | Diarias, retenidas 7 días |
| Retención de registros | **1 día** | 7 días |
| Registros de auditoría de Auth | 1 hora | 7 días |

**La regla exacta de la pausa**, citada de la documentación oficial:

> *"Projects under a paid plan cannot be paused and are not subject to automatic pausing for inactivity."*

Y sobre qué la evita:

> *"Typically a few user requests to the database each day over the previous week is enough"*

Avisan con un correo una semana antes de pausar y otro al pausarse. Un proyecto pausado se reanuda **a mano** desde el panel, y admite restauración hasta un año después.

## Opciones consideradas

### Opción A: plan Pro en producción desde ya (lo que fijó ADR-011)

- **Ventajas:** sin pausa, con copias diarias y 7 días de registros; es la opción que no exige construir nada.
- **Desventajas:** 25 USD al mes de costo fijo desde octubre, antes de que haya un solo cliente pagando. Son ~50 USD gastados antes del piloto, para cubrir un riesgo que durante la construcción no existe.
- **Esfuerzo, costo y riesgo:** esfuerzo nulo, costo 25 USD al mes, riesgo nulo.

### Opción B: plan gratuito en ambos entornos, con mitigaciones

- **Ventajas:** costo fijo total del proyecto de 5 USD al mes en lugar de 30. La capacidad sobra por un margen de 7× a 65×. La pausa se evita de forma mecánica con el Cron Trigger que **ya existe** (ADR-005, ADR-012) para la limpieza por retención.
- **Desventajas:** **no hay copias de seguridad del proveedor**; hay que construirlas. Los registros duran 1 día, lo que impide depurar el lunes un incidente del viernes. Si el cron falla en silencio, el proyecto se pausa y nadie se entera hasta que la bandeja deja de actualizarse.
- **Esfuerzo, costo y riesgo:** medio día de trabajo (latido y copias), costo 0 USD, **riesgo asumido en durabilidad**.

### Opción C: plan gratuito hasta mediados de noviembre, Pro durante el piloto

- **Ventajas:** ahorra ~37 USD y compra copias diarias justo cuando entran datos reales de clínicas.
- **Desventajas:** exige acordarse de contratar y migrar en plena recta final; el cambio de plan no es instantáneo en su efecto sobre las copias —la primera copia es posterior a la contratación.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo, costo 25 USD un mes, riesgo de olvido en la semana más cargada del calendario.

## Decisión

**Plan gratuito de Supabase en ambos entornos durante todo el MVP. La migración al plan Pro se hace cuando la idea esté validada**, es decir, cuando el piloto confirme que hay negocio.

Es la **Opción B**. Decidido por Franco el 2026-10-16, con el argumento de que el sistema se usará a diario durante la construcción y la pausa por tanto no llega a dispararse.

### Dos mitigaciones obligatorias

No son recomendaciones: sin ellas la decisión no se sostiene.

**1. Latido diario contra la base.** El Cron Trigger del Worker de la API —que ya corre a diario por ADR-005 y ADR-012— ejecuta además un `SELECT 1`. Convierte "nos vamos a acordar de entrar" en una garantía mecánica que no depende de la disciplina de dos personas ocupadas.

El riesgo que queda: si el cron falla en silencio, nos enteramos cuando el proyecto ya está pausado. Se cubre con la alerta de webhook caído que ADR-011 ya exige.

**2. Copias de seguridad propias.** `pg_dump` programado desde GitHub Actions hacia un bucket de R2 —que ya existe por ADR-010 y tiene margen gratuito de sobra—. **Hay que probar una restauración antes del piloto**; ADR-011 ya lo pedía y sigue valiendo: una copia que nunca se probó no es una copia.

### Por qué las copias no son opcionales

Esta es la parte que la decisión de plan no resuelve por sí sola, y conviene dejarla escrita sin suavizar.

**La pausa se evita con uso; la ausencia de copias no se evita con nada.** No hay actividad, disciplina ni cuidado que sustituya un respaldo. Y el piloto maneja conversaciones de clínicas estéticas —datos de salud bajo la Ley 1581, donde Numi responde como encargado del tratamiento.

Un error de migración, un borrado sin filtro o una corrupción dejan el sistema sin punto de retorno. ADR-011 fijó migraciones **solo aditivas** durante el MVP precisamente porque la reversión del código no cubre el esquema; esa regla se vuelve más importante todavía sin copias del proveedor.

### Cuándo migrar a Pro

**Disparador principal:** el piloto valida la idea y entra el primer cliente de pago.

**Disparadores adicionales, cualquiera que llegue antes:**

- Se da de alta una organización que no sea del piloto gratuito.
- La base pasa de **300 MB** (60 % del límite de 500 MB).
- Hace falta depurar un incidente con más de 1 día de antigüedad, y la falta de registros lo impide.
- El proyecto se pausa una sola vez con datos reales dentro.

Migrar es un cambio de plan en el panel; no exige trabajo de ingeniería. Lo que sí hay que hacer al migrar: **revisar `max_size` de Hyperdrive si además se cambia el tamaño de cómputo** (`arquitectura.md` §11.2).

## Consecuencias

### Más fácil

- El costo fijo del proyecto baja de **30 a 5 USD al mes**, el 83 %.
- No hay que contratar nada para empezar a construir.
- Las copias propias con `pg_dump` son portables: no atan el respaldo al proveedor, lo que acompaña bien el riesgo de concentración en Supabase que ADR-011 registró.

### Más difícil

- **La durabilidad pasa a ser responsabilidad del equipo.** Lo que antes compraban 25 USD ahora es código nuestro que hay que escribir, programar y probar.
- Depurar incidentes con más de un día de antigüedad deja de ser posible. La sección 10 de `arquitectura.md` contaba con 7 días de registros.
- Aparece una dependencia nueva: si el cron del latido se rompe, el proyecto se pausa. Es una pieza más que vigilar.
- Durante el piloto, con clientes reales, se opera sin copias del proveedor. **Es el riesgo que esta decisión asume de forma consciente**, y la razón por la que las copias propias no son negociables.

### Qué hay que construir

- `SELECT 1` diario en el Cron Trigger del Worker de la API.
- Flujo de `pg_dump` en GitHub Actions, con destino a un bucket de R2 y su propia retención.
- Una restauración probada en desarrollo **antes del 30 de noviembre**.

### Qué documentos hay que actualizar

- `docs/adr/011-secretos-entornos-despliegue.md`: el punto 3 queda revisado por este ADR; la tabla de costo mensual y la frase "las copias de seguridad las provee el plan Pro" ya no son ciertas.
- `docs/architecture/arquitectura.md` §1.4 (durabilidad), §1.6 (costo), §9 (tabla de entornos) y §9.4.
- `docs/desarrollo/00_arranque.md`: la lista de puesta en marcha dice "Supabase Pro en producción".
- `CLAUDE.md` §9: costo fijo mensual.
- `docs/01_definicion_mvp_v2.2.md` §8.1: los costos fijos registrados.

## Qué revisar cuando el sistema crezca

- **Al validar la idea con el primer cliente de pago:** migrar a Pro. Es el disparador principal y el que motivó esta decisión.
- **Al llegar a 300 MB de base:** quedan 200 MB de margen, que con 12 meses de retención se consumen rápido. Es la señal de capacidad más cercana.
- **Si el proyecto se pausa alguna vez:** revisar por qué falló el latido antes de culpar al plan. Si el cron funcionaba y se pausó igual, la regla de actividad cambió y hay que releerla.
- **Tras la primera restauración probada:** anotar cuánto tardó. Es el dato que falta para hablar de objetivo de tiempo de recuperación, que `arquitectura.md` §8 dejó sin fijar.
- **Si el equipo crece o aparece un segundo piloto:** 2 proyectos activos es el límite del plan gratuito por cuenta. Un tercer entorno exige Pro.
