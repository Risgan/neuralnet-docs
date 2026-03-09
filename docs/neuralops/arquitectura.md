# NeuralOps — Arquitectura

## Diagrama de componentes

```mermaid
graph TD
    U[Usuario] --> FE[Frontend]
    FE --> API[API NeuralOps]
    API --> DB[(Base de datos)]
    API --> K[Keycloak]
    API --> M[MinIO]
    API --> V[Vault]
```

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| Frontend | — |
| Backend / API | — |
| Base de datos | — |
| Cola de mensajes | — |

!!! note
    Completar según las tecnologías definidas para el proyecto.
