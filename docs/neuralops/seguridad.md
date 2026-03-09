# Seguridad y Autenticación — NeuralOps

Sistema de autenticación, autorización y auditoría en NeuralOps.

---

## 🔐 Arquitectura de Seguridad

```plaintext
Angular (Frontend)
    │
    └→ Keycloak (Login)
         ↓ emite JWT con tenantId + roles
         ↓
    Backend .NET (valida JWT)
         ├→ Extrae tenantId del token
         ├→ Consulta roles en Keycloak (en caché con Redis)
         └→ Aplica filtros automáticos por tenantId en queries
         
Base de Datos (PostgreSQL)
    └→ Todos los queries filtrados por tenantId (no se cruzan datos)
```

---

## 🔑 Token JWT — Estructura

**Ejemplo de JWT emitido por Keycloak:**

```json
{
  "exp": 1726578221,
  "iat": 1726574621,
  "auth_time": 1726574600,
  "jti": "abc123def456ghi789",
  "iss": "https://auth.neuralnet.com/realms/neuralnet",
  "aud": ["neuralops-api"],
  "sub": "user-uuid-1234",
  "typ": "Bearer",
  "azp": "neuralops-frontend",
  "session_state": "xyz789xyz789xyz789",
  "scope": "openid profile email",
  
  "preferred_username": "jrueda",
  "email": "john@compania.com",
  "name": "John Álvaro Rueda Forero",
  "given_name": "John",
  "family_name": "Rueda",
  
  "tenant_id": "tenant-uuid-4567",
  "companies": ["compania-a", "compania-b"],
  
  "realm_access": {
    "roles": ["default-roles-neuralnet", "user"]
  },
  
  "resource_access": {
    "neuralops-api": {
      "roles": ["admin", "user", "view_reports"]
    },
    "neuralops-frontend": {
      "roles": ["admin", "user"]
    }
  }
}
```

**Campos clave:**

| Campo | Propósito |
|---|---|
| `sub` | ID único del usuario en Keycloak |
| `preferred_username` | Usuario con el que se logueó (ej: jrueda) |
| `email` | Email del usuario |
| `tenant_id` | 👈 **CRÍTICO** — identifica la empresa/organización |
| `realm_access.roles` | Roles globales en el sistema |
| `resource_access.neuralops-api.roles` | Roles específicos para el API |
| `exp` | Cuándo expira el token (típicamente 5-15 minutos) |

---

## 🔄 Bearer Token + Refresh Token

**Bearer Token:**
- Token de acceso corta vida (ej: 5 min)
- Se envía en header `Authorization: Bearer <token>`
- Si expira, frontend obtiene otro sin que el usuario lo sepa

**Refresh Token:**
- Token de larga vida (ej: 30 días)
- Se guarda en HttpOnly cookie (seguro contra XSS)
- Backend lo usa para obtener nuevo Bearer Token sin que el usuario se loguee de nuevo

**Flujo:**
```plaintext
1. Usuario inicia sesión
   └→ Backend ↔ Keycloak
   └→ Recibe Bearer + Refresh Token

2. Angular hace request con Bearer Token
   └→ Backend valida JWT
   └→ Request exitoso ✅

3. Bearer Token expira
   └→ Backend rechaza (401 Unauthorized)
   └→ Angular automaticamente envía Refresh Token
   └→ Keycloak emite nuevo Bearer Token
   └→ Angular reintenta el request original ✅

4. Refresh Token expira
   └→ Usuario debe loguearse de nuevo
```

**En código (Angular Interceptor):**
```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    if (token) {
      req = req.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`
        }
      });
    }
    return next.handle(req).pipe(
      catchError(error => {
        if (error.status === 401) {
          // Token expiró, intentar refresh
          return this.authService.refreshToken().pipe(
            switchMap(newToken => {
              req = req.clone({
                setHeaders: {
                  Authorization: `Bearer ${newToken}`
                }
              });
              return next.handle(req);
            })
          );
        }
        return throwError(error);
      })
    );
  }
}
```

---

## 🏢 Multi-Tenancy — Aislamiento de Datos

**Estrategia: Database per Tenant con shared schema**

El tenant se identifica por `tenantId` en cada tabla:

```sql
-- Master
CREATE TABLE public.usuarios (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,  -- ← La clave mágica
  nombre VARCHAR(150),
  email VARCHAR(150),
  activo BOOLEAN,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

-- Índice para rendimiento
CREATE INDEX idx_usuarios_tenant_id ON public.usuarios(tenant_id);
```

**En Entity Framework Core:**

```csharp
// Cada query se filtra automáticamente por TenantId
public class RepositorioUsuarios : IRepositorioUsuarios
{
  private readonly IDbContextFactory<NeuralOpsContext> _contextFactory;
  private readonly ICurrentTenantService _currentTenant;

  public async Task<Usuario> ObtenerPorId(Guid id)
  {
    using var context = _contextFactory.CreateDbContext();
    return await context.Usuarios
      .Where(u => u.TenantId == _currentTenant.TenantId)
      .FirstOrDefaultAsync(u => u.Id == id);
  }
}
```

**Ventajas:**
- ✅ Datos totalmente aislados por tenant
- ✅ Evita bugs donde un tenant ve datos de otro
- ✅ Escalable (agregar nuevo tenant = crear registro, no nuevo DB)

**Riesgo:** Si se olvida el `.Where(u => u.TenantId == _currentTenant.TenantId)`, se filtra mal. **Solución:** Abstract base repository que fuerza el filtro.

---

## ✅ RBAC — Control de Acceso Basado en Roles

**Roles en NeuralOps:**

| Rol | Permisos |
|---|---|
| **superadmin** | Acceso total al sistema, gestión de tenants |
| **tenant_admin** | Administrador de su empresa, gestión de usuarios y módulos |
| **manager** | Acceso a reportes, aprobaciones, dashboards |
| **user** | Acceso a funcionalidades base de su módulo asignado |
| **viewer** | Solo lectura de datos |

**Implementación con Policies en .NET Core:**

```csharp
// En Startup
services.AddAuthorizationBuilder()
  .AddPolicy("AdminPolicy", policy =>
    policy.RequireRole("superadmin", "tenant_admin"))
  .AddPolicy("ManagerPolicy", policy =>
    policy.RequireRole("manager"))
  .AddPolicy("ViewPolicy", policy =>
    policy.RequireRole("viewer", "user", "manager", "tenant_admin"));

// En Controller
[HttpGet("{id}")]
[Authorize(Policy = "ViewPolicy")]
public async Task<IActionResult> ObtenerPedido(Guid id)
{
  var pedido = await _mediator.Send(new ObtenerPedidoQuery(id));
  return Ok(pedido);
}

[HttpPost]
[Authorize(Policy = "AdminPolicy")]
public async Task<IActionResult> CrearPedido([FromBody] CrearPedidoCommand command)
{
  await _mediator.Send(command);
  return Created();
}
```

---

## 🕵️ Auditoría — Trazabilidad Completa

**Qué se audita:**
- ✅ Quién (usuario + tenant)
- ✅ Qué (acción: crear, actualizar, eliminar)
- ✅ Cuándo (timestamp exacto)
- ✅ Dónde (IP del cliente)
- ✅ Resultado (éxito/fallo)

**Tabla de Auditoría:**

```sql
CREATE TABLE public.audit_log (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  usuario_id UUID,
  modulo VARCHAR(50),
  accion VARCHAR(50),  -- CREATE, UPDATE, DELETE, READ
  tabla_afectada VARCHAR(50),
  registro_id UUID,
  valores_anteriores JSONB,
  valores_nuevos JSONB,
  resultado VARCHAR(50),  -- SUCCESS, FAILURE
  descripcion TEXT,
  ip_origen VARCHAR(50),
  timestamp TIMESTAMP DEFAULT now(),
  FOREIGN KEY (tenant_id) REFERENCES tenants(id),
  FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);
```

**En Serilog (log estructurado):**

```csharp
using (LogContext.PushProperty("TenantId", _currentTenant.TenantId))
using (LogContext.PushProperty("UserId", _currentUser.Id))
using (LogContext.PushProperty("IpOrigen", _httpContext.Connection.RemoteIpAddress))
{
  _logger.Information(
    "Usuario {UserId} creó pedido {PedidoId} con {Items} items en módulo {Modulo}",
    _currentUser.Id,
    pedido.Id,
    pedido.Items.Count(),
    "Ventas"
  );
}
```

---

## 🛡️ Prácticas de Seguridad

### ✅ DO (Hazlo)

- ✅ Validar token en cada request
- ✅ Verificar `TenantId` en token vs. request
- ✅ Usar HTTPS en producción
- ✅ Guardar Refresh Token en HttpOnly cookie
- ✅ Implementar rate limiting
- ✅ Registrar todas las operaciones sensibles

### ❌ DON'T (No lo hagas)

- ❌ Guardar JWT en localStorage (vulnerable a XSS)
- ❌ Olvidar filtrar por `TenantId` en queries
- ❌ Loguear tokens o contraseñas
- ❌ Confiar solo en authorizacion del cliente (siempre validar en backend)
- ❌ Exponer información de error detallada en producción

---

## 🔗 Configuración en Keycloak

### Realm: `neuralnet`

```
Client: neuralops-frontend
├─ Access Type: Public
├─ Client Protocol: openid-connect
├─ Valid Redirect URIs:
│  ├─ http://localhost:4200/*
│  ├─ https://app.neuralops.com/*
├─ Web Origins: +
└─ Standard Flow Enabled: ON

Client: neuralops-api
├─ Access Type: Confidential
├─ Service Accounts Enabled: ON
└─ Roles: admin, user, view_reports
```

### Protocolmapper: Add Custom Claim (`tenant_id`)

En Keycloak:
1. Clients → neuralops-frontend → Mappers → **Add Builtin** → "Add User Attribute"
2. Mappers → **Create**
   - Name: `tenant_id`
   - Mapper Type: `User Attribute`
   - User Attribute: `tenant_id`
   - Token Claim Name: `tenant_id`
   - Add to ID token: ON
   - Add to access token: ON

---

## 📋 Checklist de Seguridad

- [ ] JWT validation en cada endpoint
- [ ] TenantId check en requests
- [ ] Rate limiting configurado
- [ ] Logs auditados en Seq
- [ ] Keycloak con certificados válidos (no self-signed en prod)
- [ ] Secrets guardados en Vault (no en env)
- [ ] CORS solo para dominios confiables
- [ ] HTTPS obligatorio en producción
- [ ] Política de expiración de tokens configurada
- [ ] Refresh token rotación habilitada
