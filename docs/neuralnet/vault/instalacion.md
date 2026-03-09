# Vault — Instalación

## Docker Compose

```yaml
services:
  vault:
    image: hashicorp/vault:latest
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: root
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    ports:
      - "8200:8200"
```

!!! warning "Modo dev"
    `VAULT_DEV_ROOT_TOKEN_ID` es solo para desarrollo. En producción usar modo servidor con almacenamiento persistente.

Levantar:

```bash
docker compose up -d
```

Consola: `http://localhost:8200`  
Token inicial: `root`
