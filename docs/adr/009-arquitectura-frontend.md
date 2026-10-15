# ADR-009: Arquitectura del frontend

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-TRV-03, RF-TRV-04, RF-TRV-05, RF-TRV-06, RF-AUT-06, RF-CNV-15, RF-PLT-ACC-04, RF-PLT-ACC-05, RF-PLT-SEG-01
- **Depende de:** ADR-002, ADR-003, ADR-007, ADR-008
- **Condiciona a:** D-11 (despliegue)

## Contexto

Hay que decidir con qué se construye la interfaz, dónde se despliega, si el CRM y la consola son una aplicación o dos, y cómo se manejan sesión, datos, estado y tiempo real.

El brief propone **Next.js en Cloudflare Workers mediante OpenNext** (`docs/01_definicion_mvp_v2.2.md` §8.1). Como ocurrió con Fastify en D-04, ese dato hay que reverificarlo antes de construir sobre él.

**Lo que hay que cubrir:** 18 vistas del CRM (`docs/02_documento_maestro_v1.1.md` §6) y 13 de la consola (`docs/03_consola_plataforma_v1.1.md` §6), con ADR-002 confirmando la consola completa. Son **31 vistas en siete semanas, con dos personas**.

### Restricciones que aplican

- Aplicación web para computador; **sin versión móvil** en el MVP (`CLAUDE.md` §8).
- Español de Colombia, zona horaria de Bogotá (`RF-TRV-05`).
- Toda vista maneja carga, vacío y error (`RF-TRV-04`).
- Accesibilidad básica: teclado, foco visible, contraste (`RF-TRV-06`).
- Un mensaje entrante aparece en la bandeja en 5 segundos o menos; la búsqueda responde en menos de 1 segundo con 5.000 contactos (`RF-TRV-03`).
- Anthony tiene asignada la interfaz (`docs/01_definicion_mvp_v2.2.md` §9).
- Hay una reunión de arranque del design system prevista (§10).

### Lo que ya está decidido y condiciona esta decisión

Tres decisiones anteriores reducen mucho el espacio de opciones, y conviene verlas juntas:

1. **ADR-007: el navegador nunca lee la base de datos.** Toda autorización vive en el backend, que verifica usuario, membresía y estado en cada petición. El frontend solo consume la API.
2. **ADR-008: el tiempo real notifica, la API entrega.** El navegador escucha un canal y recarga desde la API.
3. **ADR-002: la consola es un área con su propio marco visible** (`/plataforma`), con indicador permanente (`RF-PLT-ACC-05`).

El punto 1 es el que más pesa, y conviene decirlo sin rodeos: **la razón principal para usar Next.js desaparece.** Next.js brilla cuando el servidor renderiza páginas con acceso directo a datos —componentes de servidor, acciones de servidor, obtención de datos en el servidor—. Aquí el servidor de la interfaz no tiene acceso a datos: todo pasa por la API, que ya existe y está en otro Worker.

Dicho de otro modo: **el frontend de Numi CRM es una aplicación de sesión, no un sitio de contenido.** Todas sus pantallas están detrás de autenticación, ninguna necesita indexarse, y el primer render con datos no es posible sin sesión. Es el perfil para el que una aplicación de página única es la forma natural.

### Datos verificados

Verificados el 2026-10-15 contra la documentación oficial.

**Next.js en Workers** — [Next.js · Cloudflare Workers](https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/), [OpenNext adapter](https://developers.cloudflare.com/workers/framework-guides/web-apps/opennext/)

| Dato | Valor |
| --- | --- |
| Recomendación actual de Cloudflare | **vinext**: "Cloudflare recommends vinext as the default way to run Next.js applications on Cloudflare Workers" |
| Estado de vinext | **"vinext is in beta"** |
| Rol actual de OpenNext | Vía para mantener una aplicación existente: "You maintain an existing OpenNext application that cannot yet migrate to vinext because of a compatibility gap" |
| Límite de tamaño del Worker | 3 MiB en plan gratuito, 10 MiB en plan de pago (las fuentes difieren: un blog menciona 15 MiB) |
| Huecos conocidos de OpenNext | El middleware de Node de Next.js 15.2 no está soportado; el runtime edge tampoco |
| Cola de revalidación | "not fully suitable for production deployments, you can use it at your own risk!" |

**Aplicación de página única en Workers** — [Single Page Application](https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/), [React + Vite](https://developers.cloudflare.com/workers/framework-guides/web-apps/react/), [Vite plugin](https://developers.cloudflare.com/workers/vite-plugin/)

| Dato | Valor |
| --- | --- |
| Soporte | Documentado y directo, con complemento oficial de Vite |
| Configuración | `assets.not_found_handling` en `"single-page-application"`: toda ruta no encontrada sirve `index.html` |
| Rutas de API en el mismo Worker | Con `run_worker_first` sobre patrones de ruta |
| Andamiaje | `npm create cloudflare@latest -- --framework=react` |
| Obsoleto | Workers Sites, desde Wrangler v4; no usarlo |

### El hallazgo que define la decisión

**Cloudflare cambió su recomendación para Next.js, y ninguna de las dos opciones está madura para este caso.**

- **OpenNext** —lo que propone el brief— pasó a ser la vía para *mantener* aplicaciones existentes, no para empezar. Tiene huecos conocidos y su cola de revalidación está explícitamente desaconsejada para producción.
- **vinext** —lo que Cloudflare recomienda hoy— está **en beta**.

Es el mismo patrón que D-04 con Fastify, y conviene aprender la lección: **construir el MVP sobre una capa de adaptación en movimiento es un riesgo que este equipo no puede absorber.** Dos personas, siete semanas, 31 vistas. Un problema del adaptador en la semana cuatro no tiene plan B.

Y el punto decisivo: **no hace falta correr ese riesgo, porque Next.js ya no aporta lo que lo justificaría** (ADR-007). Se pagaría la complejidad de un framework de renderizado en servidor para no usar el renderizado en servidor.

## Opciones consideradas

### Opción A: Next.js con OpenNext en Workers

Lo que dice el brief.

- **Ventajas:** es lo acordado en §8.1; Next.js es conocido y tiene mucho material; enrutamiento por archivos y convenciones resueltas.
- **Desventajas:** Cloudflare ya no lo recomienda para proyectos nuevos; huecos conocidos (middleware de Node, runtime edge); la cola de revalidación está desaconsejada para producción; se paga la complejidad del renderizado en servidor sin poder usarlo, porque el servidor del frontend no tiene acceso a datos (ADR-007); el tamaño del Worker tiene un límite que una aplicación Next.js con 31 vistas puede acercarse a rozar; depende de una capa que ingeniería inversa la salida de compilación de Next.js, con cambios impredecibles entre versiones.
- **Esfuerzo, costo y riesgo:** esfuerzo medio, **riesgo alto** de calendario por depender de una capa de adaptación en transición.
- **Veredicto:** es la propuesta del brief, y los datos verificados la desaconsejan igual que ocurrió con Fastify.

### Opción B: Next.js con vinext en Workers

Lo que Cloudflare recomienda hoy.

- **Ventajas:** es la vía recomendada por el proveedor, y la que recibirá atención de aquí en adelante; mejor arquitectura que OpenNext según Cloudflare.
- **Desventajas:** **está en beta**; apostar un MVP con fecha dura a software en beta de un proveedor es exactamente el riesgo que ADR-001 ya identificó con Meta, y dos apuestas de ese tipo son una de más; la razón para usar Next.js sigue sin existir (ADR-007).
- **Esfuerzo, costo y riesgo:** riesgo alto por madurez.
- **Veredicto:** probablemente sea la respuesta correcta en 2027. No para noviembre de 2026.

### Opción C: Aplicación de página única con React y Vite, servida desde Workers

React con Vite, compilada a archivos estáticos y servida por Workers, consumiendo la API por HTTP.

- **Ventajas:** encaja exactamente con lo que ADR-007 y ADR-008 dejaron: el frontend solo consume API y escucha un canal; sin capa de adaptación entre el framework y el runtime —se sirven archivos estáticos, que es lo que Workers hace mejor; camino documentado y maduro, con complemento oficial de Vite; compilación rápida, que con dos personas y siete semanas se traduce en iteración; un solo modelo mental —todo corre en el navegador— en lugar de la distinción entre componentes de servidor y de cliente, que es la mayor fuente de confusión en Next.js moderno; menos superficie que aprender; `RF-TRV-03` no se ve afectado, porque el tiempo lo domina la API, no el primer render.
- **Desventajas:** sin renderizado en servidor —irrelevante aquí, porque todas las vistas están detrás de sesión y ninguna necesita indexarse; el primer render muestra un marco vacío mientras carga, lo que se resuelve con estados de carga que `RF-TRV-04` exige de todos modos; hay que elegir enrutador y biblioteca de datos en vez de recibirlos impuestos; el paquete inicial crece con la aplicación y hay que vigilarlo.
- **Esfuerzo, costo y riesgo:** esfuerzo medio-bajo. Riesgo bajo.

### Opción D: Aplicación de página única en Cloudflare Pages

Igual que C, pero en Pages en lugar de Workers.

- **Ventajas:** producto pensado para sitios estáticos.
- **Desventajas:** Cloudflare está consolidando Pages dentro de Workers, y las guías de framework apuntan a Workers; usar dos productos distintos complica el despliegue sin ganar nada.
- **Veredicto:** sin ventaja sobre C.

## Decisión

**Opción C: aplicación de página única con React y Vite, servida desde Cloudflare Workers.**

Razón, en orden de peso:

1. **Next.js resolvería un problema que este sistema no tiene.** ADR-007 dejó al frontend como consumidor puro de una API. El renderizado en servidor —lo único que justifica la complejidad de Next.js— no se puede usar, porque el servidor del frontend no tiene acceso a datos.
2. **Las dos vías de Next.js en Workers son inmaduras**, una en mantenimiento y la otra en beta. El equipo no puede absorber un problema de adaptador en la semana cuatro.
3. **Es la forma natural de una aplicación de sesión.** 31 vistas, todas detrás de autenticación, ninguna indexable.
4. **Menos piezas que aprender es tiempo de desarrollo.** Con siete semanas, dos personas y 31 vistas, la simplicidad del modelo mental es una ventaja medible, no estética.

### Una aplicación, dos áreas

**El CRM y la consola son una sola aplicación**, confirmando la decisión 1 de `docs/03_consola_plataforma_v1.1.md` §2: un solo código, un solo despliegue.

- Rutas del CRM en la raíz (`/bandeja`, `/contactos`, `/metricas`, `/usuarios`, `/configuracion`).
- Rutas de la consola bajo `/plataforma`.
- **Marco visual distinto y permanente** para la consola (`RF-PLT-ACC-05`, L02): barra lateral con tratamiento propio y etiqueta "Consola de plataforma · Numi", para que nunca se confunda con el CRM de un cliente.
- **Las rutas de `/plataforma` se cargan de forma diferida.** Un asesor nunca descarga el código de la consola. Esto importa porque el paquete inicial de una aplicación de página única es su punto débil, y 31 vistas en un solo paquete sería un problema.

Hay un punto de seguridad que conviene dejar explícito, porque es un error frecuente: **que el código de la consola llegue al navegador no otorga ningún permiso.** La protección real es `RF-PLT-SEG-01` —el servidor verifica el rol de plataforma en cada petición (ADR-007)—. La carga diferida es rendimiento, no seguridad. Un asesor que fuerce la ruta `/plataforma` verá el marco y recibirá errores de autorización en cada llamada.

### Sesión

- **El cliente de Supabase Auth en el navegador se usa solo para autenticar** —iniciar sesión, cerrar sesión, refrescar el token, el doble factor TOTP del operador (`RF-PLT-ACC-02`, `RF-PLT-ACC-03`)—. **Nunca para consultar datos.**
- El token de acceso acompaña cada llamada a la API.
- **La interfaz no decide permisos; los refleja.** Oculta lo que el usuario no puede hacer, pero la autoridad está en el backend (ADR-007).
- `RF-AUT-06`: si la API responde que la sesión expiró, la aplicación guarda la ruta actual, lleva a `V01` y devuelve al usuario a donde estaba tras entrar.
- `RF-AUT-05` y `RF-AUT-07`: usuario desactivado u organización suspendida llevan a `V05` con el mensaje correspondiente. **Esto puede ocurrir a mitad de sesión**, porque ADR-007 lo verifica en cada petición; la interfaz debe manejar esa respuesta en cualquier llamada, no solo al entrar.

### Obtención de datos y estado

- **Una biblioteca de datos de servidor** (por ejemplo TanStack Query) para obtención, caché, reintentos y revalidación. No construir esto a mano: es el núcleo de lo que hace la aplicación y tiene muchos casos difíciles ya resueltos.
- **El estado de servidor y el de interfaz se separan.** Casi todo el estado de este producto es estado de servidor —conversaciones, mensajes, contactos, usuarios—. El estado propio de la interfaz —filtros, paneles abiertos, borradores— es poco y local.
- **No hace falta un gestor de estado global.** `CLAUDE.md` §4, regla 9: toda pieza debe justificar qué problema resuelve hoy. Con la biblioteca de datos encargándose del estado de servidor, un gestor global no tiene trabajo.
- **La búsqueda de contactos es del servidor, no del cliente** (`RF-TRV-03`: menos de 1 segundo con 5.000 contactos). No se descargan 5.000 contactos para filtrar en el navegador.

### Integración con el tiempo real

Siguiendo ADR-008:

1. El navegador se suscribe al canal privado de su organización con el cliente de Supabase.
2. Al recibir un aviso, **invalida las consultas afectadas**; la biblioteca de datos recarga desde la API.
3. **No se escriben datos en la caché desde el aviso.** El aviso no trae contenido (ADR-008) y escribir desde él abriría un camino de datos que no pasa por la API.
4. Red de seguridad: revalidación cada 60 segundos, al recuperar el foco de la ventana y al reconectar el canal.
5. El contador de conversaciones sin responder (`RF-CNV-15`) se **recalcula** desde la API, nunca se incrementa a ciegas.

Este diseño tiene una propiedad que vale la pena notar: **el tiempo real y la red de seguridad usan el mismo camino** —invalidar y recargar—. No hay dos rutas por las que los datos lleguen a la pantalla, así que no hay forma de que se contradigan.

### Requisitos transversales

| Requisito | Cómo se cumple |
| --- | --- |
| `RF-TRV-04` | Estados de carga, vacío y error como componentes compartidos del design system; ninguna vista los implementa por su cuenta |
| `RF-TRV-05` | Formateo centralizado en una utilidad: fechas en zona de Bogotá, textos en español de Colombia. Nunca formateo suelto en las vistas |
| `RF-TRV-06` | Componentes accesibles desde la base —foco visible, navegación por teclado, contraste— porque agregarlo después cuesta mucho más |
| `RF-TRV-03` | La percepción de velocidad depende de la API y del tiempo real, no del primer render |
| `V60` | Estados del sistema —no encontrado, sin permiso, error, sesión expirada— como rutas y como estados dentro de una vista |

### Design system

La reunión de arranque del design system está prevista en §10 de la definición del MVP y su alcance está definido en §11 del documento maestro. Dos precisiones que esta decisión aporta:

- **Componentes compartidos entre el CRM y la consola.** Las tablas, los formularios, los estados y las ventanas son los mismos; lo que cambia es el marco visual del área. Una sola aplicación hace esto natural.
- **Conviene una base de componentes ya resuelta** en vez de construir desde cero. Con 31 vistas y siete semanas, el tiempo está en las vistas, no en reimplementar un selector accesible. La elección concreta corresponde a la reunión del design system, pero el criterio es ese.

### Punto abierto: la elección de React no está justificada en este ADR

Las opciones A a D comparan **arquitecturas** (aplicación de página única frente a Next.js, Workers frente a Pages), no **frameworks**. React se da por sentado sin compararlo con Vue, Svelte o Solid, que son alternativas legítimas para este perfil de producto.

El criterio que debería decidirlo, dado el calendario, no es el rendimiento ni la elegancia del framework sino **cuántos componentes se pueden no escribir**: con 31 vistas, siete semanas y `RF-TRV-06` exigiendo accesibilidad real, lo que pesa es la oferta de bases de componentes accesibles ya resueltas —tablas con orden y paginación, selectores, ventanas modales con foco atrapado—. Por ahí React tiene ventaja, y Svelte la desventaja de que el código más corto por vista se pierde si hay que construir esos componentes.

Hay además un dato del proyecto que decide por encima de todo lo anterior: **qué conoce ya Anthony**, que tiene asignada la interfaz (`docs/01_definicion_mvp_v2.2.md` §9). Aprender un framework nuevo en noviembre no se compensa con ninguna ventaja técnica de esta lista.

**Este punto queda abierto y debe resolverse antes de empezar la interfaz.** Todo lo demás de este ADR —aplicación de página única, dos Workers, una aplicación con dos áreas, sesión, obtención de datos, tiempo real— **no depende del framework elegido** y se mantiene igual con Vue o Svelte.

### Qué fija esta decisión para el diseño

1. **El frontend no tiene acceso a la base de datos.** Ni directo, ni por el cliente de Supabase, ni por el canal de tiempo real.
2. **Dos Workers separados:** uno sirve la aplicación, otro es la API. Se despliegan por separado y fallan por separado.
3. **La API se consume por HTTP desde el navegador**, con el token en cada llamada.
4. **Carga diferida por área** —CRM y consola— y por vistas pesadas, como métricas.
5. **Sin renderizado en servidor ni generación estática de vistas con datos.** Todas las vistas están detrás de sesión.
6. **Los secretos nunca llegan al navegador.** La clave de publicación de Broadcast (ADR-008) y los tokens de WhatsApp (`RF-PLT-SEG-02`) viven solo en el servidor. El navegador solo lleva la URL de la API y la clave pública de Supabase Auth.

## Consecuencias

### Más fácil

- Sin capa de adaptación entre framework y runtime: lo que se compila es lo que se sirve.
- Un solo modelo mental, sin frontera entre servidor y cliente dentro del frontend.
- Compilación rápida, que con siete semanas se traduce en más iteraciones.
- Componentes compartidos entre CRM y consola sin esfuerzo extra.
- Despliegue simple: archivos estáticos más configuración de enrutamiento.

### Más difícil

- Hay que elegir enrutador y biblioteca de datos, en vez de recibirlos impuestos.
- **El tamaño del paquete hay que vigilarlo.** La carga diferida por área lo controla, pero es disciplina continua.
- El primer render muestra el marco vacío mientras carga. Se resuelve con los estados que `RF-TRV-04` exige igual.
- Se abandona lo que dice §8.1 de la definición del MVP, que hay que actualizar.
- Sin indexación en buscadores —irrelevante para un producto detrás de sesión, pero conviene decirlo por si alguna vez hay página pública de marketing, que sería otro proyecto.

### Qué hay que construir

- Proyecto de React con Vite y el complemento de Cloudflare, con `not_found_handling` en modo de página única.
- Enrutamiento con áreas separadas y carga diferida de `/plataforma`.
- Capa de cliente de la API con el token, manejo uniforme de errores y tratamiento de sesión expirada, usuario desactivado y organización suspendida.
- Integración del canal de tiempo real con invalidación de consultas.
- Utilidades de formato de fecha y número para Bogotá y español de Colombia.
- Componentes compartidos de carga, vacío y error.
- Marco de la aplicación (L01) y marco de la consola (L02) con su indicador permanente.
- Las 31 vistas.

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.1: la fila "Frontend" pasa de *"Next.js en Cloudflare Workers mediante OpenNext"* a *"Aplicación de página única con React y Vite, servida desde Cloudflare Workers"*.
- `docs/01_definicion_mvp_v2.2.md` §13, fila "Complejidad del stack": el riesgo baja, porque se quita una capa de adaptación.
- `CLAUDE.md` §9: la fila "Frontend" de la tabla de stack de partida.

### Impacto en el cronograma

**31 vistas en siete semanas con dos personas sigue siendo el riesgo central del proyecto**, y esta decisión lo reduce pero no lo elimina. ADR-002 ya definió el corte de emergencia para la consola y ADR-004 el de multimedia. Esta decisión agrega un argumento: **quitar la capa de adaptación elimina una clase entera de problemas de calendario** —los que no se pueden estimar porque dependen de un adaptador ajeno.

## Qué revisar cuando el sistema crezca

- **Si el paquete inicial se vuelve lento:** medir antes de actuar. Lo más probable es que sea una biblioteca pesada en una vista concreta, resoluble con carga diferida.
- **Si hace falta una página pública** (marketing, documentación, estado del servicio): es otro proyecto, con otras necesidades. No conviene meterla en esta aplicación.
- **Si se necesita versión móvil:** hoy está fuera del MVP (`CLAUDE.md` §8). Una aplicación de página única con React no cierra ninguna puerta.
- **Si vinext sale de beta y se estabiliza:** no es razón para migrar. Esta decisión no se tomó contra Next.js, sino a favor de no tener una capa de adaptación que no aporta nada a este diseño.
- **Si el equipo crece:** las convenciones impuestas por un framework pasan a valer más. Vale la pena revisar entonces si la libertad de una aplicación de página única sigue siendo ventaja.
- **Si la bandeja se vuelve lenta con muchas conversaciones:** el problema estará en la cantidad de elementos en pantalla, no en la arquitectura. La solución es virtualizar la lista, no cambiar de framework.
