# Reconstructor

**Comando:** `/reversa-reconstructor`
**Rol:** agente independiente (fuera del pipeline de Descubrimiento)

---

## 🧱 El constructor bottom-up

Cuando las specs ya existen, el Reconstructor puede reconstruir todo el sistema a partir de ellas, una tarea a la vez. Lee solo lo necesario en cada paso, ejecuta una única tarea por sesión y pausa, así una reconstrucción larga nunca consume más tokens de los necesarios.

---

## Qué hace

El Reconstructor convierte las specs del Reversa en un **plan de reconstrucción ejecutable** y luego implementa cada tarea bajo demanda, bottom-up.

Opera en dos modos:

1. **Modo planificación** (primera invocación): lee un conjunto pequeño de archivos, infiere el árbol de dependencias y produce `reconstruction-plan.md` con la lista completa de tareas. El orden se elige por profundidad de dependencia: schema y entidades núcleo primero, hojas del árbol antes que sus dependientes, capa de API y flujos de usuario al final.
2. **Modo ejecución** (cada invocación posterior): toma la siguiente tarea no marcada en el plan, lee únicamente los archivos que la tarea declara necesitar, la implementa, la marca como hecha y se detiene.

Si existe una carpeta `migration/` finalizada, el Reconstructor pregunta si reconstruir a partir de las **specs originales** (fiel al legado) o de las **specs de la migración** (sistema nuevo en la stack objetivo), y etiqueta el plan en consecuencia.

---

## Por qué bottom-up

Componentes sin dependencias primero, después todo lo que va arriba. Esto evita stubs, andamios y retrabajo: cada capa se apoya en algo que ya existe.

## Por qué una tarea por sesión

Preservación de tokens. Cada tarea lleva solo el contexto que necesita. Puedes pausar y retomar en cualquier momento sin reconstruir contexto, y cualquier agente puede tomar la siguiente tarea sin releer todo el proyecto.

---

## Qué lee (planificación)

- `.reversa/state.json` (metadatos del proyecto, si existen)
- `_reversa_sdd/gaps.md` (cuando está disponible)
- `_reversa_sdd/confidence-report.md` (cuando está disponible)
- `_reversa_sdd/architecture.md`
- `_reversa_sdd/dependencies.md`
- `_reversa_sdd/traceability/code-spec-matrix.md` (cuando está disponible)
- `_reversa_sdd/migration/handoff.md` (cuando hay migración concluida)

Los archivos a nivel de unit (`<unit>/requirements.md`, `design.md`, `tasks.md`) no se leen en planificación, solo en la ejecución de la tarea correspondiente.

---

## Qué produce

| Archivo | Contenido |
|---------|-----------|
| `_reversa_sdd/reconstruction-plan.md` | Lista completa de tareas bottom-up con `Lee:` y `Listo cuando:` por tarea, más alertas pre-vuelo mapeadas desde `gaps.md` |

Durante la ejecución, el Reconstructor escribe el código real en el proyecto objetivo (según `paradigm_decision.md`/`target_architecture.md` cuando la fuente es la migración). Cada tarea finalizada se marca en `reconstruction-plan.md`.

---

## Cuándo usarlo

- Después de que `/reversa` terminó y quieres reimplementar el legado desde cero bajo las specs originales.
- Después de que `/reversa-migrate` terminó y quieres materializar el sistema nuevo a partir de las specs de la migración.
- Como camino de recuperación: pausa cuando quieras y retoma después sin reiniciar el contexto.

---

## Cómo se conecta

```
/reversa  →  specs (_reversa_sdd/)
                  │
                  ▼
/reversa-reconstructor  →  reconstruction-plan.md (una vez)  →  código, una tarea a la vez
```

Es **independiente del pipeline principal**: nunca bloquea `/reversa`, `/reversa-forward` ni `/reversa-migrate`. Se invoca por separado cuando eliges reconstruir.
