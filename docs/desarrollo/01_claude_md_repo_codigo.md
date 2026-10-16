# `CLAUDE.md` para el repositorio de código

- **Fecha:** 2026-10-16
- **Qué es:** el borrador del `CLAUDE.md` que va en el repositorio de desarrollo (`numi-crm`), no en este.
- **Por qué está aquí:** el repositorio de código todavía no existe. Cuando se cree, se copia el bloque de la sección 2 a su raíz como `CLAUDE.md`.

El `CLAUDE.md` de **este** repositorio describe el método de la etapa de arquitectura: una decisión por sesión, opciones antes que respuestas, no ampliar el alcance. **Nada de eso sirve construyendo.** El repositorio de código necesita otro: convenciones, cómo correr las pruebas, cómo desplegar y las reglas que los ADR convirtieron en obligaciones de código.

Este documento **no decide nada nuevo**. Todo lo que afirma viene de un ADR y lo cita. Donde algo no está decidido —la estructura de carpetas, por ejemplo— aparece marcado como **propuesta**, para que Franco y Anthony lo cierren en la reunión de arranque.

---

## 1. Qué cambia respecto del `CLAUDE.md` de arquitectura

| Regla de arquitectura | Qué pasa en el repositorio de código |
| --- | --- |
| Una decisión por sesión | No aplica. Las decisiones están tomadas |
| Opciones antes que respuestas | No aplica para lo ya decidido. Sigue valiendo para lo que ningún ADR cubre |
| No amplíes el alcance | **Sigue valiendo**, y con más fuerza: el calendario está ajustado |
| No inventes; cita la fuente | **Sigue valiendo.** Ahora la fuente es el ADR o el requisito |
| Verifica los datos de proveedores | **Sigue valiendo.** Ya falló una vez con Fastify en Workers |
| Escribe en español | **Sigue valiendo** para la conversación y la documentación. El código va en inglés |

Lo que se agrega: convenciones de código, comandos, y las reglas que no se pueden olvidar porque fallan en silencio.

---

## 2. El archivo, listo para copiar

Todo lo que sigue, entre las dos líneas de guiones, es el contenido de `CLAUDE.md` del repositorio de código.

---

# CLAUDE.md — Numi CRM

Este archivo le da a Claude el contexto y las reglas de trabajo de este repositorio. Se lee al inicio de cada sesión.

**La arquitectura ya está decidida.** Las quince decisiones viven en el repositorio `numi-crm-architecture`, en `docs/adr/`. Este repositorio las implementa. Si algo aquí contradice un ADR, **manda el ADR**; avisa del conflicto antes de seguir.

## 0. Cómo arranca cada sesión

Franco o Anthony abren con algo corto. Claude, en orden:

1. **Lee el estado:** este archivo, el tablero de tareas y el último commit.
2. **Reporta en un máximo de 6 líneas:** en qué fase está el desarrollo, qué se construyó por última vez y cuál es el siguiente paso del plan.
3. **Propone la agenda** y espera confirmación antes de escribir código.
4. **Trabaja**, siguiendo la sección 4.
5. **Cierra** dejando el trabajo en un estado que pasa las pruebas.

## 1. Qué es Numi CRM

CRM conversacional **multiempresa** para WhatsApp, vendido como servicio a negocios colombianos. Dos aplicaciones: el **CRM** (asesores y administradores de cada negocio) y la **consola de plataforma** (operadores de Numi, bajo `/plataforma`).

- **MVP:** 30 de noviembre de 2026. Revisión de avance el 16 de noviembre. Piloto el 1 de diciembre.
- **Equipo:** Franco y Anthony, ambos desarrollando, tiempo parcial. No hay más personas.
- **Volumen:** 500 conversaciones al mes por negocio; 2 o 3 negocios en el piloto.
- **Legal:** Ley 1581 de 2012. Las conversaciones pueden traer **datos sensibles de salud**. El negocio es responsable del tratamiento; Numi es encargado.

## 2. Stack

| Capa | Qué | ADR |
| --- | --- | --- |
| Lenguaje | TypeScript | — |
| Backend | Hono en Cloudflare Workers | 003 |
| Frontend | React con Vite, página única, servida desde Workers | 009 |
| Componentes | shadcn/ui sobre Radix, copiado al repositorio | decidido 2026-10-16 |
| Base de datos | Postgres de Supabase (plan gratuito) vía Hyperdrive | 003, 015 |
| Identidad | Supabase Auth solo como identidad | 007 |
| Archivos | R2, servido por el Worker en flujo | 004, 010 |
| Asíncrono | Tres colas de Cloudflare Queues y **un** Cron Trigger | 005, 012, 014 |
| Tiempo real | Supabase Realtime, Broadcast desde el servidor | 008 |
| Correo | Resend con dominio propio | 011 |
| Agente | API de Anthropic, detrás de su propia cola | 014 |

**Fastify no funciona en Cloudflare Workers.** Está verificado y es la razón de que el backend sea Hono. No lo reintroduzcas.

## 3. Las reglas que no se negocian

Estas no son preferencias de estilo. Cada una protege un requisito cuyo incumplimiento no se nota hasta que es tarde. Si una tarea parece exigir romper una, **para y pregunta**.

### 3.1 Toda consulta pasa por la función de transacción

Hay **una sola función** en el código que abre transacciones contra Postgres. Recibe la organización y la operación a ejecutar, y fija el contexto con `set_local` (ADR-006).

- Ninguna consulta abre una transacción por su cuenta.
- Ningún módulo importa el cliente de Postgres directamente.
- Si una operación necesita varias escrituras atómicas —crear contacto, conversación y mensaje, por ejemplo— van en **una sola** llamada a esa función (`arquitectura.md` §6.6).

Sin esta regla el aislamiento depende de que nadie se olvide nunca, y con dos personas con prisa alguien se olvida.

### 3.2 La organización nunca viene del cliente

Sale siempre de la membresía verificada del usuario, resuelta en el servidor. Si una ruta recibe un identificador de organización, se verifica contra la membresía antes de usarlo (ADR-006, ADR-007).

### 3.3 Nada sensible llega a los registros

Los registros reciben **identificadores, no objetos**. Registrar un objeto de evento completo "para depurar" es exactamente como se filtra contenido sensible (ADR-011).

Nunca se registra: texto de mensajes, nombres o teléfonos de contactos, tokens (ni parciales), contenido de archivos, ni nombres originales de archivo que puedan identificar a una persona.

### 3.4 Los tokens se descifran en memoria y nunca se leen hacia la interfaz

AES-GCM con Web Crypto. La clave es un secreto del Worker, **no está en la base**. Se descifra solo en el momento de llamar a Meta. La consola muestra que el token está guardado y cuándo se actualizó, nunca el valor ni un fragmento (ADR-001, ADR-011).

### 3.5 Las vistas SQL se declaran con `security_invoker`

Sin eso una vista se ejecuta con los permisos de su dueño y **devuelve datos de todas las organizaciones**. Es el error más grave posible del proyecto y es silencioso (ADR-006, ADR-013).

### 3.6 Las migraciones son solo aditivas

Agregar columnas y tablas; no renombrar ni borrar, durante todo el MVP. La reversión de Cloudflare cubre el código, **no el esquema**, y con el plan gratuito de Supabase no hay copias del proveedor (ADR-011, ADR-015).

### 3.7 Sin estado en memoria entre peticiones

No hay variables de módulo que acumulen datos, ni cachés en memoria, ni contadores. Un Worker puede morir entre dos peticiones. Todo estado vive en la base (ADR-003).

### 3.8 El framework vive solo en la capa de entrada

Rutas, validación y serialización usan Hono. El código de negocio **no importa nada de Hono** y recibe sus dependencias como parámetros (ADR-003).

### 3.9 WhatsApp vive detrás de la interfaz de canal

Nada específico de WhatsApp fuera de ella, ni en la entrada ni en la salida (ADR-012). La interfaz recibe la organización en toda llamada (ADR-001).

### 3.10 Toda biblioteca de npm se verifica en el runtime de Workers

Se prueba que **funcione**, no solo que compile. Vale especialmente para criptografía y para el cliente de Postgres (ADR-003).

## 4. Cómo se trabaja

1. **No inventes.** Si falta un dato, pregunta. Si propones algo que ningún ADR cubre, márcalo como propuesta y explica por qué.
2. **Cita la fuente.** El ADR, o el identificador del requisito (`RF-CNV-08`, `RF-PLT-WA-02`).
3. **Verifica los datos de proveedores** contra su documentación oficial, con enlace. Tu conocimiento puede estar desactualizado.
4. **No amplíes el alcance.** El calendario está ajustado. Si crees que algo debería entrar, propónlo con su costo.
5. **Toda pieza nueva justifica qué problema resuelve hoy** y dice cuándo dejaría de bastar.
6. **Separa hechos, supuestos y recomendaciones.**
7. **Escribe en español,** claro y directo, sin relleno.

## 5. Convenciones de código

- **Código, tablas y endpoints en inglés.** Documentación y conversación en español (`CLAUDE.md` de arquitectura §9).
- **Inmutabilidad:** se crean objetos nuevos, no se mutan.
- **Archivos pequeños y enfocados:** 200–400 líneas típicas, 800 como máximo. Organizados por dominio, no por tipo.
- **Funciones cortas:** menos de 50 líneas, sin anidamiento profundo.
- **Validación en el borde:** toda entrada de usuario se valida al entrar, con esquemas.
- **Errores manejados:** nada de fallos silenciosos; mensajes útiles sin filtrar datos sensibles.
- **Sin `console.log`** en el código que se despliega. Los registros pasan por el registrador, que aplica la regla 3.3.
- **Sin valores fijos en el código:** la configuración viene del entorno; los secretos, de los secretos del Worker.
- **Fechas en `timestamptz`.** Presentación en hora de Bogotá, formateo centralizado en una utilidad (`RF-TRV-05`).

## 6. Estructura del repositorio

> **Propuesta.** Ningún ADR la fija. Conviene cerrarla en la reunión de arranque y actualizar esta sección con lo acordado.

```
numi-crm/
├── CLAUDE.md
├── apps/
│   ├── api/              # Worker de la API (Hono)
│   └── web/              # Aplicación de página única (React + Vite)
├── packages/
│   ├── core/             # Código de negocio, sin dependencias del framework
│   │   ├── identity/     # Identidad y organizaciones
│   │   ├── channel/      # Interfaz de canal; WhatsApp detrás de ella
│   │   ├── conversations/
│   │   ├── contacts/
│   │   ├── assignment/
│   │   ├── metrics/
│   │   ├── console/
│   │   └── audit/
│   └── db/               # Migraciones, roles, la función de transacción
└── tests/
    └── isolation/        # Las siete pruebas de RF-TRV-01
```

Los módulos de `core/` salen de `CLAUDE.md` de arquitectura §7, sección 4. Cada uno declara de qué datos es dueño y qué eventos emite.

## 7. Comandos

> **Propuesta.** Se completa al montar el repositorio, en la fase 0.

| Qué | Comando |
| --- | --- |
| Desarrollo | `npm run dev` |
| Pruebas | `npm test` |
| **Pruebas de aislamiento** | `npm run test:isolation` |
| Tipos | `npm run typecheck` |
| Migraciones | `npm run db:migrate` |
| Despliegue | Automático desde GitHub. La rama principal va a producción |

**Las pruebas de aislamiento corren antes de cada despliegue. Si alguna falla, no se despliega** (ADR-011).

## 8. Pruebas

- **Las siete pruebas de aislamiento van primero**, antes que las vistas. `RF-TRV-01` es la condición de éxito del piloto y el único innegociable verificable de forma automática en cada cambio (ADR-006).
- Casos que deben estar cubiertos y es fácil olvidar:
  - Sin contexto de organización fijado, **cero filas** (la política falla cerrado).
  - Una fila de la tabla de eventos **sin organización** no la ve ningún rol de negocio.
  - Una consulta de `app_platform` contra mensajes **falla por permisos**, no devuelve datos filtrados.
  - Ningún rol tiene `bypassrls` (se verifica consultando el catálogo).
  - La **configuración del agente** de una organización nunca llega a la conversación de otra (ADR-014).
  - Un usuario de A pidiendo el archivo de un mensaje de B recibe **el mismo resultado que si no existiera**, no un error distinto (ADR-010).
- **La firma del webhook se prueba con tildes y emoji.** Meta calcula la firma sobre el cuerpo con unicode escapado y hexadecimal en minúsculas; es el error típico y silencioso (ADR-005).

## 9. Definición de "terminado" por vista

Una vista no está terminada hasta que cumple las cuatro (`docs/desarrollo/00_arranque.md` §4):

1. **Los tres estados** de `RF-TRV-04`: carga, vacío y error. No solo el camino feliz.
2. **Accesible** (`RF-TRV-06`): navegable con teclado, foco visible, contraste suficiente.
3. **Formato regional** (`RF-TRV-05`): fechas como "12 oct 2026", horas de 12 h, zona de Bogotá.
4. **Aislada:** toda consulta pasa por la función de transacción y nunca recibe `organization_id` del frontend.

## 10. Commits

Formato `<tipo>: <descripción>`, con tipos `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`. Mensajes en inglés, atómicos.

Antes de cualquier commit: sin secretos en el código, entradas validadas, consultas parametrizadas, nada sensible en los registros.

## 11. Dónde está cada cosa

| Qué busco | Dónde |
| --- | --- |
| Por qué algo se decidió así | `numi-crm-architecture/docs/adr/NNN-*.md` |
| Cómo encajan las piezas | `numi-crm-architecture/docs/architecture/arquitectura.md` |
| Qué debe hacer el CRM | `numi-crm-architecture/docs/02_documento_maestro_v1.1.md` |
| Qué debe hacer la consola | `numi-crm-architecture/docs/03_consola_plataforma_v1.1.md` |
| En qué orden se construye | `numi-crm-architecture/docs/desarrollo/00_arranque.md` |
| Qué pasó en sesiones previas | `numi-crm-architecture/docs/PROGRESO.md` |

---

## 3. Qué falta para que este archivo esté completo

Tres cosas, todas de la fase 0:

1. **La estructura de carpetas** (sección 6), a cerrar en la reunión de arranque.
2. **Los comandos reales** (sección 7), al montar el repositorio.
3. **El tablero de tareas** que la sección 0 menciona: dónde se lleva el avance día a día. Hoy no existe.

## 4. Qué hacer con el repositorio de memoria

El grafo de contexto de Engram está cargado bajo el proyecto `numi-crm-arquitecture`, detectado por el remoto de git. **El repositorio de código será un proyecto distinto**, así que una búsqueda desde allí no verá estos nodos por defecto.

Dos salidas, a elegir al crear el repositorio:

- Buscar con `all_projects: true` cuando haga falta contexto de arquitectura. Es lo más simple y no requiere preparación.
- Fusionar ambos proyectos en Engram, para que el contexto sea uno solo. Es más cómodo después, pero mezcla la bitácora de arquitectura con la de desarrollo.

**Recomendación:** empezar con la primera. Es reversible, y durante las primeras semanas la consulta de arquitectura será frecuente pero breve.
