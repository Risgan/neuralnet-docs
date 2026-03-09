# MinIO — Instalación

## Docker Compose

```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"   # API S3
      - "9001:9001"   # Consola web
    volumes:
      - minio_data:/data

volumes:
  minio_data:
```

Levantar:

```bash
docker compose up -d
```

- API S3: `http://localhost:9000`
- Consola: `http://localhost:9001`
