# Modelo de Datos — NeuralOps

Estructura de base de datos para NeuralOps. Multi-tenant con aislamiento por `tenant_id`.

---

## 🏗️ Convenciones

- **PKs:** UUID (no INT auto-increment)
- **Tenant:** Toda tabla tiene `tenant_id UUID` + FK a `tenants`
- **Auditoría:** `created_at`, `created_by`, `updated_at`, `updated_by` en tablas críticas
- **Soft Delete:** `deleted_at` para datos que "se borran" lógicamente
- **State:** `estado` VARCHAR para máquinas de estado
- **JSON:** Para configuraciones flexibles

---

## 📋 Tablas Core

### **public.tenants**

```sql
CREATE TABLE public.tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  nombre VARCHAR(150) NOT NULL,
  email_contacto VARCHAR(150),
  telefono VARCHAR(20),
  pais VARCHAR(50),
  estado VARCHAR(20) DEFAULT 'activo',  -- activo, suspendido, inactivo
  configuracion JSONB,  -- logo, colores, parámetros
  fecha_creacion TIMESTAMP DEFAULT now(),
  fecha_activacion TIMESTAMP,
  fecha_cancelacion TIMESTAMP
);
```

### **public.usuarios**

```sql
CREATE TABLE public.usuarios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  keycloak_id UUID UNIQUE NOT NULL,  -- ID en Keycloak
  nombre_completo VARCHAR(150) NOT NULL,
  email VARCHAR(150) NOT NULL,
  telefono VARCHAR(20),
  rol_predeterminado VARCHAR(50),  -- admin, manager, user, viewer
  activo BOOLEAN DEFAULT true,
  ultimo_acceso TIMESTAMP,
  fecha_creacion TIMESTAMP DEFAULT now(),
  UNIQUE(tenant_id, email)
);
CREATE INDEX idx_usuarios_tenant_id ON public.usuarios(tenant_id);
```

### **public.roles**

```sql
CREATE TABLE public.roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  nombre VARCHAR(50) NOT NULL,
  descripcion TEXT,
  permisos JSONB,  -- { "modulos": ["vnt", "inv"], "acciones": ["read", "write"] }
  fecha_creacion TIMESTAMP DEFAULT now(),
  UNIQUE(tenant_id, nombre)
);
```

---

## 🛍️ Inventario (`inv`).

### **inv.productos**

```sql
CREATE TABLE inv.productos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  codigo VARCHAR(50) NOT NULL,
  nombre VARCHAR(150) NOT NULL,
  descripcion TEXT,
  categoria_id UUID REFERENCES inv.categorias(id),
  precio_compra DECIMAL(12,2),
  precio_venta DECIMAL(12,2),
  stock_actual INT DEFAULT 0,
  stock_minimo INT DEFAULT 10,
  stock_maximo INT DEFAULT 1000,
  unidad_medida VARCHAR(20),  -- pz, kg, l, etc.
  activo BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now(),
  UNIQUE(tenant_id, codigo)
);
CREATE INDEX idx_productos_tenant_id ON inv.productos(tenant_id);
CREATE INDEX idx_productos_categoria_id ON inv.productos(categoria_id);
```

### **inv.categorias**

```sql
CREATE TABLE inv.categorias (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  nombre VARCHAR(100) NOT NULL,
  descripcion TEXT,
  padre_id UUID REFERENCES inv.categorias(id),  -- N-ésimo nivel
  activo BOOLEAN DEFAULT true,
  UNIQUE(tenant_id, nombre)
);
```

### **inv.movimientos**

```sql
CREATE TABLE inv.movimientos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  producto_id UUID NOT NULL REFERENCES inv.productos(id),
  tipo_movimiento VARCHAR(20),  -- ENTRADA, SALIDA, AJUSTE, DEVOLUCIÓN
  cantidad INT NOT NULL,
  modulo_origen VARCHAR(20),  -- vnt (ventas), com (compras), prd (producción)
  referencia_id UUID,  -- ID del pedido/orden que genera este movimiento
  motivo TEXT,
  usuario_id UUID REFERENCES public.usuarios(id),
  stock_anterior INT,
  stock_posterior INT,
  fecha_movimiento TIMESTAMP DEFAULT now()
);
CREATE INDEX idx_movimientos_tenant_id ON inv.movimientos(tenant_id);
CREATE INDEX idx_movimientos_producto_id ON inv.movimientos(producto_id);
```

---

## 🛒 Ventas (`vnt`)

### **vnt.clientes**

```sql
CREATE TABLE vnt.clientes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  nombre VARCHAR(150) NOT NULL,
  email VARCHAR(150),
  telefono VARCHAR(20),
  documento VARCHAR(20),  -- RUC, DNI, etc.
  direccion TEXT,
  ciudad VARCHAR(100),
  pais VARCHAR(50),
  limite_credito DECIMAL(12,2) DEFAULT 0,
  cliente_tipo VARCHAR(20),  -- persona, empresa
  activo BOOLEAN DEFAULT true,
  fecha_creacion TIMESTAMP DEFAULT now(),
  UNIQUE(tenant_id, documento)
);
```

### **vnt.pedidos**

```sql
CREATE TABLE vnt.pedidos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  cliente_id UUID NOT NULL REFERENCES vnt.clientes(id),
  numero_pedido VARCHAR(20) NOT NULL,
  fecha_pedido TIMESTAMP DEFAULT now(),
  fecha_entrega_esperada TIMESTAMP,
  estado VARCHAR(20) DEFAULT 'pendiente',  -- pendiente, procesando, completado, cancelado
  
  subtotal DECIMAL(12,2),
  descuento DECIMAL(12,2) DEFAULT 0,
  impuesto DECIMAL(12,2),
  total DECIMAL(12,2),
  
  notas TEXT,
  usuario_creador_id UUID REFERENCES public.usuarios(id),
  usuario_aprobador_id UUID REFERENCES public.usuarios(id),
  fecha_aprobacion TIMESTAMP,
  
  UNIQUE(tenant_id, numero_pedido)
);
```

### **vnt.pedido_items**

```sql
CREATE TABLE vnt.pedido_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pedido_id UUID NOT NULL REFERENCES vnt.pedidos(id) ON DELETE CASCADE,
  producto_id UUID NOT NULL REFERENCES inv.productos(id),
  cantidad INT NOT NULL,
  precio_unitario DECIMAL(12,2),
  descuento_item DECIMAL(12,2) DEFAULT 0,
  subtotal DECIMAL(12,2),
  secuencia INT
);
```

### **vnt.facturas**

```sql
CREATE TABLE vnt.facturas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  pedido_id UUID REFERENCES vnt.pedidos(id),
  numero_factura VARCHAR(20) NOT NULL,
  fecha_emision TIMESTAMP DEFAULT now(),
  fecha_vencimiento TIMESTAMP,
  
  subtotal DECIMAL(12,2),
  impuesto DECIMAL(12,2),
  total DECIMAL(12,2),
  
  estado VARCHAR(20),  -- emitida, pagada, anulada
  medio_pago VARCHAR(50),  -- efectivo, tarjeta, transferencia
  
  UNIQUE(tenant_id, numero_factura)
);
```

---

## 🏭 Compras (`com`)

### **com.proveedores**

```sql
CREATE TABLE com.proveedores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  nombre VARCHAR(150) NOT NULL,
  email VARCHAR(150),
  telefono VARCHAR(20),
  contacto_nombre VARCHAR(100),
  direccion TEXT,
  
  calificacion DECIMAL(3,2),  -- 0 a 5 estrellas
  dias_pago INT DEFAULT 30,  -- plazo promedio
  
  activo BOOLEAN DEFAULT true,
  fecha_creacion TIMESTAMP DEFAULT now()
);
```

### **com.ordenes_compra**

```sql
CREATE TABLE com.ordenes_compra (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  proveedor_id UUID NOT NULL REFERENCES com.proveedores(id),
  numero_oc VARCHAR(20) NOT NULL,
  fecha_orden TIMESTAMP DEFAULT now(),
  fecha_entrega_esperada TIMESTAMP,
  
  estado VARCHAR(20) DEFAULT 'pendiente',  -- pendiente, enviada, recibida, cancelada
  
  subtotal DECIMAL(12,2),
  impuesto DECIMAL(12,2),
  total DECIMAL(12,2),
  
  UNIQUE(tenant_id, numero_oc)
);
```

---

## 💰 Finanzas (`fin`)

### **fin.asientos**

```sql
CREATE TABLE fin.asientos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  numero_asiento VARCHAR(20),
  fecha_asiento TIMESTAMP NOT NULL,
  descripcion TEXT,
  total_debito DECIMAL(12,2),
  total_credito DECIMAL(12,2),
  estado VARCHAR(20),  -- pendiente, validado, anulado
  
  -- Trazabilidad
  modulo_origen VARCHAR(50),  -- vnt, com, rrhh, etc.
  referencia_id UUID,  -- ID del documento que genera el asiento
  
  UNIQUE(tenant_id, numero_asiento)
);
```

### **fin.lineas_asiento**

```sql
CREATE TABLE fin.lineas_asiento (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  asiento_id UUID NOT NULL REFERENCES fin.asientos(id) ON DELETE CASCADE,
  cuenta_id UUID NOT NULL REFERENCES fin.cuentas(id),
  tipo_linea VARCHAR(20),  -- DEBITO, CREDITO
  monto DECIMAL(12,2),
  descripcion TEXT,
  secuencia INT
);
```

### **fin.cuentas (Chart of Accounts)**

```sql
CREATE TABLE fin.cuentas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  codigo_cuenta VARCHAR(20) NOT NULL,
  nombre VARCHAR(150) NOT NULL,
  tipo_cuenta VARCHAR(50),  -- ACTIVO, PASIVO, PATRIMONIO, INGRESO, GASTO
  cuenta_padre_id UUID REFERENCES fin.cuentas(id),  -- Jerarquía
  
  activo BOOLEAN DEFAULT true,
  UNIQUE(tenant_id, codigo_cuenta)
);
```

---

## 👥 RRHH (`rrhh`)

### **rrhh.empleados**

```sql
CREATE TABLE rrhh.empleados (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  usuario_id UUID REFERENCES public.usuarios(id),
  numero_documento VARCHAR(20) NOT NULL,
  nombre_completo VARCHAR(150) NOT NULL,
  email_corporativo VARCHAR(150),
  telefono VARCHAR(20),
  
  cargo VARCHAR(100),
  departamento VARCHAR(100),
  fecha_ingreso DATE NOT NULL,
  fecha_egreso DATE,
  
  salario_base DECIMAL(12,2),
  tipo_contrato VARCHAR(20),  -- indefinido, plazo fijo, temporal
  
  estado VARCHAR(20),  -- activo, licencia, retirado
  UNIQUE(tenant_id, numero_documento)
);
```

### **rrhh.nomina**

```sql
CREATE TABLE rrhh.nomina (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  empleado_id UUID NOT NULL REFERENCES rrhh.empleados(id),
  periodo VARCHAR(20) NOT NULL,  -- 2025-03
  
  salario_neto DECIMAL(12,2),
  total_deducciones DECIMAL(12,2),
  total_bonificaciones DECIMAL(12,2),
  
  estado VARCHAR(20),  -- borradores, aprobada, pagada
  fecha_creacion TIMESTAMP DEFAULT now(),
  UNIQUE(tenant_id, empleado_id, periodo)
);
```

---

## 🏭 Producción (`prd`)

### **prd.ordenes_produccion**

```sql
CREATE TABLE prd.ordenes_produccion (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  numero_op VARCHAR(20) NOT NULL,
  
  producto_final_id UUID NOT NULL REFERENCES inv.productos(id),
  cantidad_a_producir INT NOT NULL,
  
  fecha_inicio TIMESTAMP DEFAULT now(),
  fecha_finalizacion_esperada TIMESTAMP,
  
  estado VARCHAR(20),  -- creada, en_progreso, completada, cancelada
  
  UNIQUE(tenant_id, numero_op)
);
```

---

## 🔍 Tablas de Auditoría

### **public.audit_log** (Central)

```sql
CREATE TABLE public.audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES public.tenants(id),
  usuario_id UUID REFERENCES public.usuarios(id),
  
  modulo VARCHAR(50),  -- vnt, inv, com, rrhh, etc.
  tabla_afectada VARCHAR(50),
  accion VARCHAR(20),  -- CREATE, UPDATE, DELETE, READ (si es crítica)
  
  registro_id UUID,  -- PK del registro modificado
  valores_anteriores JSONB,
  valores_nuevos JSONB,
  
  resultado VARCHAR(20),  -- SUCCESS, FAILURE
  descripcion_error TEXT,
  
  ip_origen VARCHAR(50),
  user_agent TEXT,
  
  timestamp_evento TIMESTAMP DEFAULT now(),
  
  INDEX (tenant_id, timestamp_evento)
);
```

---

## 🔗 Relaciones Clave

```plaintext
tenants (root)
    ├── usuarios
    ├── roles
    │
    ├── inv.productos ← inv.categorias
    ├── inv.movimientos
    │
    ├── vnt.clientes
    ├── vnt.pedidos ← vnt.pedido_items ← inv.productos
    ├── vnt.facturas ← vnt.pedidos
    │
    ├── com.proveedores
    ├── com.ordenes_compra ← com.ordenes_compra_items ← inv.productos
    │
    ├── fin.asientos ← fin.lineas_asiento ← fin.cuentas
    │
    ├── rrhh.empleados ← rrhh.nomina
    ├── prd.ordenes_produccion
    │
    └── audit_log
```

---

## 📊 Índices Recomendados

```sql
-- Performance critical
CREATE INDEX idx_usuarios_tenant ON public.usuarios(tenant_id);
CREATE INDEX idx_audit_tenant_timestamp ON public.audit_log(tenant_id, timestamp_evento);
CREATE INDEX idx_pedidos_tenant_estado ON vnt.pedidos(tenant_id, estado);
CREATE INDEX idx_productos_tenant_categoria ON inv.productos(tenant_id, categoria_id);
CREATE INDEX idx_factura_tenant_fecha ON vnt.facturas(tenant_id, fecha_emision);
CREATE INDEX idx_asientos_tenant_fecha ON fin.asientos(tenant_id, fecha_asiento);
```

---

## 🔐 Particionamiento Futuro (Escalabilidad)

Si creces a millones de datos por tenant, considera partir tablas por `tenant_id`:

```sql
-- Ej: Tabla vnt.pedidos particionada por tenant
CREATE TABLE vnt.pedidos (
  id UUID,
  tenant_id UUID,
  ...
) PARTITION BY LIST (tenant_id);

CREATE TABLE vnt.pedidos_tenant_abc PARTITION OF vnt.pedidos
  FOR VALUES IN ('tenant-uuid-abc');
```

Esto evita scans en toda la tabla cuando consultas por un tenant específico.
