# Keycloak — Configuración

## Crear un Realm

1. Ingresar al panel de administración
2. Menú superior izquierdo → **Create Realm**
3. Nombre del realm: `neuralnet`
4. Guardar

## Crear un Cliente (por aplicación)

1. Ir a **Clients** → **Create client**
2. `Client ID`: nombre del sistema (ej. `truckmanager`)
3. `Client type`: `OpenID Connect`
4. Activar **Client authentication**
5. `Valid redirect URIs`: URL de la aplicación

## Crear Roles y Usuarios

- **Realm roles**: roles globales del ecosistema
- **Client roles**: roles específicos por aplicación
