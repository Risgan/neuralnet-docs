# TruckManager — Arquitectura

## Diagrama de componentes

```mermaid
graph TD
    OP[Operador] --> FE[Frontend]
    DR[Conductor] --> APP[App móvil]
    FE --> API[API TruckManager]
    APP --> API
    API --> DB[(Base de datos)]
    API --> K[Keycloak]
    API --> M[MinIO]
    API --> V[Vault]
```

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| Frontend | — |
| App móvil | — |
| Backend / API | — |
| Base de datos | — |

!!! note
    Completar según las tecnologías definidas para el proyecto.
