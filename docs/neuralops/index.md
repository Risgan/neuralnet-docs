# NeuralOps

NeuralOps es una plataforma **SaaS multi-tenant** de gestión empresarial integral. Su objetivo es proporcionar soluciones modulares para digitalizar procesos administrativos, operativos y financieros con soporte para inteligencia artificial e integración total con el ecosistema NeuralNet.

## Qué hace

- **Automatiza procesos** administrativos y operativos
- **Gestiona múltiples módulos** empresariales (Ventas, Inventario, Finanzas, RRHH, Producción, etc.)
- **Reduce trabajo manual** y mejora trazabilidad
- **Soporta multi-tenancy** — cada empresa con datos aislados
- **Integra IA** para predicción, optimización y análisis
- **Centraliza información** en una plataforma única

## Stack Tecnológico — Resumen

| Capa | Tecnología | Propósito |
|---|---|---|
| **Frontend** | Angular 20 (recomendado) o React | Interface web moderna |
| **Backend** | .NET Core 8 | API, lógica de negocio, CQRS + Clean Architecture |
| **Base de datos** | PostgreSQL 17 | Datos multi-tenant |
| **IA/ML** | Python 3.11 + FastAPI | Microservicio de predicción |
| **Mensajería** | Kafka | Comunicación asíncrona |
| **Caché** | Redis | Sesiones y caché distribuida |
| **Autenticación** | Keycloak | IAM, SSO, RBAC |
| **Almacenamiento** | MinIO | Documentos y archivos |
| **Logging** | Serilog + Seq | Auditoría y observabilidad |

## Documentación

- [Stack Tecnológico Detallado](stack-tecnologico.md) — cada herramienta explicada
- [Arquitectura del Sistema](arquitectura.md) — diagramas y componentes
- [Módulos Empresariales](modulos.md) — qué cubre cada módulo
- [Modelo de Datos](base-datos.md) — tablas y estructura
- [Seguridad y Autenticación](seguridad.md) — JWT, RBAC, multi-tenancy
- [Angular vs React](angular-vs-react.md) — comparativa y decisión
- [Roadmap de Implementación](roadmap-implementacion.md) — orden paso a paso
