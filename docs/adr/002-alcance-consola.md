# ADR-002: Alcance de la consola de plataforma en el MVP

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-PLT-ACC-01 a RF-PLT-SEG-05 (los 52 de `docs/03_consola_plataforma_v1.1.md`), RF-USR-01 a RF-USR-07, RF-AUT-07, RF-CNV-16
- **Depende de:** ADR-001

## Contexto

La consola de plataforma es el área interna desde la que el equipo de Numi administra todas las organizaciones (`docs/03_consola_plataforma_v1.1.md` §1). Tiene **52 requisitos** y **13 vistas**. El documento ya trae una priorización: 44 requisitos en P0 y 8 en P1; 11 vistas en P0 y 2 en P1 (P10, panel general, y P30, auditoría global).

Hay que decidir **cuánto de eso se construye para el MVP del 30 de noviembre de 2026**, porque la consola compite por las mismas siete semanas que el CRM, que tiene 63 requisitos y 18 vistas (`docs/02_documento_maestro_v1.1.md`).

### Restricciones que aplican

- Dos personas desarrollando con tiempo parcial; MVP el 30 de noviembre de 2026, revisión de avance el 16 de noviembre, piloto el 1 de diciembre (`CLAUDE.md` §8).
- Volumen del piloto: pocas organizaciones. Por ADR-001, el piloto corre en modo transitorio con números del portafolio de Numi, lo que lo limita a **2 clientes** mientras el negocio no esté verificado ante Meta y a **20** una vez verificado.
- Dos operadores conocidos y fijos: Franco y Anthony. La propia consola decide no construir gestión de operadores por esa razón (`docs/03_consola_plataforma_v1.1.md` §2, decisión 9).
- Las conversaciones pueden contener datos sensibles de salud; la consola nunca muestra su contenido (`CLAUDE.md` §8, `RF-TRV-07`, `RF-PLT-USO-04`).

### Confirmación de Franco y Anthony (2026-10-15)

Tres puntos quedaron confirmados en esta sesión y encuadran la decisión:

1. **La consola es la herramienta de administración de Numi, no un autoservicio del cliente.** El alta de cada negocio la hace un operador desde la consola; no existe registro público.
2. **Los asesores de cada negocio los invita el administrador del negocio desde el CRM** (`RF-USR-01` a `RF-USR-07`), no el operador desde la consola. La consola crea e invita únicamente al **primer administrador** (`RF-PLT-ORG-06`).
3. **El doble factor del operador se resuelve con lo que ofrezca el proveedor de identidad** (TOTP de Supabase Auth), no con un flujo propio. La confirmación técnica corresponde a D-08.

### El punto que hay que resolver

El documento de la consola fue escrito pensando en un producto de administración maduro. Varias cosas marcadas P0 lo están porque se asumió una escala que el MVP no tiene: buscar por NIT entre tres organizaciones, filtrar por cinco criterios una lista que cabe en una pantalla, un asistente de tres pasos para un formulario que un operador llena dos veces.

La pregunta no es si esas cosas son buenas —lo son—, sino si construirlas en noviembre de 2026 compra algo que no se pueda comprar en enero de 2027.

Contra eso pesa un argumento real: **la consola es lo que hace al producto multiempresa**. Sin ella, dar de alta un negocio significa escribir en la base de datos a mano, y eso rompe la auditoría (`RF-PLT-AUD-01`), el cifrado de tokens (`RF-PLT-SEG-02`) y la confianza en el aislamiento (`RF-TRV-01`). La consola no es accesoria.

## Opciones consideradas

### Opción A: Consola completa, los 44 requisitos P0 y las 11 vistas P0

Se construye la consola tal como está escrita en `docs/03_consola_plataforma_v1.1.md`, dejando fuera solo lo ya marcado P1: P10 (panel general), P30 (auditoría global), `RF-PLT-ORG-12`, `RF-PLT-USO-02`, `RF-PLT-USO-03`, `RF-PLT-AUD-04`, `RF-PLT-WA-08`.

- **Ventajas:** la consola queda terminada y no hay que volver a ella; el soporte del piloto tiene todas las herramientas desde el día uno (buscar, filtrar, ver actividad por organización, gestionar cualquier usuario); no hay deuda de producto ni una segunda ronda de diseño; el documento ya está escrito y validado, así que no hay que rediscutir el alcance requisito por requisito.
- **Desventajas:** es el mayor consumo de las siete semanas fuera del CRM; buena parte de ese trabajo (buscador, filtros, ordenamiento, asistente de tres pasos, gráficos) no tiene usuario real con dos o tres organizaciones.
- **Esfuerzo, costo y riesgo:** esfuerzo alto. Riesgo concentrado en el calendario: lo que se gasta aquí no se gasta en la bandeja del CRM, que es lo que el piloto valida.
- **Fuentes:** `docs/03_consola_plataforma_v1.1.md` §5 y §6.

### Opción B: Consola mínima operativa

Se construye solo lo que hace falta para operar: P01, P02, L02, P20 (lista sin buscador ni filtros), P21 (formulario de una pantalla en vez de asistente de tres pasos), P22, P24, P26, P27. Fuera del MVP: P23 completa, P25, P10, P30 y los indicadores de uso. La auditoría se **escribe** desde el día uno (`RF-PLT-AUD-01`, `-02`, `-05`) pero no se **muestra**.

- **Ventajas:** aproximadamente 20 requisitos P0 reales en vez de 44; libera semanas para el CRM; conserva intactos todos los no-negociables de `CLAUDE.md` §8.
- **Desventajas:** el soporte del piloto se hace con menos herramientas; sin P25 no hay forma de ver desde la interfaz qué se hizo sobre una organización, aunque el registro exista; hay que volver a la consola en una segunda etapa, con el costo de retomar contexto.
- **Esfuerzo, costo y riesgo:** esfuerzo medio-bajo. Riesgo bajo en lo técnico, medio en lo operativo durante el piloto.

### Opción C: Consola mínima más scripts

La consola solo lee (lista de organizaciones, salud de WhatsApp) y las acciones —alta, suspensión, conexión de WhatsApp— se hacen con scripts contra la base de datos.

- **Ventajas:** el menor esfuerzo de interfaz.
- **Desventajas:** rompe `RF-PLT-AUD-01` (las acciones no quedan registradas con operador y detalle), obliga a manipular producción a mano con credenciales de base de datos, y hace que el token de WhatsApp pase por fuera del camino cifrado de `RF-PLT-SEG-02`. Además contradice el propósito explícito de la consola: "dar soporte y seguir el uso de cada organización **sin depender de scripts ni de la base de datos**" (`docs/03_consola_plataforma_v1.1.md` §1).
- **Esfuerzo, costo y riesgo:** esfuerzo bajo, riesgo alto.
- **Veredicto:** descartada. Compromete requisitos que `CLAUDE.md` §8 marca como no negociables.

## Decisión

**Opción A: la consola se construye completa**, con los 44 requisitos P0 y las 11 vistas P0 de `docs/03_consola_plataforma_v1.1.md`, decidida por Franco y Anthony el 2026-10-15.

Queda fuera del MVP lo ya marcado P1 en el documento, que no se reabre:

| Fuera del MVP | Qué es |
| --- | --- |
| P10 | Panel general de la plataforma |
| P30, `RF-PLT-AUD-04` | Registro global de auditoría con filtros |
| `RF-PLT-ORG-12` | Valor inicial de la asignación automática desde la consola |
| `RF-PLT-USO-02` | Gráfico de uso por día de los últimos 14 días |
| `RF-PLT-USO-03` | Indicadores del panel general |
| `RF-PLT-WA-08` | Registro integrado de Meta (Embedded Signup). Sube a P0 al obtener el rol de Tech Provider (ADR-001) |

Mientras P10 no exista, la ruta `/plataforma` abre la lista de organizaciones (`docs/03_consola_plataforma_v1.1.md` §6.2).

### Precisiones que fija esta decisión

**1. Vía normal de alta de usuarios: el CRM, no la consola.**

El operador crea e invita al **primer administrador** de cada negocio (`RF-PLT-ORG-06`). A partir de ahí, los asesores los invita y gestiona el administrador del negocio desde el CRM (`RF-USR-01` a `RF-USR-07`).

P23 se construye completa igual, pero como **vista de soporte**, no como la vía normal: existe para que un operador pueda desatascar a un negocio cuyo administrador perdió el acceso, reenviar una invitación que no llegó (`F-PLT-2`) o corregir un rol mal asignado. El diseño de P23 no cambia; lo que cambia es que no es el camino por el que entran los asesores.

Consecuencia directa: la funcionalidad de usuarios del CRM (`RF-USR-*`) es **P0 y no puede recortarse**, porque es el único camino normal de alta de asesores. Si se recortara, la consola quedaría como única vía y el operador tendría que dar de alta a mano a los asesores de cada negocio.

**2. Doble factor del operador: del proveedor de identidad.**

`RF-PLT-ACC-02` y `RF-PLT-ACC-03` se cumplen con el TOTP que ofrece Supabase Auth, no con un flujo propio de generación de secretos y códigos QR. La vista P02 sigue existiendo con su diseño; lo que cambia es que detrás llama al proveedor. La verificación de que Supabase Auth soporta TOTP para este caso, y cómo se combina con el rol de plataforma, corresponde a **D-08** y debe confirmarse contra la documentación oficial antes de construir.

Si en D-08 resultara que el proveedor no cubre el caso, esta decisión se revisa: la alternativa es construirlo como está escrito en §6.3, no quitarlo.

**3. La auditoría se escribe siempre, desde la primera acción.**

`RF-PLT-AUD-01`, `-02` y `-05` son P0 y no dependen de que exista una vista. Toda acción de operador se registra desde el primer día, incluso las que ocurran antes de que P25 esté construida. Esto importa porque un registro de auditoría con huecos no sirve: no se puede reconstruir hacia atrás.

**4. La consola no ve el contenido de las conversaciones.**

`RF-PLT-USO-04` aplica a todas las vistas sin excepción. Los indicadores de uso de `RF-PLT-USO-01` son cantidades y fechas; ni la salud del webhook (`RF-PLT-WA-05`) ni la auditoría (`RF-PLT-AUD-02`) pueden incluir texto de mensajes, nombres ni teléfonos de contactos.

**5. Consola y CRM en la misma aplicación, área separada.**

Se confirma la decisión 1 de `docs/03_consola_plataforma_v1.1.md` §2: ruta `/plataforma`, menú propio e indicador visual permanente. La forma concreta —si son una aplicación o dos despliegues— se resuelve en **D-09**; esta decisión fija el requisito de que la consola sea un área con su propio marco visible, no la forma de desplegarla.

## Consecuencias

### Más fácil

- El soporte del piloto tiene todas las herramientas desde el primer día: diagnosticar "no me llegan los mensajes" (`F-PLT-4`), desatascar una invitación (`F-PLT-2`), seguir un piloto (`F-PLT-5`).
- No hay una segunda ronda de diseño ni de discusión de alcance sobre la consola.
- Incorporar clientes después del piloto no requiere trabajo nuevo de consola: el alta ya está completa.

### Más difícil

- Es el compromiso de calendario más grande fuera del CRM. Con 44 requisitos de consola más 63 del CRM en siete semanas y dos personas a tiempo parcial, **el cronograma queda ajustado y hay que vigilarlo en la revisión del 16 de noviembre**.
- Hay que construir interfaz que no tiene usuario real al volumen del piloto (buscador, filtros, ordenamiento sobre dos o tres organizaciones).

### Qué hay que construir

- Entrada de operador con doble factor y rol de plataforma verificado en el servidor (P01, P02, `RF-PLT-SEG-01`).
- Marco de la consola con indicador permanente (L02).
- Lista de organizaciones con búsqueda, filtros y ordenamiento (P20).
- Asistente de alta de tres pasos, con creación e invitación del primer administrador (P21).
- Detalle de organización: resumen con preparación y uso, usuarios, WhatsApp, actividad y configuración (P22 a P26).
- Suspensión y reactivación con confirmación escrita (P27, `RF-PLT-SEG-04`).
- Registro de auditoría de toda acción de operador, inmutable desde la interfaz (`RF-PLT-AUD-01`, `-02`, `-05`).
- Correo propio con entrega confiable para invitaciones y restablecimientos (`RF-PLT-SEG-05`). **Esto no está resuelto:** el servidor de correo por defecto de Supabase no sirve para producción (`CLAUDE.md` §9). Se decide en **D-11**.

### Qué documentos hay que actualizar

- `docs/03_consola_plataforma_v1.1.md` §5.4: anotar que P23 es vista de soporte y que la vía normal de alta de asesores es el CRM (`RF-USR-*`). No cambia ninguna prioridad.
- `docs/03_consola_plataforma_v1.1.md` §2, decisión 8: anotar que el doble factor se resuelve con el proveedor de identidad, pendiente de confirmar en D-08.
- `docs/01_definicion_mvp_v2.2.md` §12 (cronograma): registrar que la consola se construye completa y revisar el reparto de las siete semanas entre CRM y consola.

### Dependencias que esta decisión deja abiertas

| Decisión | Qué tiene que resolver para esta consola |
| --- | --- |
| D-08 | Doble factor del operador y rol de plataforma fuera de las organizaciones |
| D-09 | Si consola y CRM son una aplicación o dos; cómo se separa el área `/plataforma` |
| D-11 | Correo propio con entrega confiable (`RF-PLT-SEG-05`) y cifrado de tokens (`RF-PLT-SEG-02`) |

## Qué revisar cuando el sistema crezca

- **En la revisión de avance del 16 de noviembre de 2026:** si la consola completa está consumiendo tiempo que necesita el CRM, el corte de emergencia es, en este orden: P25 (actividad por organización, el registro se sigue escribiendo), luego los indicadores de uso de `RF-PLT-USO-01`, luego el buscador y los filtros de P20 (`RF-PLT-ORG-02`, `-03`, `-04`). Ninguno de los tres compromete un no-negociable de `CLAUDE.md` §8.
- **Al pasar de 20 organizaciones:** P10 y P30 dejan de ser P1. Con esa cantidad, la lista de organizaciones sin panel general ya no basta para ver el estado de la plataforma.
- **Si entra un tercer operador:** la decisión 9 de §2 (operadores creados por fuera de la interfaz) deja de valer y hay que construir gestión de operadores.
- **Al obtener el rol de Tech Provider:** `RF-PLT-WA-08` sube a P0 (ADR-001) y P24 tiene que soportar el registro integrado de Meta.
- **Si un cliente pide administrar varias sedes como organizaciones separadas:** el modelo actual de una organización por negocio tendría que revisarse.
