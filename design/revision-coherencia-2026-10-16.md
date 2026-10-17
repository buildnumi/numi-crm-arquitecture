# Revisión de coherencia: arquitectura vs. pantallas de Stitch

*16 oct 2026 · 37 pantallas en `design/stitch/` contra `02_documento_maestro_v1.1.md`, `03_consola_plataforma_v1.1.md`, `01_definicion_mvp_v2.2.md`, `architecture/arquitectura.md` y ADR 001–015.*

**Veredicto:** la estructura general es coherente. Están las vistas principales del CRM y de la consola, y respetan bien varias reglas difíciles: la ventana de 24 h, el filtro por rol en la bandeja, el token de solo escritura, la confirmación escribiendo el nombre, el último administrador y el enlace de un solo uso. Pero hay **8 contradicciones con la arquitectura**, **11 elementos fuera del alcance del MVP** y **requisitos P0 sin pantalla**, sobre todo la configuración del agente de IA y los estados de mensaje.

Severidad: 🔴 contradice una decisión de arquitectura o de seguridad · 🟠 fuera de alcance o requisito incumplido · 🟡 dato o texto inconsistente.

---

## 1. Contradicciones con la arquitectura (🔴)

| # | Pantalla | Qué muestra | Qué dice la arquitectura | Corrección |
|---|---|---|---|---|
| 1 | 28 Actividad (P25) | Botón **"Acceder al CRM"** en el encabezado de la organización | El operador no es miembro de las organizaciones y **no ve contenido** de conversaciones (03 §2 dec. 2 y 4; ADR-006/007; RF-TRV-07) | Quitarlo. *Ir al CRM* solo existe en el menú del operador y lleva a **su** organización de prueba (RF-PLT-ACC-05) |
| 2 | 18 Organizaciones vacío | "Aislamiento total: **bases de datos independientes**" | Una sola base compartida con RLS + `organization_id` (ADR-006) | Cambiar el texto, o quitar el bloque |
| 3 | 35 Entrada consola (P01) | Casilla **"Mantener sesión iniciada en este terminal"** | Sesión del operador: 12 h máximo o 30 min sin actividad (RF-PLT-SEG-03, ADR-007) | Quitar la casilla |
| 4 | 30 Reactivar (fondo) | "**Mensajes en cola retenida: 47**, listos para entrega inmediata al reconectar" | Durante la suspensión los entrantes **se guardan** y ya están en la bandeja; no hay cola de salida. El envío es síncrono (ADR-012) | Mostrar "Mensajes recibidos durante la suspensión: 47 (ya en la bandeja)" |
| 5 | 29 Suspender (modal) | "Los mensajes … se seguirán guardando **en auditoría**"; "no se podrán enviar mensajes **automáticos ni campañas**" | Se guardan en la **bandeja** del negocio, y se bloquea **todo** envío, también el manual del asesor (RF-CNV-16, RF-PLT-EST-03) | Usar el texto canónico de 03 §9 "Confirmar suspensión" |
| 6 | 37 Panel general (P10) | "Servidores Core Bogotá · CPU 24% · RAM 4.2/16 GB", "+100% SLA", "ping automático a tenants >72 h" | No hay servidores propios: Cloudflare Workers + Supabase. El "ping" automático no está diseñado | Quitar el bloque de infraestructura (P10 solo pide tarjetas y tres listas, 03 §6.4) |
| 7 | 28, 36 Auditoría | "Hash SHA-256 verificado", "cadena de custodia criptográfica", "retención 180 días" | La arquitectura no define encadenamiento por hash ni plazo de retención de la auditoría. Solo exige que no se edite ni se borre desde la interfaz (RF-PLT-AUD-05) | Quitar las promesas, o abrir un ADR si se quieren |
| 8 | 23 Confirmación de alta | "Protocolo de onboarding **cifrado de extremo a extremo** · Cumplimiento Res. 1995/1999 MinSalud" | No hay cifrado de extremo a extremo (Numi lee los mensajes). El cumplimiento legal está **pendiente** (Ley 1581, arquitectura §pendientes legales) | Quitar. Una promesa de cumplimiento falsa es un riesgo legal |

Promesas de marketing que también conviene quitar porque no son ciertas: "Cifrado cuántico activo" (34), "Protección Zero-Trust" y "Bloqueo temporal de IP" (32; hay que confirmar qué ofrece Supabase Auth), "datos bancarios" (31).

## 2. Fuera del alcance del MVP (🟠)

| Pantalla | Elemento | Referencia |
|---|---|---|
| 11 Bandeja 24 h vencida | "**Plantillas aprobadas** por WhatsApp disponibles para reanudar el contacto" | Las plantillas están fuera; una conversación vencida **no se puede retomar** (01 §6, ADR-012) |
| 14 Configuración | **Logotipo** y **color de marca y tema** (6 colores) | No está en RF-CFG; el design system usa una identidad fija, Violeta Autoridad (02 §11.1) |
| 14 Configuración | Especialidad por asesor ("Ortodoncia y Estética", "Implantes & Cirugía") | No existe en el modelo de usuario (02 §9) |
| 13 Mi perfil | Toda la vista: cambiar nombre, **correo** y contraseña | No figura en el inventario (02 §6.1) ni en RF-AUT/USR. Si se quiere, hay que agregar requisitos y vista |
| 08, 09 Contacto | Campo **Documento de identificación (CC)** | Datos del contacto: nombre, teléfono, correo, estado (02 §5.3). Además suma un dato personal no previsto |
| 18, 22, 23, 29 Consola | **Planes** ("Empresarial Pro", "Salud Enterprise Pro", "Plan Pro Enterprise AI", "Escalamiento AI") | Solo existe el **tipo de cuenta** Piloto/Cliente, y es informativo (03 §4) |
| 25 Usuarios org. | "Límite del plan piloto: **10 asientos**" | El tipo de cuenta "no cambia funciones ni límites" (03 §4) |
| 26 Configuración org. | "El piloto habilita **soporte prioritario y telemetría**" | Ídem |
| 19, 21, 23 Asistente | Rol **SuperAdmin**, "acceso total", "métodos de **facturación**" | Solo existen Asesor y Administrador. No hay facturación |
| 25 Usuarios org. | "Los administradores gestionan **facturación**"; "sesiones en la **app móvil**" | Sin facturación ni app móvil (01 §6) |
| 13, 29 | "Notificaciones de **agenda y citas**", "Sincronización masiva de **citas médicas**" | Las integraciones de agenda están fuera (ADR-014) |

## 3. Requisitos sin pantalla o incompletos (🟠)

**Agente de IA (ADR-014, entró al MVP el 15 oct):**
- **14 Configuración no tiene la sección del agente:** interruptor (apagado por defecto), información del negocio, nombre y tono, reglas de escalada, preguntas y respuestas fijas y tope diario (RF-CFG-04 a 09). Es el hueco más grande.
- **"Devolver al bot"** (05, 11, 16) es una acción **que la arquitectura no define**. ADR-014 solo describe cuándo el agente escala a un asesor, no cómo vuelve. Hay que decidirlo (y escribirlo en ADR-014) o quitar el botón.
- La bandeja no muestra ningún **mensaje del agente** diferenciado de uno humano (ADR-014 punto 13).
- **02 Dashboard no tiene RF-MET-09** (% de conversaciones resueltas por el agente sin escalar).

**Bandeja / conversación (V10–V12):**
- Estados de envío: solo aparece ✓✓. Faltan *enviando*, *error* con **Reintentar** y "No se pudo enviar" (RF-CNV-09).
- Adjuntos: no hay imagen, audio ni documento, ni los estados "aún descargando" y **"archivo no recuperado"** (RF-CNV-10b/c/e). Tampoco el marcador de tipo no soportado (RF-CNV-10).
- Mensajes de sistema ("Conversación asignada a…", "Estado cambiado a…").
- Compositor bloqueado por **organización suspendida** y por **conversación de otro asesor** (V11).
- Indicador de **tiempo real desconectado** (L01) y contador de sin responder en la navegación.
- 05: el panel muestra **"Reabrir conversación" y "Cerrar conversación" a la vez** para una conversación abierta. Debe aparecer uno según el estado.

**Configuración del negocio (V50):** falta la **sección WhatsApp de solo lectura** (número, estado, última verificación, texto "La gestiona el equipo de Numi"), RF-CFG-02.

**Usuarios (V40–V42):**
- 07 Invitar: **falta el campo Nombre** (RF-USR-02) y la línea "el enlace vence en 7 días".
- 07/10 Lista: falta la columna **Disponible para asignación automática** (RF-USR-06). El estado "Invitado" no distingue *pendiente* de *vencida*. Faltan *Reenviar* y *Cancelar invitación* (RF-USR-03).
- **V42 Detalle del usuario** no existe: rol, interruptor de disponibilidad y desactivar con el aviso "sus {n} conversaciones pasarán a Sin asignar".

**Contactos (V20–V22):**
- 04 Tabla y filtros: falta **Responsable** (columna y filtro, RF-CON-06).
- 09 Crear: agrega "Estado inicial" (el contacto nuevo nace en *Nuevo*; el estado solo se edita). Falta el error de **teléfono duplicado** con *Ver contacto* (RF-CON-07).

**Métricas (02 Dashboard = V30):**
- "Tiempo **promedio**" como valor principal; debe ser la **mediana**, con el promedio secundario (RF-MET-02), y llevar texto de ayuda: con el agente mide "cuánto tardó una persona en entrar".
- Falta el selector de período completo (mes actual, mes anterior, 30 días, rango) y el cambio **Todo el negocio / Por asesor** (RF-MET-06/07).
- Tabla por asesor: pide *contactos asignados*; muestra "Abiertas ahora".
- La vista se llama "Dashboard". El inventario la llama *Métricas* y define la Bandeja como inicio. Hay que alinear el nombre.

**Vistas ausentes:** V05 *Sin acceso* (usuario desactivado y organización suspendida), V60 (no encontrada, error, sesión expirada), enlace de invitación **vencido** (V02), estados vacío, carga y error de la bandeja.

**Consola:**
- 26 Configuración: **faltan las Notas internas** (RF-PLT-ORG-11, P0). Además dice que el administrador cambia la asignación "desde la consola de operación interna"; es **desde su configuración del CRM** (RF-PLT-ORG-12).
- 26: la lista de sectores no coincide con la del asistente (20). Debe ser la misma: clínica estética, inmobiliaria, odontología, veterinaria, otro.
- 24 WhatsApp: no muestra el **método de conexión** (Manual / Registro integrado), ni el estado *sin configurar* con *Conectar con Meta* (ADR-001, modelo objetivo Tech Provider), ni el error "número ya conectado a otra organización" (RF-PLT-WA-07).
- 25 Usuarios: solo se ve el menú de una invitación pendiente. Falta el menú de usuario activo (*restablecer contraseña*, *cambiar rol*, *desactivar*) y el de desactivado (*Reactivar*).
- 18: el estado vacío trae un **formulario rápido de alta** que salta el asistente de 3 pasos (RF-PLT-ORG-05).
- 28: "trazabilidad de acciones de operadores **y usuarios**". La auditoría es solo de operadores (RF-PLT-AUD-01). Y "Expira en **48 horas**" contradice los 7 días.
- 36: "Anthony (**Admin**)" y "Franco (**Operador**)" sugieren dos roles de operador. Hay uno solo.

## 4. Revisión pantalla por pantalla

| # | Pantalla | Vista | Estado | Hallazgos |
|---|---|---|---|---|
| 01 | Inicio de sesión | V01 | ✅ con ajustes | Correcta. "Recordarme" no está en la especificación. Faltan los estados de error, desactivado y sesión expirada. La marca dice "v2.4 · multi-agente" |
| 02 | Dashboard | V30 | 🟠 | Promedio en vez de mediana; falta RF-MET-09, el selector de período y la vista por asesor; el nombre difiere |
| 03 | Recuperar contraseña | V03 | ✅ | Falta el estado de confirmación (mismo mensaje exista o no el correo) |
| 04 | Contactos | V20 | 🟠 | Falta Responsable (columna y filtro) |
| 05 | Bandeja | V10–V12 | 🟠 | Buena base (filtros por rol, ventana 24 h, historial). "Devolver al bot" sin definir; Cerrar y Reabrir juntos; "Conversaciones: 3" (ver §5); sin estados de envío ni adjuntos |
| 06 | Contactos sin resultados | V20 | ✅ | Coincide con el estado "sin resultados" |
| 07 | Invitar usuario | V41 | 🟠 | Falta Nombre y la línea de los 7 días |
| 08 | Detalle de contacto | V21 | 🟠 | Campo CC fuera de modelo; varias conversaciones por contacto |
| 09 | Nuevo contacto | V22 | 🟠 | CC y "Estado inicial" de más; falta el error de duplicado |
| 10 | Usuarios | V40 | 🟠 | Sin disponibilidad, sin pendiente/vencida, sin Reenviar/Cancelar; menú en vez de V42 |
| 11 | Bandeja 24 h vencida | V11 | 🟠 | El bloqueo está bien; ofrece **plantillas** (fuera de alcance); el texto difiere del canónico |
| 12 | Restablecer contraseña | V04 | ✅ | Faltan los estados de enlace vencido y de éxito |
| 13 | Mi perfil | — | 🟠 | Vista sin requisito; "sede central"; "agenda y citas" |
| 14 | Configuración | V50 | 🔴/🟠 | **Sin agente de IA ni WhatsApp de solo lectura**; logotipo y colores fuera de alcance; "Rotación / Siguiente: Valentina" sugiere turno rotativo, cuando la regla es **menor carga** |
| 15 | Aceptar invitación | V02 | ✅ con ajustes | Correcta; falta el estado de enlace vencido |
| 16 | Bandeja asesor sin asignar | V10/V11 | ✅ | El asesor ve solo Mías/Sin asignar y "Tomar" (RF-CNV-11). Sobra "Devolver al bot" |
| 17 | Organizaciones | P20 | ✅ | Coincide con RF-PLT-ORG-01/03/04. Jerga: "Núcleo operativo activo" |
| 18 | Organizaciones vacío | P20 | 🔴 | "Bases independientes", planes, alta rápida que salta el asistente |
| 19 | Asistente paso 2 | P21 | 🟠 | SuperAdmin y facturación. El texto de invitación está bien |
| 20 | Asistente paso 1 | P21 | ✅ | Campos, sectores, duplicado y fechas del piloto coinciden |
| 21 | Asistente paso 3 | P21 | 🟡 | "SuperAdmin"; teléfono del administrador (pertenece al contacto principal) |
| 22 | Organizaciones sin resultados | P20 | 🟡 | La columna "Plan" no existe |
| 23 | Confirmación de alta | P21 final | 🔴 | Acciones correctas. Promesa de E2E y de MinSalud, plan inventado, "Bogotá" para una clínica de Envigado |
| 24 | WhatsApp con error | P24 | ✅ con ajustes | Diagnóstico, token de solo escritura y errores sin contenido: bien. Ruta de navegación equivocada ("Clínica Bella Piel"); falta el método de conexión |
| 25 | Usuarios org. | P23 | 🟠 | Regla del último administrador y enlace copiado: bien. "10 asientos", facturación, app móvil |
| 26 | Configuración org. | P26 | 🟠 | **Faltan las notas internas**; sectores distintos; texto de asignación equivocado |
| 27 | Resumen org. | P22 | ✅ | Preparación 4/4, uso de 7 días y datos clave coinciden. Falta el extracto de notas |
| 28 | Actividad | P25 | 🔴 | **"Acceder al CRM"**; usuarios en la auditoría; 48 h; hash |
| 29 | Modal suspender | P27 | 🔴 | La estructura del modal es correcta; los textos de impacto son incorrectos; el fondo trae datos de agenda y plan |
| 30 | Modal reactivar | P27 | 🔴 | El modal está bien; el fondo trae la "cola retenida" y "pacientes" |
| 31 | Configurar 2FA | P02 alta | ✅ | QR, clave, 6 dígitos y *Activar*: coincide (TOTP de Supabase, ADR-007) |
| 32 | Demasiados intentos | P02 | 🟡 | Coincide; confirmar el bloqueo real de Supabase (5 min / IP) |
| 33 | Acceso denegado | V60 | ✅ | Texto canónico exacto |
| 34 | Verificación 2FA | P02 | ✅ | Coincide; quitar "cifrado cuántico" |
| 35 | Entrada consola error | P01 | 🔴 | "Mantener sesión iniciada" |
| 36 | Auditoría global | P30 | ✅ con ajustes | Filtros, tabla, detalle y paginación coinciden. Hash, 180 días, roles Admin/Operador |
| 37 | Panel general | P10 | 🔴 | Tarjetas y listas correctas; el bloque de infraestructura es inventado |

## 5. Datos de ejemplo inconsistentes (🟡)

Los datos de ejemplo no son requisito, pero sí se contradicen entre pantallas y confunden a quien revise:

- **Varias conversaciones por contacto** (05 y 08: "Conversaciones: 3", con fechas). El modelo dice **una conversación por contacto** (02 §4). Esto sí es conceptual: hay que corregirlo.
- Juan David Mejía: +57 301 667 3320 en 04 y +57 315 889 4210 en 11.
- Usuarios de Clínica Bella Piel: 5 (17), 6 (25), 4 (26/28), 4/5 (27), 18 (29).
- Ciudad de Bella Piel: Medellín y Bogotá (29). Aurora: Envigado y Bogotá (23).
- Correo de Mariana: mariana@bellapiel.co y mariana.toro@bellapiel.com (28).
- 15: invitación para laura@ "como Asesor", pero Laura es la Administradora.
- 13/14: encabezado "Numi CRM Dental & Estética".
- Sector de Bella Piel: "Clínica estética" y "Cuidado Personal y Belleza" (28).

## 6. Prioridad sugerida para la siguiente ronda de diseño

1. **14 Configuración:** agregar el agente de IA y WhatsApp de solo lectura; quitar marca y colores.
2. **Quitar las 8 contradicciones de §1**, sobre todo "Acceder al CRM" (28) y las promesas de seguridad o cumplimiento (18, 23, 34, 36).
3. **Decidir "Devolver al bot"**: definirlo en ADR-014 o quitarlo.
4. **Bandeja:** estados de envío y Reintentar, adjuntos y "archivo no recuperado", mensaje del agente, compositor bloqueado por suspensión.
5. **Una conversación por contacto** en 05 y 08.
6. Pantallas faltantes: V05 (dos variantes), V42, V60, notas internas en P26.
7. Limpiar planes, SuperAdmin, facturación, asientos y plantillas en todas las pantallas.
