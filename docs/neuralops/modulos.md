# Módulos Empresariales — NeuralOps

NeuralOps está organizado en módulos independientes, cada uno con responsabilidades claras y funcionalidades específicas para diferentes áreas de la empresa.

---

## 📋 Catálogo de Módulos

| Módulo | Código | Descripción |
|--------|--------|-------------|
| **Master** | `master` | Gestión central: tenants, usuarios, roles, configuración |
| **Inventario** | `inv` | Productos, categorías, movimientos de stock |
| **Ventas** | `vnt` | Clientes, pedidos, cotizaciones, facturación |
| **Compras** | `com` | Proveedores, órdenes de compra, recepciones |
| **Finanzas** | `fin` | Contabilidad, transacciones, reportes financieros |
| **RRHH** | `rrhh` | Empleados, nómina, documentos, evaluaciones |
| **Producción** | `prd` | Órdenes de producción, recursos, seguimiento |
| **Seguridad y Salud** | `sst` | Incidents, políticas, reportes de conformidad |
| **Operaciones** | `opr` | Procesos operativos, automatización, monitoreo |
| **Legal** | `leg` | Documentos legales, contratos, cumplimiento |
| **Gerencia** | `ger` | Dashboards ejecutivos, KPIs, análisis |
| **Herramientas** | `her` | Utilidades, integraciones, webhooks |
| **Configuración** | `cfg` | Parámetros del sistema, personalizaciones |

---

## 🔹 Master — Gestión Central

**Responsabilidad:** Foundation del sistema. Gestiona tenants, usuarios y configuración global.

**Funcionalidades:**
- CrearTenant / ModificarTenant / EliminarTenant
- AgregarUsuario / CambiarRol / DesactivarUsuario
- ConfigurarPermisos por rol
- Auditoría global

**Entidades principales:**
- `Tenant` (empresa/organización)
- `Usuario` (con vinculación a Keycloak)
- `Rol` (superadmin, tenant_admin, user)
- `AuditLog` (todas las acciones)

---

## 🔹 Inventario — Gestión de Stock

**Responsabilidad:** Rastrea productos, categorías y movimientos.

**Funcionalidades:**
- CRUD de Productos (código, nombre, categoría, precio, stock)
- Categorización jerárquica
- Movimientos de entrada/salida
- Alertas de bajo stock
- Reportes de rotación

**Entidades principales:**
- `Producto`
- `Categoria`
- `Movimiento` (entrada, salida, ajuste)
- `AlertaStock`

---

## 🔹 Ventas — Orden a Efectivo

**Responsabilidad:** Gestiona todo el ciclo de venta: clientes → pedidos → facturas.

**Funcionalidades:**
- Registro de Clientes (datos de contacto, crédito)
- Cotizaciones (presupuestos)
- Pedidos (con items, descuentos, impuestos)
- Facturas (generadas automáticamente desde pedidos)
- Pagos (seguimiento)

**Entidades principales:**
- `Cliente`
- `Cotizacion`
- `Pedido` + `PedidoItem`
- `Factura`
- `Pago`

**Integraciones:**
- 📊 Kafka: evento "PedidoCreado" → Inventario descuenta stock, IA predice demanda
- 🤖 Python: predicción de tasa de conversión, recomendación de productos

---

## 🔹 Compras — Orden a Recepción

**Responsabilidad:** Gestiona adquisiciones: proveedores → órdenes → recepciones.

**Funcionalidades:**
- Registro de Proveedores (calificación, términos)
- Órdenes de Compra (con ítems, cantidades, precios)
- Recepciones (validación de lo recibido vs. lo esperado)
- Devoluciones
- Reporting de costos

**Entidades principales:**
- `Proveedor`
- `OrdenCompra` + `OrdenCompraItem`
- `Recepcion`
- `DevolucionCompra`

**Integraciones:**
- 📊 Kafka: evento "OrdenCompraCreada" → Auditoría
- 🤖 Python: optimización de proveedores, predicción de costos

---

## 🔹 Finanzas — Contador General

**Responsabilidad:** Registro contable y análisis financiero.

**Funcionalidades:**
- Asientos contables automáticos desde Ventas/Compras
- Conciliación bancaria
- Reportes: Estado de Resultados, Balance, Flujo de Caja
- Análisis de ratios financieros
- Presupuestos vs. Real

**Entidades principales:**
- `Asiento` (con débitos/créditos)
- `Cuenta` (chart of accounts)
- `Transaccion`
- `EstadoFinanciero`

**Integraciones:**
- 🤖 Python: análisis predictivo de cash flow, detección de anomalías

---

## 🔹 RRHH — Gestión de Personal

**Responsabilidad:** Administración de empleados y nómina.

**Funcionalidades:**
- Registro de Empleados (datos personales, contrato, cargo)
- Cálculo de Nómina (salarios, deducciones, beneficios)
- Vacaciones y Ausencias
- Evaluaciones de desempeño
- Documentos (constancia laboral, certificados)

**Entidades principales:**
- `Empleado`
- `Contrato`
- `Nomina` (generada mensualmente)
- `Ausencia`
- `Evaluacion`

---

## 🔹 Producción — Orden a Entrega

**Responsabilidad:** Seguimiento de orden de producción.

**Funcionalidades:**
- Creación de Órdenes de Producción
- Asignación de recursos y máquinas
- Seguimiento en tiempo real
- Captura de cambios en estado
- Reporte de eficiencia

**Entidades principales:**
- `OrdenProduccion`
- `Recurso` (máquinas, materia prima)
- `Operario`
- `RegistroProduccion` (hecho/completado)

---

## 🔹 Seguridad y Salud (SST)

**Responsabilidad:** Cumplimiento normativo en seguridad laboral.

**Funcionalidades:**
- Registro de Incidentes
- Análisis de causas (5 por qué)
- Planes de acción
- Checklists de inspección
- Reportes de conformidad regulatoria

**Entidades principales:**
- `Incidente`
- `ClasificacionRiesgo`
- `PlanAccion`
- `Inspection`

---

## 🔹 Operaciones — Procesos Internos

**Responsabilidad:** Automatización de workflows operativos.

**Funcionalidades:**
- Definición de Procesos (pasos, reglas, escalaciones)
- Ejecución y monitoreo
- Aprobaciones (multi-nivel)
- Alertas de vencimiento
- Dashboard de KPIs operacionales

**Entidades principales:**
- `Proceso`
- `EstadiaProcesal`
- `Aprobacion`
- `AlertaOperacional`

---

## 🔹 Legal — Documentos y Contratos

**Responsabilidad:** Gestión de conformidad y documentación legal.

**Funcionalidades:**
- Registro de Contratos
- Vencimientos y renovaciones
- Almacenamiento de documentos (en MinIO)
- Checklist de cumplimiento
- Auditoría de cambios

**Entidades principales:**
- `Contrato`
- `Documento`
- `ChecklistLegal`

---

## 🔹 Gerencia — Inteligencia de Negocio

**Responsabilidad:** Dashboards y reportes ejecutivos.

**Funcionalidades:**
- KPIs en tiempo real (ventas, márgenes, eficiencia)
- Comparativa: presupuesto vs. real
- Análisis de tendencias
- Alertas de desviaciones
- Export de reportes (PDF, Excel)

**Entidades principales:**
- `KPI`
- `Reporte` (generados bajo demanda)
- `DashboardPersonalizado`

**Integraciones:**
- 🤖 Python: machine learning para forecasting, anomalías

---

## 🔹 Herramientas — Integraciones y Extensiones

**Responsabilidad:** Conectar NeuralOps con sistemas externos.

**Funcionalidades:**
- Webhooks salientes (evento → URL externa)
- Webhooks entrantes (sistema externo → NeuralOps)
- API keys por tenant
- Auditoría de integraciones

**Casos de uso:**
- Enviar pedido a shipping → calcular costo
- Recibir confirmación de pago externo → marcar como pagado
- Notificación de cambio → procesar en sistema legacy

---

## 🔹 Configuración — Personalización por Tenant

**Responsabilidad:** Parámetros y customización específica por empresa.

**Funcionalidades:**
- Parámetros globales (IVA, periodo fiscal, etc.)
- Branding (logo, paleta de colores)
- Flujos de aprobación personalizados
- Campos adicionales (custom fields)
- Idioma y formato regional

**Entidades principales:**
- `ConfiguracionTenant`
- `ParametroSistema`
- `Campo Personalizado`

---

## 🎯 Dependencias entre Módulos

```plaintext
Master (base de todo)
    ├── Inventario
    │   ├── Ventas (despacha desde stock)
    │   ├── Producción (consume materia prima)
    │   └── Compras (repone stock)
    │
    ├── Ventas
    │   ├── Finanzas (genera asientos)
    │   └── Gerencia (reportes de ventas)
    │
    ├── Compras
    │   └── Finanzas (genera asientos)
    │
    ├── Finanzas
    │   └── Gerencia (reportes financieros)
    │
    ├── RRHH
    │   └── Finanzas (nómina → asientos)
    │
    ├── Producción
    │   ├── Inventario (consume y genera)
    │   └── Finanzas (costos)
    │
    ├── Operaciones
    │   └── Todo (orquesta procesos)
    │
    └── Legal, SST, Herramientas, Configuración (transversales)
```

---

## 📊 Orden Recomendado de Implementación

1. **Master** ← foundation
2. **Inventario** ← base de operaciones
3. **Ventas** ← genera ingresos
4. **Compras** ← necesario para inventario
5. **Finanzas** ← cierre de ciclo
6. **RRHH** ← nómina importante
7. **Producción** ← si aplica al negocio
8. **SST** ← cumplimiento normativo
9. **Operaciones** ← automatización continua
10. **Legal** ← contractual
11. **Gerencia** ← reporting
12. **Herramientas** ← integraciones avanzadas
13. **Configuración** ← mejoras continuas
