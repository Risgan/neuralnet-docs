# Roadmap de Implementación — NeuralOps

Orden paso a paso para levantar NeuralOps desde cero.

---

## 📍 Fases de Implementación

### **FASE 0: Infraestructura Base** (Semana 1)

El cimiento sobre el que corre todo. Docker Compose con los servicios principales.

#### Objetivo
Tener todos los servicios corriendo localmente sin que algo dependa de otros.

#### 1️⃣ Dockerfile compose inicial

Archivo: [docker-compose.yml](../../../docker-compose.yml)

```yaml
version: '3.8'

services:
  # 🗄️ Base de datos
  postgresql:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: neuralops
      POSTGRES_USER: neuralops
      POSTGRES_PASSWORD: neuralops123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - neuralnet

  # 🖧 Admin de DB (opcional)
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@neuralops.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    networks:
      - neuralnet

  # 🔐 Keycloak
  keycloak:
    image: keycloak/keycloak:24.0
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
    networks:
      - neuralnet

  # 📩 Kafka
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    networks:
      - neuralnet

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - neuralnet

  # 📊 Seq (Logging)
  seq:
    image: datalust/seq:latest
    environment:
      ACCEPT_EULA: Y
    ports:
      - "5341:80"
    networks:
      - neuralnet

  # 📦 MinIO
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    networks:
      - neuralnet

  # 🔴 Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - neuralnet

volumes:
  postgres_data:
  minio_data:

networks:
  neuralnet:
    driver: bridge
```

#### Levantar

```bash
docker compose up -d
```

**URLs de acceso:**
- PostgreSQL: `localhost:5432`
- PgAdmin: `http://localhost:5050`
- Keycloak: `http://localhost:8080`
- Seq: `http://localhost:5341`
- MinIO API: `http://localhost:9000`
- MinIO Console: `http://localhost:9001`
- Redis: `localhost:6379`
- Kafka: `localhost:9092`

✅ **Señal de éxito:** Todos los servicios healthy en `docker ps`

---

### **FASE 1: Backend .NET Core** (Semana 2-3)

API REST con autenticación y base de datos.

#### Objetivo
API respondiendo en `http://localhost:6000/api/*` con autenticación JWT de Keycloak.

#### 1️⃣ Crear proyecto .NET

```bash
dotnet new globaljson --sdk-version 8.0.0
dotnet new sln -n NeuralOps
dotnet new webapi -n NeuralOps.API
cd NeuralOps.API
dotnet add package Microsoft.EntityFrameworkCore.PostgreSQL
dotnet add package MediatR
dotnet add package mapping AutoMapper
dotnet add package FluentValidation
dotnet add package Serilog.AspNetCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

#### 2️⃣ Configurar DbContext (Entity Framework)

**Data/NeuralOpsContext.cs**

```csharp
public class NeuralOpsContext : DbContext
{
  public NeuralOpsContext(DbContextOptions<NeuralOpsContext> options) : base(options) { }

  public DbSet<Tenant> Tenants { get; set; }
  public DbSet<Usuario> Usuarios { get; set; }
  public DbSet<AuditLog> AuditLogs { get; set; }

  protected override void OnModelCreating(ModelBuilder modelBuilder)
  {
    base.OnModelCreating(modelBuilder);

    // Índices para multi-tenancy
    modelBuilder.Entity<Usuario>()
      .HasIndex(u => new { u.TenantId, u.Email })
      .IsUnique();

    modelBuilder.Entity<AuditLog>()
      .HasIndex(a => new { a.TenantId, a.Timestamp });
  }
}
```

**Domain/Entities/Tenant.cs**

```csharp
public class Tenant
{
  public Guid Id { get; set; }
  public string Nombre { get; set; }
  public string Email { get; set; }
  public DateTime FechaCreacion { get; set; }
}
```

**Domain/Entities/Usuario.cs**

```csharp
public class Usuario
{
  public Guid Id { get; set; }
  public Guid TenantId { get; set; }
  public string NombreCompleto { get; set; }
  public string Email { get; set; }
  public Guid KeycloakId { get; set; }
  public DateTime FechaCreacion { get; set; }
}
```

#### 3️⃣ Configurar autenticación con Keycloak

**Program.cs**

```csharp
var builder = WebApplicationBuilder.CreateBuilder(args);

// DbContext
builder.Services.AddDbContext<NeuralOpsContext>(options =>
  options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Keycloak
builder.Services.AddAuthentication(options =>
{
  options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
  options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
  options.Authority = "http://localhost:8080/realms/neuralnet";
  options.Audience = "neuralops-api";
  options.RequireHttpsMetadata = false;
});

builder.Services.AddAuthorization();

// Swagger
builder.Services.AddSwaggerGen(o => o.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
{
  Type = SecuritySchemeType.OAuth2,
  Flows = new OpenApiOAuthFlows
  {
    Implicit = new OpenApiOAuthFlow
    {
      AuthorizationUrl = new Uri("http://localhost:8080/realms/neuralnet/protocol/openid-connect/auth"),
      TokenUrl = new Uri("http://localhost:8080/realms/neuralnet/protocol/openid-connect/token")
    }
  }
}));

// Mediator (CQRS)
builder.Services.AddMediatR(config => config.RegisterServicesFromAssemblies(typeof(Program).Assembly));

// AutoMapper
builder.Services.AddAutoMapper(AppDomain.CurrentDomain.GetAssemblies());

// Serilog
builder.Host.UseSerilog((context, config) =>
{
  config.WriteTo.Console()
        .WriteTo.Seq("http://localhost:5341");
});

var app = builder.Build();

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.UseSwagger();
app.UseSwaggerUI();

app.Run("http://localhost:6000");
```

#### 4️⃣ Crear un endpoint de prueba

**Controllers/TenantController.cs**

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class TenantController : ControllerBase
{
  private readonly IMediator _mediator;

  public TenantController(IMediator mediator)
  {
    _mediator = mediator;
  }

  [HttpGet("{id}")]
  [AllowAnonymous]
  public async Task<IActionResult> GetTenant(Guid id)
  {
    var query = new ObtenerTenantQuery(id);
    var result = await _mediator.Send(query);
    return Ok(result);
  }
}
```

**Application/Queries/ObtenerTenantQuery.cs**

```csharp
public record ObtenerTenantQuery(Guid Id) : IRequest<TenantDTO>;

public class ObtenerTenantQueryHandler : IRequestHandler<ObtenerTenantQuery, TenantDTO>
{
  private readonly NeuralOpsContext _context;
  private readonly IMapper _mapper;

  public async Task<TenantDTO> Handle(ObtenerTenantQuery request, CancellationToken cancellationToken)
  {
    var tenant = await _context.Tenants.FirstOrDefaultAsync(t => t.Id == request.Id, cancellationToken);
    if (tenant == null)
      throw new KeyNotFoundException($"Tenant {request.Id} no encontrado");
    
    return _mapper.Map<TenantDTO>(tenant);
  }
}
```

#### 5️⃣ Correr migraciones

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

#### ✅ Resultado

```bash
dotnet run
```

Acceder a `http://localhost:6000/swagger` y probar `GET /api/tenant/{guid}`

---

### **FASE 2: Frontend Angular** (Semana 4-5)

Interface web con autenticación y dashboard.

#### Objetivo
Login funcional, usuario logueado, dashboard simple.

#### 1️⃣ Crear proyecto Angular

```bash
ng new neuralops-web
cd neuralops-web
npm install keycloak-js @keycloak/keycloak-angular @angular/material
```

#### 2️⃣ Configurar Keycloak

**app.config.ts**

```typescript
export function initializeKeycloak(keycloak: KeycloakService): Promise<boolean> {
  return keycloak.init({
    config: {
      url: 'http://localhost:8080',
      realm: 'neuralnet',
      clientId: 'neuralops-frontend',
    },
    initOptions: {
      onLoad: 'login-required',
      silentCheckSsoRedirectUri: `${window.location.origin}/silent-check-sso.html`
    }
  });
}

export const provideKeycloak = (): Provider => ({
  provide: APP_INITIALIZER,
  useFactory: initializeKeycloak,
  deps: [KeycloakService],
  multi: true
});
```

**main.ts**

```typescript
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideAnimations(),
    provideKeycloak()
  ]
}).catch(err => console.error(err));
```

#### 3️⃣ Crear páginas básicas

**login.component.ts**

```typescript
@Component({
  selector: 'app-login',
  template: `
    <div class="login-container">
      <h1>NeuralOps</h1>
      <button (click)="login()">Iniciar Sesión</button>
    </div>
  `
})
export class LoginComponent {
  constructor(private keycloak: KeycloakService) {}

  login() {
    this.keycloak.login();
  }
}
```

**dashboard.component.ts**

```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <h1>Bienvenido {{ (userName$ | async) }}</h1>
      <div class="cards">
        <mat-card>
          <mat-card-title>Ventas del mes</mat-card-title>
          <mat-card-content>$0</mat-card-content>
        </mat-card>
        <mat-card>
          <mat-card-title>Órdenes pendientes</mat-card-title>
          <mat-card-content>0</mat-card-content>
        </mat-card>
      </div>
    </div>
  `
})
export class DashboardComponent implements OnInit {
  userName$: Observable<string>;

  constructor(
    private keycloak: KeycloakService,
    private http: HttpClient
  ) {
    this.userName$ = of(this.keycloak.getUsername());
  }

  ngOnInit(): void {
    this.cargarDatos();
  }

  cargarDatos() {
    this.http.get('/api/tenant/me').subscribe();
  }
}
```

#### 4️⃣ Routing

**app.routes.ts**

```typescript
export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [AuthGuard]
  },
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' }
];
```

#### ✅ Resultado

```bash
ng serve
```

Acceder a `http://localhost:4200`  
→ Redirige a Keycloak  
→ Login  
→ Dashboard con nombre del usuario

---

### **FASE 3: Módulos de Negocio** (Semana 6+)

Implementar uno por uno.

#### 3.1 Inventario (Módulo base)

**Backend:**
1. Crear DB tables: `productos`, `categorias`, `movimientos`
2. Endpoints CRUD básicos
3. Kafka: evento al agregar producto

**Frontend:**
1. Tabla de productos
2. Formulario crear/editar
3. Listado de categorías

#### 3.2 Ventas (depende de Inventario)

**Backend:**
1. DB tables: `clientes`, `pedidos`, `pedido_items`, `facturas`
2. Crear pedido → descuentacon automático de inventario
3. Kafka: evento "PedidoCreado" → IA predice demanda

**Frontend:**
1. Crear pedido (con búsqueda de productos)
2. Facturas generadas automáticamente

#### 3.3 Compras (depende de Inventario)

**Backend:**
1. DB tables: `proveedores`, `ordenes_compra`
2. Recibir mercancía → suma stock

**Frontend:**
1. Órdenes de compra
2. Recepciones

#### 3.4 Finanzas (depende de Ventas + Compras)

**Backend:**
1. Asientos contables automáticos desde Ventas/Compras
2. Reportes financieros

#### Y así...

---

## 📅 Timeline Estimado

| Fase | Semanas | Entrega |
|---|---|---|
| **Fase 0** | 1 | Docker compose corriendo |
| **Fase 1** | 2-3 | API con autenticacion |
| **Fase 2** | 2-3 | Angular login + dashboard |
| **Fase 3.1** | 2 | Módulo Inventario |
| **Fase 3.2** | 2 | Módulo Ventas |
| **Fase 3.3** | 1-2 | Módulo Compras |
| **Fase 3.4** | 1-2 | Módulo Finanzas |
| **Testing + Deploy** | Variable | Producción |
| **Total mínimo** | **13-16 semanas** | MVP multi-tenant funcional |

---

## ✅ Checklist de Fases

### Fase 0 ✓
- [ ] Docker compose levanta sin errores
- [ ] Todos los servicios healthy
- [ ] Acceso a pgAdmin, Keycloak, Seq, MinIO

### Fase 1 ✓
- [ ] API con autenticación JWT
- [ ] Swagger respondiendo
- [ ] Endpoint de prueba devuelve datos
- [ ] Base de datos con tenants + usuarios

### Fase 2 ✓
- [ ] Angular compila sin errores
- [ ] Login funciona en Keycloak
- [ ] Dashboard visible después de login
- [ ] HttpInterceptor agrega token JWT

### Fase 3.1+ ✓
- [ ] CRUD de productos completo
- [ ] Stock se descuenta al crear pedido
- [ ] Reportes básicos de ventas
- [ ] Auditoría en Seq

---

## 🚀 Próximos Pasos

1. **Ejecutar FASE 0** inmediatamente
2. **Crear repositorio Git** con estructura clara
3. **Asignar desarrolladores:**
   - 1x Backend .NET (lidera)
   - 2x Frontend Angular (UI)
   - 1x DevOps (Docker/Prod)
4. **Daily standups** para sincronizar avance
5. **Testing desde el inicio** (no al final)
