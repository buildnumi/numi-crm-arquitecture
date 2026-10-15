# Diagrama de contenedores — Numi CRM

`candidate.json` es la fuente del diagrama de arquitectura (Archify, tipo `architecture`),
derivado de `docs/architecture/arquitectura.md` §3.

## Estado

- **Contenido y layout: válidos.** `archify validate` no reporta ningún error de
  composición, límites, etiquetas ni geometría.
- **Pendiente: la entrega del HTML.** Falla en esta máquina por un problema de entorno,
  no del diagrama.

## El fallo de entrega

```
delivery/ownership-lost
finalize/receipt-publication (finalize-receipt-changed-during-inspection)
```

Archify escribe sus archivos de forma atómica y comprueba que nadie los haya tocado
entre la escritura y la lectura. En esta máquina esa comprobación falla siempre: el
archivo de bloqueo queda en 0 bytes. Se reprodujo en el repositorio y en `%TEMP%`, con
`finalize`, `validate` y `deliver`, así que no depende de la carpeta de destino.

Causa más probable: el escáner en tiempo real de Windows Defender interceptando la
escritura. `archify doctor` reporta la instalación sana.

## Cómo generar el HTML

Cualquiera de estas opciones, desde la raíz del repositorio:

1. Añadir una exclusión de Defender para la carpeta del repositorio y volver a ejecutar:

   ```
   node ~/.claude/skills/archify/bin/archify.mjs finalize architecture \
     .archify/architecture-numi-crm-20261015-1200/candidate.json \
     docs/architecture/numi-crm-contenedores.html --quality showcase --json
   ```

2. Ejecutar ese mismo comando en otra máquina o en WSL.

El candidato no necesita cambios: ya pasa la validación.
