# TruckManager — Flujo de transporte

## Ciclo de vida de un viaje

```mermaid
stateDiagram-v2
    [*] --> Programado
    Programado --> EnCurso : Conductor inicia viaje
    EnCurso --> EnDestino : Llegada al destino
    EnDestino --> Completado : Entrega confirmada
    EnCurso --> Incidente : Novedad reportada
    Incidente --> EnCurso : Novedad resuelta
    Completado --> [*]
```

## Estados del viaje

| Estado | Descripción |
|---|---|
| `Programado` | Viaje creado, pendiente de inicio |
| `En curso` | Conductor en ruta |
| `En destino` | Llegó al punto de entrega |
| `Completado` | Entrega exitosa confirmada |
| `Incidente` | Novedad activa durante el trayecto |
