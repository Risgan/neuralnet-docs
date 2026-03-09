# Stack Tecnológico — NeuralOps

Desglose detallado de cada tecnología, qué hace y por qué se usa en NeuralOps.

---

## 🖥️ Backend — .NET Core 8

**Qué es:**  
Framework de Microsoft para crear APIs rápidas, seguras y multiplataforma.

**Para qué se usa:**
- Servir el **API REST** que consume Angular
- Manejar la lógica de negocio (validaciones, reglas, procesos)
- Conectarse a **PostgreSQL** usando **Entity Framework Core (ORM)**
- Integrarse con **Kafka** para tareas asíncronas
- Documentación automática con **Swagger/OpenAPI**

**Librerías principales:**
- **AutoMapper** → convierte entidades ↔ DTOs
- **FluentValidation** → valida datos de entrada robustamente
- **JWT Bearer** → valida tokens JWT de Keycloak
- **Serilog + Seq** → logging estructurado y auditoría
- **Entity Framework Core 8** → ORM para PostgreSQL

---

## 🎨 Frontend — Angular 20

**Qué es:**  
Framework de Google para construir aplicaciones web de una sola página (SPA) completas.

**Para qué se usa:**
- Renderizar dashboards, formularios y pantallas
- Consumir datos del backend vía HTTP/REST
- Manejar estado y eventos en tiempo real con **RxJS**
- Usar **Angular Material** para componentes UI profesionales
- Control de vistas según roles (ocultar/mostrar menús)

**Librerías principales:**
- **Angular Material** → componentes UI (tablas, formularios, diálogos)
- **RxJS** → programación reactiva
- **Keycloak-js** → login directo contra Keycloak
- **HttpInterceptor** → agregar JWT automáticamente a requests

**Alternativa: React**  
Si prefieres React, requiere elegir adicionalmente:
- Router: TanStack Router o React Router
- State: Zustand, Redux o Context API
- UI: Material-UI, Chakra o ShadcN

Ver [Angular vs React](angular-vs-react.md) para comparativa completa.

---

## 🗄️ Base de Datos — PostgreSQL 17

**Qué es:**  
Base de datos relacional de alto rendimiento, confiable y altamente escalable.

**Para qué se usa:**
- Guardar datos de usuarios, tenants, eventos, auditoría
- Soporte a transacciones ACID y constraints de integridad
- Escalabilidad horizontal con replicación y sharding
- Integración directa con Entity Framework Core

**Características clave:**
- Multi-tenant → datos aislados por `tenantId`
- JSON/JSONB native para configuraciones flexibles
- Full-text search integrada
- Triggers para auditoría automática

---

## 🔐 Keycloak

**Qué es:**  
Plataforma de identidad y gestión de accesos (IAM) open-source.

**Para qué se usa:**
- Gestión centralizada de **usuarios, roles y permisos (RBAC)**
- Autenticación con **SSO (Single Sign-On)**
- Emisión de **JWT + Refresh Tokens**
- Multi-tenant → cada cliente/empresa con su propio realm o realm compartido con isolamento
- Claims personalizados (ej: `tenantId` en el token)

**Flujo típico:**
1. Usuario intenta loguearse en Angular
2. Angular redirige a Keycloak
3. Keycloak emite JWT con `tenantId` y roles
4. Angular guarda el token y hace requests al backend
5. Backend valida el JWT y aplica RBAC

---

## 🤖 Inteligencia Artificial — Python 3.11 + FastAPI

**Qué es:**  
Lenguaje ideal para IA/ML. FastAPI expone modelos como microservicios REST.

**Para qué se usa:**
- Entrenar y servir modelos de predicción/clasificación
- **FastAPI** → exponer modelos como endpoints REST
- **TensorFlow / PyTorch** → frameworks de ML (si es necesario)
- **MLflow** → gestionar ciclo de vida de modelos
- **Pandas / NumPy** → procesamiento de datos
- **Redis** → cachear predicciones rápidas

**Casos de uso en NeuralOps:**
- Predicción de demanda por producto
- Optimización de inventario
- Análisis financiero y detección de anomalías
- Clasificación automática de documentos

---

## 📩 Kafka

**Qué es:**  
Broker de mensajería basado en colas para comunicación asíncrona entre servicios.

**Para qué se usa:**
- Comunicación **desacoplada** entre backend → IA → reportes
- Procesar tareas largas en segundo plano (reportes, auditoría, notificaciones)
- Garantizar entrega de mensajes
- Escalabilidad horizontal de procesos

**Ejemplo de flujo:**
1. Usuario crea un pedido en Ventas
2. Backend publica evento "PedidoCreado" en Kafka
3. Consumer de IA analiza el pedido y predice problemas
4. Consumer de reportes genera reporte de ventas del día
5. Consumer de auditoría registra la operación

---

## 📊 Serilog + Seq

**Serilog:**  
Librería de logging estructurado en .NET.

**Seq:**  
Servidor centralizado para visualizar y consultar logs.

**Para qué se usa:**
- Capturar trazas, errores y eventos en backend
- Consultar logs por tenant, usuario, módulo
- Auditoría completa: quién hizo qué, cuándo
- Alertas sobre errores críticos

**Ejemplo de log estructurado:**
```json
{
  "Timestamp": "2025-03-09T10:30:00Z",
  "Level": "Information",
  "MessageTemplate": "Usuario {UserId} creó pedido {PedidoId} en tenant {TenantId}",
  "UserId": "user-uuid-123",
  "PedidoId": "pedido-uuid-456",
  "TenantId": "tenant-uuid-789",
  "IP": "192.168.1.100"
}
```

---

## 📦 MinIO

**Qué es:**  
Sistema de almacenamiento objeto-oriented, compatible con S3.

**Para qué se usa:**
- Guardar documentos, reportes, facturas
- Archivos adjuntos de procesos
- Modelos entrenados de IA
- Alternativa privada a Amazon S3

**Flujo:**
1. Usuario sube un PDF en Compras
2. Backend guarda en MinIO y obtiene una URL firmada
3. Frontend descarga con la URL firmada

---

## 💾 Redis

**Qué es:**  
Base de datos en memoria de alta velocidad.

**Para qué se usa:**
- Cache de predicciones de IA (evita cálculos repetidos)
- Sesiones distribuidas del usuario
- Locks distribuidos para operaciones críticas
- Rate limiting para API

---

## 🏗️ Arquitectura — Clean + CQRS

### Clean Architecture
Separa la aplicación en capas independientes:

```plaintext
Domain (Entidades, reglas de negocio)
    ↓
Application (Casos de uso, DTOs, excepciones)
    ↓
Infrastructure (Base de datos, HTTP, Kafka)
    ↓
Presentation (Controllers, DTOs de respuesta)
```

**Ventaja:** cambios en un nivel no afectan otros. Si cambias de PostgreSQL a MongoDB, solo toca Infrastructure.

### CQRS (Command Query Responsibility Segregation)
Divide operaciones en dos tipos:

- **Commands** → escribir (crear, actualizar, eliminar)
  ```csharp
  public class CrearPedidoCommand { }
  public class CrearPedidoCommandHandler : ICommandHandler<CrearPedidoCommand> { }
  ```

- **Queries** → leer (consultas, reportes)
  ```csharp
  public class ObtenerPedidoPorIdQuery { }
  public class ObtenerPedidoPorIdQueryHandler : IQueryHandler<ObtenerPedidoPorIdQuery> { }
  ```

**Ventaja:** queries sin efectos secundarios, commands con validación completa. Mejor rendimiento, escalabilidad.

---

## 🔗 Patrones Arquitectónicos

| Patrón | Para qué | Ejemplo |
|---|---|---|
| **Repository** | Abstrae acceso a datos | `IProductoRepository.GetById()` |
| **Unit of Work** | Agrupa cambios en transacción | `unitOfWork.SaveChanges()` |
| **Mediator** | Desacopla commands/queries | `mediator.Send(command)` |
| **Decorator** | Agrega comportamiento | `[Authorize]` en controllers |
| **Factory** | Crea objetos complejos | `EntityFactory.Create()` |

---

## ✅ Resumen del Stack

| Componente | Tecnología | Razón |
|---|---|---|
| API REST | .NET Core 8 | Rendimiento, ecosistema, escalabilidad |
| Interfaz web | Angular 20 | Modularidad, TypeScript fuerte, empresarial |
| Datos | PostgreSQL 17 | Confiabilidad, ACID, escalabilidad |
| IAM | Keycloak | Multi-tenancy, SSO, RBAC, open-source |
| IA/ML | Python + FastAPI | Ecosistema ML robusto |
| Mensajería | Kafka | Desacoplamiento, escalabilidad |
| Caché/sesiones | Redis | Rendimiento, distribuido |
| Almacenamiento | MinIO | Privado, S3-compatible |
| Observabilidad | Serilog + Seq | Logging estructurado, auditoría |
