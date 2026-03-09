# Vault — Configuración

## Habilitar motor de secretos KV

```bash
vault secrets enable -path=neuralnet kv-v2
```

## Guardar un secreto

```bash
vault kv put neuralnet/truckmanager \
  db_password="supersecret" \
  minio_key="AKIAIOSFODNN7EXAMPLE"
```

## Leer un secreto

```bash
vault kv get neuralnet/truckmanager
```

## Crear política de acceso por aplicación

```hcl
# policy-truckmanager.hcl
path "neuralnet/data/truckmanager" {
  capabilities = ["read"]
}
```

```bash
vault policy write truckmanager policy-truckmanager.hcl
```
