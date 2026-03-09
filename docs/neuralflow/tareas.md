# NeuralFlow — Gestión de tareas

## Ciclo de vida de una tarea

```mermaid
stateDiagram-v2
    [*] --> Pendiente
    Pendiente --> EnProgreso : Iniciar
    EnProgreso --> EnRevision : Solicitar revisión
    EnRevision --> EnProgreso : Requiere cambios
    EnRevision --> Completada : Aprobada
    Pendiente --> Cancelada : Cancelar
    Completada --> [*]
    Cancelada --> [*]
```

## Campos de una tarea

| Campo | Descripción |
|---|---|
| `Título` | Descripción breve de la tarea |
| `Asignado a` | Usuario responsable |
| `Prioridad` | Baja / Media / Alta / Crítica |
| `Fecha límite` | Fecha de vencimiento |
| `Estado` | Pendiente / En progreso / En revisión / Completada |
| `Etiquetas` | Categorías libres |
