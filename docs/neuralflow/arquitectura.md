# NeuralFlow — Arquitectura

## Diagrama de componentes

```mermaid
graph TD
    U[Usuario] --> FE[Frontend]
    FE --> API[API NeuralFlow]
    API --> DB[(Base de datos)]
    API --> K[Keycloak]
    API --> M[MinIO]
    API --> V[Vault]
    API --> N[Servicio de notificaciones]
```

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| Frontend | — |
| Backend / API | — |
| Base de datos | — |
| Notificaciones | — |

!!! note
    Completar según las tecnologías definidas para el proyecto.
