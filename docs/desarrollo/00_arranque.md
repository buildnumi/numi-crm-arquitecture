# Arranque del desarrollo — Numi CRM

- **Fecha:** 2026-10-16
- **Base:** ADR-001 a ADR-014 aprobadas, `docs/architecture/arquitectura.md` terminado
- **Fecha límite:** MVP el 30 de noviembre de 2026; revisión de avance el 16 de noviembre; piloto el 1 de diciembre
- **Equipo:** Franco y Anthony, ambos desarrollando, a tiempo parcial

Este documento traduce la arquitectura en un plan de construcción. **No decide nada nuevo**: todo lo técnico viene de los ADR y lo funcional de `docs/02` y `docs/03`. Si algo aquí contradice un ADR, manda el ADR.

---

## 1. Lo que ya está decidido

| Capa | Decidido | ADR |
| --- | --- | --- |
| Backend | Hono en Cloudflare Workers | 003 |
| Frontend | React con Vite, página única, servida desde Workers | 009 |
| **Base de componentes** | **shadcn/ui sobre Radix** | Decidido el 2026-10-16 |
| Base de datos | Postgres de Supabase vía Hyperdrive, RLS y tres roles | 003, 006 |
| Identidad | Supabase Auth solo como identidad; autorización en el backend | 007 |
| Archivos | R2 servido por el Worker, en flujo, sin URL firmadas | 004, 010 |
| Asíncrono | Tres colas de Cloudflare Queues y **un** Cron Trigger | 005, 012, 014 |
| Tiempo real | Supabase Realtime, Broadcast desde el servidor | 008 |
| Correo | Resend con dominio propio | 011 |
| Agente | API de Anthropic, detrás de su propia cola | 014 |

### Por qué shadcn/ui sobre Radix

**El código se copia al repositorio en lugar de instalarse como dependencia.** Eso da dos cosas que importan en este calendario: la accesibilidad de Radix ya resuelta —foco atrapado en modales, navegación con teclado, ARIA—, que es exactamente lo que `RF-TRV-06` exige y lo más caro de construir a mano; y control total del marcado para adaptarlo sin pelear con la librería.

**Lo que cubre y lo que no**, contra `docs/02` §11.3:

| Grupo de componentes | Estado |
| --- | --- |
| Acciones, Formularios, Navegación, Superposiciones, Estados | **Cubierto** por la base |
| Datos (tabla, paginación, etiquetas) | **Cubierto**, con trabajo de adaptación |
| **Conversación** (8 piezas: burbujas, estados de envío, separadores, campo de escritura) | **A construir** — es propio del producto |
| **Métricas** (3 piezas: tarjeta de indicador, gráfico, tabla por asesor) | **A construir** — es propio del producto |

**Esta es la observación que ordena el trabajo de interfaz:** ninguna librería del mercado trae el grupo Conversación, que es el corazón del producto. La base resuelve lo genérico para que el tiempo se gaste en las 11 piezas que sí son de Numi.

---

## 2. Orden de construcción: camino crítico primero

El criterio: **construir primero lo que valida el piloto**, y que lo recortable quede naturalmente al final. Así, si el 16 de noviembre hay que recortar, se recorta lo último que se empezó en vez de negociar bajo presión.

### Fase 0 — Cimientos (antes de cualquier vista)

Sin esto nada de lo demás se puede probar.

1. **Repositorio y entornos.** Dos Workers (aplicación y API), entornos de desarrollo y producción, despliegue automático desde GitHub (ADR-011).
2. **Esquema inicial y los tres roles de base.** `app_user`, `app_platform`, `app_worker`, ninguno con `bypassrls`, políticas que fallan cerrado y `FORCE ROW LEVEL SECURITY` (ADR-006).
3. **Las siete pruebas de aislamiento, corriendo en CI.** Van antes que las vistas, no después: `RF-TRV-01` es la condición de éxito del piloto y es el único innegociable verificable de forma automática en cada cambio.
4. **La función de transacción** que fija el contexto de organización. Es el único lugar del código que abre transacciones (ADR-006).

> **Regla que no se negocia:** ninguna consulta abre transacciones por su cuenta. Si esta función no existe desde el día uno, el aislamiento pasa a depender de que nadie se olvide nunca — y con dos personas con prisa, alguien se olvida.

### Fase 1 — Entrada de mensajes (el corazón)

Es el camino que contiene las tres invariantes del sistema.

5. **Webhook** con validación de firma HMAC sobre el cuerpo crudo. **Probarlo con tildes y emoji antes de darlo por bueno** — ADR-005 advierte que es el error típico y silencioso.
6. **Tabla de eventos crudos** con `organization_id` nulo admitido, y guardado antes del 200.
7. **Cola de eventos y su consumidor:** crear contacto, conversación y mensaje **en una sola transacción** (`arquitectura.md` §6.6), con unicidad `(organization_id, wamid)`.
8. **Broadcast desde el servidor** y la consulta de respaldo de 60 s en el cliente.

### Fase 2 — Vistas del camino crítico

| Orden | Vistas | Por qué aquí |
| --- | --- | --- |
| 9 | **V01–V05** sesión, invitación, recuperación, sin acceso | Sin login no se prueba nada |
| 10 | **V10 Bandeja** y **V11 Conversación** + V12 panel del contacto | **El corazón del producto.** Es lo que el piloto valida |
| 11 | **Envío de mensajes** con ventana de 24 h y los cinco estados | Cierra el ciclo: recibir y responder |
| 12 | **P01, P02, P21** consola mínima: entrar, doble factor, crear organización | Hace falta para dar de alta un negocio y probar el CRM con datos reales |

**Al terminar la fase 2 el producto ya se puede demostrar**: se da de alta un negocio, entra un asesor, llega un mensaje y se responde. Es el hito que conviene tener antes del 16 de noviembre.

### Fase 3 — Lo que completa el MVP

| Orden | Vistas |
| --- | --- |
| 13 | **V20–V22** contactos |
| 14 | **V40–V42** usuarios del negocio, **V50** configuración |
| 15 | **P20, P22, P24, P27** organizaciones, resumen, WhatsApp, suspensión |
| 16 | **Multimedia** (ADR-004): cola de archivos, R2 en flujo, visor |
| 17 | **V30 Métricas** (ADR-013) y **P23, P25, P26** |
| 18 | **Agente de IA** (ADR-014) con sus topes de turnos y gasto |
| 19 | **V60** estados del sistema, y el Cron Trigger común |

### Lo recortable, en orden de corte

Si el 16 de noviembre el avance no alcanza, **se corta desde el final de esta lista hacia arriba**:

1. **P10 panel general** y **P30 auditoría global** — ya son P1, no P0.
2. **Agente de IA** — es alcance que se agregó el 2026-10-15 sobre un calendario ya ajustado; el CRM funciona sin él.
3. **P25 actividad** y buscador y filtros de **P20** — con 2 o 3 organizaciones no tienen usuario real.
4. **V30 métricas** reducida a los indicadores principales, sin desglose por asesor.
5. **Multimedia** degradada al marcador de tipo, que era el comportamiento previo a ADR-004.

**Lo que nunca se corta:** los innegociables de `CLAUDE.md` §8 — aislamiento, webhook sin pérdidas ni duplicados, orden de mensajes, ventana de 24 h, estados del mensaje, datos sensibles fuera de la consola, tokens cifrados y la auditoría de la consola.

---

## 3. Lista de puesta en marcha

Configuración que **no está en el código** y que, si se olvida, falla en silencio. De ADR-011 y de la auditoría de escalabilidad.

### Antes de escribir código

- [ ] **`max_size` de Hyperdrive en 40.** Hyperdrive admite ~100 conexiones y Supabase acepta 60 directas —**el mismo número en el plan gratuito y en Micro**, así que ADR-015 no cambia este valor—: sin fijarlo, las que sobran fallan bajo carga (`arquitectura.md` §11.2). Al cambiar el tamaño de cómputo, revisar este número en la misma operación.
- [ ] **Canal de Realtime sin acceso público.** Es un interruptor manual; un canal público no da error, simplemente deja de aislar.
- [ ] **Los tres roles de base creados sin `bypassrls`**, verificado por la prueba que consulta el catálogo.
- [ ] **Plan de pago de Workers.** El gratuito da 10 ms de CPU y retención de cola de 24 h, inaceptable para `RF-TRV-02`.
- [ ] **Latido diario contra la base**, dentro del Cron Trigger que ya existe. El plan gratuito de Supabase pausa el proyecto tras 7 días de poca actividad; un `SELECT 1` al día lo evita sin depender de que alguien entre (ADR-015).
- [ ] **Copias propias con `pg_dump`** desde GitHub Actions hacia R2. El plan gratuito **no incluye ninguna copia de seguridad** (ADR-015).

### Ruta crítica, no la controla el equipo

- [ ] **Verificación de negocio ante Meta.** **Es lo más urgente de esta lista.** Sin ella el piloto queda limitado a 2 clientes en vez de 20, y el trámite depende de un tercero.
- [ ] **Resend con SPF, DKIM y DMARC.** Sin correo no hay invitación y **no hay alta de organizaciones**. La propagación de DNS tampoco depende del equipo. ADR-011 recomendaba resolverlo en la semana del 19 de octubre.
- [ ] **Preparación para HIPAA en Anthropic**, antes de conectar cualquier cliente real. Se habilita por autoservicio con efecto inmediato, pero **es permanente**: decisión de una sola dirección.

### Antes del piloto

- [ ] **Medir V30** con 6.000 conversaciones y 60.000 mensajes en una organización (ADR-013).
- [ ] **Probar una restauración** de copia de seguridad. Con el plan gratuito, las únicas copias son las propias: si la restauración no funciona, **no hay segunda red** (ADR-015).
- [ ] **Calibrar el umbral de la alerta de webhook** con tráfico real durante la primera semana.

---

## 4. Definición de "terminado" por vista

Una vista no está terminada hasta que cumple las cuatro:

1. **Los tres estados** de `RF-TRV-04`: carga, vacío y error. No solo el camino feliz.
2. **Accesible** (`RF-TRV-06`): navegable con teclado, foco visible, contraste suficiente.
3. **Formato regional** (`RF-TRV-05`): fechas como "12 oct 2026", horas en formato de 12 h, zona horaria de Bogotá.
4. **Aislada:** toda consulta pasa por la función de transacción, nunca recibe `organization_id` del frontend.

---

## 5. Riesgos abiertos al arrancar

| Riesgo | Estado |
| --- | --- |
| **Verificación de negocio ante Meta** | Ruta crítica, fuera del control del equipo. Limita el piloto a 2 clientes si no se resuelve |
| **Calendario** | 111 requisitos P0 y 31 vistas en siete semanas con dos personas a tiempo parcial. **Nadie sumó el alcance total ni priorizó una sola lista de repliegue**; la sección 2 de este documento propone una, pendiente de validar con Anthony |
| **Configuración de Resend** | Bloquea el alta de organizaciones; depende de propagación de DNS |
| **Concentración en Supabase** | Postgres, identidad y tiempo real en un proveedor con antecedentes de caídas. Asumido conscientemente (ADR-011) |
| **Durabilidad sin copias del proveedor** | El plan gratuito no incluye copias de seguridad. Durante el piloto se opera con datos de salud y solo las copias que construya el equipo. Asumido conscientemente (ADR-015); las copias propias y su restauración probada son condición, no recomendación |

---

## 6. Qué falta decidir

Nada de arquitectura. Lo que queda es de ejecución y conviene cerrarlo en la reunión de arranque del design system (`docs/02` §13):

- **Fundamentos visuales:** colores, tipografía, espaciado (`docs/02` §11.2).
- **Reparto del trabajo** entre Franco y Anthony.
- **Validar el orden de construcción** de la sección 2 con Anthony, que tiene asignada la interfaz.
