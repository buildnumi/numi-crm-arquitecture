# ADR-007: Identidad, roles y sesiones

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-AUT-01 a RF-AUT-07, RF-USR-01 a RF-USR-08, RF-PLT-ACC-01 a RF-PLT-ACC-05, RF-PLT-SEG-01, RF-PLT-SEG-03, RF-PLT-USR-01 a RF-PLT-USR-07, RF-TRV-08
- **Depende de:** ADR-002, ADR-003, ADR-006
- **Condiciona a:** D-07 (tiempo real), D-09 (frontend), D-11 (correo)

## Contexto

Hay que decidir cómo se autentican y autorizan **tres clases de personas** con necesidades distintas:

1. **Usuarios de un negocio** (asesor, administrador): correo y contraseña, pertenecen a una organización (`RF-AUT-01` a `RF-AUT-07`).
2. **Operadores de plataforma** (Franco y Anthony): acceden a la consola, con doble factor obligatorio, y **no pertenecen a ninguna organización** (`RF-PLT-ACC-01` a `RF-PLT-ACC-05`).
3. **El consumidor de la cola** (ADR-005): no es una persona y no tiene sesión.

ADR-002 ya decidió que el doble factor del operador se resuelve con el proveedor de identidad, **pendiente de confirmar aquí**. ADR-006 fijó tres roles de base de datos y dejó explícito que *"el rol de plataforma no es una membresía de organización"*, dejando a esta decisión cómo representarlo.

### Restricciones que aplican

- Dos operadores conocidos y fijos; la consola decide no construir gestión de operadores (`docs/03_consola_plataforma_v1.1.md` §2, decisión 9).
- Numi nunca conoce ni transmite contraseñas de los clientes: el primer acceso es una invitación (decisión 3 de §2).
- Sesión del operador: **12 horas, o 30 minutos sin actividad** (`RF-PLT-SEG-03`).
- Invitaciones: vencen a los 7 días, un solo uso (`RF-USR-08`, `RF-PLT-USR-07`).
- Las acciones de la consola se ejecutan en el servidor, que verifica el rol en **cada petición**; el navegador nunca recibe permisos para leer datos de todas las organizaciones (`RF-PLT-SEG-01`).
- Runtime: Hono en Cloudflare Workers (ADR-003). Acceso a Postgres con roles de aplicación propios, ninguno con `bypassrls` (ADR-006).

### Datos verificados de Supabase

Verificados el 2026-10-15 contra la documentación oficial.

| Dato | Valor | Fuente |
| --- | --- | --- |
| Doble factor TOTP | Soportado. "TOTP MFA API is **free to use** and is **enabled on all Supabase projects by default**" | [Auth MFA · TOTP](https://supabase.com/docs/guides/auth/auth-mfa/totp) |
| Alta del factor | Tres pasos: `enroll()` devuelve código QR y clave, `challenge()` devuelve un identificador, `verify()` confirma con el código | mismo |
| Vigencia del código | 30 segundos, con tolerancia de un intervalo de desfase de reloj | mismo |
| Nivel de garantía | `getAuthenticatorAssuranceLevel()` devuelve nivel actual y siguiente (`aal1`, `aal2`) | mismo |
| Sesión acotada en el tiempo | Configurable: `auth.sessions.timebox` ("Force sign out after the specified duration") | [User sessions](https://supabase.com/docs/guides/auth/sessions) |
| Sesión por inactividad | Configurable: `auth.sessions.inactivity_timeout` | mismo |
| Vigencia del token de acceso | `auth.jwt_expiry`, por defecto 3600 s; se desaconseja bajar de 5 minutos | mismo |
| Aplicación de los límites | **Perezosa:** "Sessions are not proactively destroyed... the check is enforced whenever a session is refreshed next". La duración efectiva es el tiempo configurado **más** la vigencia del token | mismo |
| Cerrar sesión | **No invalida el token de acceso:** "the access token JWT will still be valid until it expires" | mismo |
| Rotación de tokens de refresco | Activa por defecto; un token de refresco se canjea una sola vez | mismo |
| Verificación del token desde el backend | Por JWKS en `/auth/v1/.well-known/jwks.json`, comparando `kid` y `alg`. "To verify the signature avoid implementing the algorithms yourself" | [JWTs](https://supabase.com/docs/guides/auth/jwts) |
| Claims personalizados | "You can use a Custom Access Token Hook to add, remove or change claims present in the token" | mismo |

### Los tres hechos que condicionan el diseño

**Primero: el doble factor está resuelto y no cuesta nada.** TOTP viene activo por defecto, gratis, en todos los proyectos. La decisión de ADR-002 queda confirmada: `RF-PLT-ACC-02` y `RF-PLT-ACC-03` se cumplen llamando al proveedor, sin construir generación de secretos ni códigos QR.

**Segundo: cerrar sesión no invalida el token.** Esto choca de frente con dos requisitos:

- `RF-AUT-07`: una organización suspendida cierra *"las sesiones abiertas en la siguiente acción"*.
- `RF-PLT-USR-06` y `RF-USR-05`: desactivar a un usuario le quita el acceso.

Si la autorización se apoyara solo en el token, un usuario desactivado seguiría entrando hasta una hora. **La consecuencia es directa: el estado de la organización y del usuario se verifica en el servidor en cada petición, contra la base de datos, no contra el token.**

**Tercero: los límites de sesión se aplican de forma perezosa.** La duración real es el tiempo configurado más la vigencia del token. Para cumplir `RF-PLT-SEG-03` —30 minutos de inactividad— con un token de una hora, el corte efectivo sería de hasta 90 minutos. **También esto obliga a verificar en el servidor.**

Los tres apuntan a lo mismo, y conviene decirlo de una vez: **el token sirve para saber quién es el usuario; nunca para saber qué puede hacer.** Esa separación es lo que hace que el resto encaje.

## Opciones consideradas

### Opción A: Supabase Auth con RLS directo desde el navegador

El frontend usa el cliente de Supabase y consulta la base directamente; RLS resuelve el aislamiento con los claims del token.

- **Ventajas:** menos código de backend; es el camino que Supabase documenta y optimiza; habilita de forma natural el tiempo real de Supabase (D-07).
- **Desventajas:** **contradice `RF-PLT-SEG-01`**, que exige que las acciones se ejecuten en el servidor y que el navegador nunca reciba permisos amplios; el navegador tendría credenciales de base de datos, aunque acotadas; la suspensión y la desactivación quedarían en manos del token, que no se puede revocar (segundo hecho); el operador de plataforma necesitaría leer todas las organizaciones desde el navegador, que es justo lo que la consola prohíbe; no hay lugar donde esconder los tokens de WhatsApp (`RF-PLT-SEG-02`).
- **Esfuerzo, costo y riesgo:** esfuerzo bajo, **riesgo alto**. Choca con requisitos que no se negocian.
- **Veredicto:** descartada. No es compatible con `RF-PLT-SEG-01`.

### Opción B: Supabase Auth solo como identidad; toda autorización en el backend

Supabase Auth autentica y emite el token. **Todo el acceso a datos pasa por la API**, que verifica el token, resuelve la membresía y el estado contra la base, y usa los roles de ADR-006.

- **Ventajas:** cumple `RF-PLT-SEG-01` literalmente; la suspensión y la desactivación tienen efecto inmediato, porque se verifican contra la base y no contra el token; el doble factor, las invitaciones, el restablecimiento de contraseña y la rotación de tokens vienen hechos; los tres roles de ADR-006 encajan sin forzar nada; los tokens de WhatsApp nunca salen del servidor.
- **Desventajas:** cada petición consulta la base para resolver membresía y estado, lo que agrega una consulta —barata e indexada, pero existe; hay que escribir la verificación del token con JWKS; el tiempo real de Supabase necesita tratamiento aparte, porque va directo del navegador a la base (D-07).
- **Esfuerzo, costo y riesgo:** esfuerzo medio. Riesgo bajo.

### Opción C: Identidad propia

Construir autenticación, contraseñas, invitaciones, restablecimiento y doble factor.

- **Ventajas:** control total; sin dependencia de proveedor.
- **Desventajas:** es trabajo de semanas que no distingue al producto, en un calendario que ADR-002 y ADR-004 ya dejaron ajustado; el almacenamiento de contraseñas y el doble factor son terreno donde los errores propios son caros; habría que construir también la rotación de tokens y la gestión de sesiones.
- **Esfuerzo, costo y riesgo:** esfuerzo alto, riesgo alto, beneficio nulo.
- **Veredicto:** descartada.

## Decisión

**Opción B: Supabase Auth como proveedor de identidad; toda la autorización en el backend.**

Dicho en una frase: **Supabase Auth responde "quién eres"; la API responde "qué puedes hacer", y lo consulta en la base en cada petición.**

### Qué resuelve Supabase Auth

| Requisito | Cómo |
| --- | --- |
| `RF-AUT-01`, `RF-PLT-ACC-01` | Inicio de sesión con correo y contraseña |
| `RF-AUT-03`, `RF-USR-08` | Invitación por correo con enlace; vencimiento a 7 días y un solo uso se configuran en el proveedor |
| `RF-AUT-04` | Restablecimiento de contraseña por correo |
| `RF-PLT-ACC-02`, `RF-PLT-ACC-03` | TOTP, activo por defecto y gratis; el primer ingreso usa `enroll()` con código QR |
| `RF-PLT-SEG-03` | `timebox` de 12 horas e `inactivity_timeout` de 30 minutos, **más** la verificación en el servidor descrita abajo |

**Los correos no se envían con el servidor por defecto de Supabase**, que no sirve para producción (`CLAUDE.md` §9). `RF-TRV-08` y `RF-PLT-SEG-05` exigen remitente propio con entrega confiable. Se configura un servicio de correo externo; cuál, es D-11. **Sin esto no hay invitación, y sin invitación no hay alta de negocio** (`RF-PLT-ORG-06`): es dependencia de la ruta crítica, no un detalle de configuración.

### Qué resuelve el backend, en cada petición

Este es el núcleo de la decisión. En **toda** petición autenticada, antes de cualquier trabajo:

1. **Verificar el token** por JWKS, comparando `kid` y `alg`, con una biblioteca establecida. No implementar el algoritmo.
2. **Resolver el sujeto** (`sub`) a un usuario.
3. **Consultar en la base**, en una sola consulta: el estado del usuario, su membresía, su rol y el estado de su organización.
4. **Rechazar** si el usuario está desactivado (`RF-AUT-05`), si la organización está suspendida (`RF-AUT-07`), o si no hay membresía.
5. **Fijar el contexto de organización** en la transacción, con el rol de base de datos que corresponda (ADR-006).

El punto 3 es lo que hace que la suspensión y la desactivación tengan **efecto inmediato** pese a que el token siga siendo válido. Es la respuesta directa al segundo hecho verificado.

**Esa consulta no se cachea en memoria.** Workers no garantiza estado entre peticiones (ADR-003), y cachearla reintroduciría exactamente el retraso que se quiere evitar.

### Cómo se representa el rol de plataforma

ADR-006 fijó que el operador no es miembro de ninguna organización. Concretamente:

- Los operadores viven en una **tabla propia**, separada de los usuarios de las organizaciones, vinculada al identificador de Supabase Auth.
- **El rol de plataforma no viaja en el token.** Se resuelve consultando esa tabla en cada petición. Un claim personalizado sería más rápido, pero `RF-PLT-SEG-01` exige verificación del rol en cada petición, y un claim en un token de una hora no permite revocar a un operador.
- Las rutas de `/plataforma` usan el rol de base de datos `app_platform` (ADR-006), que **no tiene permiso sobre las tablas de mensajes**. `RF-TRV-07` queda garantizado por permisos, no por disciplina.
- **Los operadores se crean fuera de la interfaz**, como decide `docs/03_consola_plataforma_v1.1.md` §2, decisión 9: alta en Supabase Auth más una fila en la tabla de operadores, por migración.
- **El doble factor es obligatorio para operadores.** Si el nivel de garantía no es `aal2`, toda ruta de `/plataforma` se rechaza, incluidas las de la API. No alcanza con que la pantalla lo pida.
- Un operador puede además ser usuario normal de una organización de demostración (§3 de la consola). Son **dos pertenencias independientes**: su rol de plataforma no le da acceso a datos de ningún negocio, y su membresía de demostración no le da poderes de consola.

### Roles dentro de una organización

- Dos roles: **Asesor** y **Administrador** (`docs/02_documento_maestro_v1.1.md` §3).
- **El rol vive en la membresía**, no en el usuario: una fila por usuario y organización, con su rol y su estado.
- **El rol tampoco viaja en el token**, por la misma razón: `RF-USR-04` permite cambiarlo, y ese cambio debe surtir efecto sin esperar a que expire un token.
- `RF-USR-07` —siempre al menos un administrador activo— se verifica en el servidor al desactivar o cambiar rol, con la fila bloqueada para evitar que dos operaciones simultáneas dejen la organización sin administrador.

### Qué fija esta decisión para el diseño

1. **El token identifica; no autoriza.** Ningún permiso se decide leyendo el token más allá de la identidad y el nivel de garantía.
2. **Una sola función resuelve el contexto de la petición** —usuario, membresía, rol, estado— y entrega el contexto de organización que pide ADR-006. Las rutas no la saltan.
3. **La sesión se verifica en dos capas:** los límites del proveedor (`timebox` e `inactivity_timeout`) y la verificación del servidor en cada petición. La primera sola no alcanza por la aplicación perezosa.
4. **Las invitaciones son de un solo uso y vencen a los 7 días.** Generar una nueva invalida la anterior (`RF-USR-03`, `RF-PLT-USR-04`). **Ningún enlace se guarda para volver a mostrarse** (decisión 3 de §2 de la consola).
5. **Nunca se transmiten contraseñas.** El operador no fija contraseñas de usuarios de un negocio; solo envía invitaciones y restablecimientos (`RF-PLT-USR-05`).
6. **El consumidor de la cola no tiene identidad de usuario.** Usa `app_worker` (ADR-006) y no pasa por esta verificación: su autorización es el hecho de haber recibido un evento con firma válida (ADR-005).
7. **Un usuario puede pertenecer a más de una organización.** El esquema lo permite —la membresía es una tabla— aunque el MVP no construya el cambio entre organizaciones. Esto evita una migración dolorosa después.
8. **El cierre de sesión borra el token del cliente y revoca el token de refresco.** El token de acceso sigue siendo válido hasta expirar, pero la verificación del servidor lo vuelve inofensivo si el usuario fue desactivado.

### Configuración del proveedor

| Opción | Valor | Por qué |
| --- | --- | --- |
| `jwt_expiry` | 3600 s (por defecto) | Se desaconseja bajarlo de 5 minutos; con verificación en el servidor, la vigencia del token deja de ser crítica |
| `sessions.timebox` | 12 h | `RF-PLT-SEG-03` |
| `sessions.inactivity_timeout` | 30 min | `RF-PLT-SEG-03` |
| Rotación de tokens de refresco | Activa (por defecto) | Un token de refresco se canjea una sola vez |
| Correo | Servicio externo con remitente propio | `RF-TRV-08`, `RF-PLT-SEG-05`; el servidor por defecto no sirve para producción |

`RF-PLT-SEG-03` aplica a operadores. Para usuarios de un negocio, `RF-AUT-06` solo pide que una sesión expirada devuelva al usuario a donde estaba. Los mismos valores sirven para ambos, y usar uno solo evita configurar dos proyectos.

## Consecuencias

### Más fácil

- No se construye autenticación, contraseñas, doble factor ni rotación de tokens.
- La suspensión y la desactivación tienen efecto inmediato, que es lo que `RF-AUT-07` pide.
- `RF-PLT-SEG-01` se cumple por construcción: el navegador nunca toca la base.
- `RF-TRV-07` queda garantizado por permisos de base de datos.
- Los tokens de WhatsApp no tienen camino hacia el navegador.

### Más difícil

- Cada petición agrega una consulta de contexto. Es barata e indexada, pero hay que medirla si la bandeja se acerca al objetivo de 5 segundos (`RF-TRV-03`).
- Hay que escribir la verificación de token con JWKS y manejar la caché de claves —el proveedor recomienda esperar al menos 20 minutos al rotar una clave de firma.
- **El tiempo real de Supabase va directo del navegador a la base y no pasa por esta verificación.** D-07 tiene que resolverlo explícitamente; ADR-006 ya anotó que sus políticas son un sistema aparte.
- Dependencia de un proveedor de identidad: si cambia precios o condiciones, afecta a todos los usuarios.
- Dos tablas de personas (usuarios de organización y operadores) con dos caminos de autorización distintos.

### Qué hay que construir

- Verificación de token por JWKS, con caché de claves.
- Función única de contexto de petición: usuario, membresía, rol, estado de usuario y de organización.
- Tabla de operadores de plataforma, vinculada a Supabase Auth, poblada por migración.
- Exigencia de `aal2` en toda ruta de plataforma, en la API y no solo en la interfaz.
- Tabla de membresías con rol y estado, y la verificación de `RF-USR-07` con bloqueo de fila.
- Flujo de invitación: alta en el proveedor, correo con remitente propio, vencimiento de 7 días, un solo uso, invalidación de la anterior.
- Pantallas `V01` a `V05` del CRM y `P01`, `P02` de la consola (D-09 define cómo).

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.1: precisar que Supabase Auth se usa **solo como identidad** y que la autorización vive en el backend.
- `docs/01_definicion_mvp_v2.2.md` §8.2 punto 2: la advertencia sobre tiempo real y políticas por fila sigue abierta y pasa a D-07.
- `docs/03_consola_plataforma_v1.1.md` §2, decisión 8: anotar que el doble factor es TOTP del proveedor, **confirmado disponible y gratuito**.

### Dependencia de ruta crítica

**El correo con remitente propio bloquea el alta de organizaciones.** Sin él no hay invitación (`RF-PLT-ORG-06`), y sin invitación el primer administrador no entra. Se decide en D-11, pero conviene resolverlo antes: es barato y desbloquea el flujo principal de la consola.

## Qué revisar cuando el sistema crezca

- **Si la consulta de contexto pesa:** antes de cachear, revisar índices. Cachear reintroduce el retraso en la suspensión; si aun así hiciera falta, debe ser con vida muy corta (segundos) y con invalidación explícita al suspender o desactivar.
- **Si entra un tercer operador:** la decisión 9 de §2 de la consola deja de valer y hay que construir gestión de operadores. La tabla ya existe; falta la interfaz.
- **Si un usuario necesita pertenecer a varias organizaciones en la práctica:** el esquema lo soporta; falta el selector de organización y su efecto en el contexto.
- **Si se agrega inicio de sesión con proveedores externos:** revisar que el nivel de garantía y la exigencia de `aal2` para operadores sigan aplicando.
- **Al revisar la rotación de claves de firma:** respetar los 20 minutos de espera que recomienda el proveedor, o las verificaciones fallarán mientras la caché se actualiza.
- **Si el doble factor se extiende a administradores de negocio:** hoy es solo para operadores (`RF-PLT-ACC-02`). Extenderlo es configuración más una exigencia de `aal2` en las rutas correspondientes, pero cambia el flujo de alta del usuario y debe decidirse como alcance.
