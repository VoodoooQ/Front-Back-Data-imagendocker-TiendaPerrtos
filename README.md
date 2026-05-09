# Tienda Perritos - Monorepo con Docker Hub

## Arquitectura monorepo
Este proyecto usa arquitectura **monorepo** y mantiene los tres componentes separados por carpeta:
- `frontend/`
- `backend/`
- `db/`

Los workflows de CI están en `.github/workflows/` y cada uno construye/publica solo su componente cuando hay cambios en su carpeta.

## Estructura del repositorio
```text
Front-Back-Data-imagendocker-TiendaPerrtos/
├── frontend/
├── backend/
├── db/
├── .github/
│   └── workflows/
│       ├── dockerhub-frontend.yml
│       ├── dockerhub-backend.yml
│       └── dockerhub-db.yml
├── docker-compose.yml
└── .env.example
```

## Ejecución local con Docker Compose
`docker-compose.yml` está pensado para desarrollo local y levanta:
- `frontend`
- `backend`
- `db`
- red interna (`tienda-net`)
- volumen persistente de MySQL (`dbdata`)

Además, las variables se leen desde `.env`.

```bash
cp .env.example .env
docker compose up --build
```

## CI/CD simplificado
1. Push a rama deploy.
2. GitHub Actions detecta cambios por carpeta.
3. Construye la imagen Docker correspondiente.
4. Publica la imagen en Docker Hub.
5. El traslado y despliegue en EC2 se realiza manualmente por seguridad de la arquitectura.

## Imágenes Docker Hub
- DOCKERHUB_USERNAME/tienda-frontend:latest
- DOCKERHUB_USERNAME/tienda-backend:latest
- DOCKERHUB_USERNAME/tienda-db:latest

Cada workflow también publica un tag adicional con el commit:
- `DOCKERHUB_USERNAME/tienda-frontend:${github.sha}`
- `DOCKERHUB_USERNAME/tienda-backend:${github.sha}`
- `DOCKERHUB_USERNAME/tienda-db:${github.sha}`

## Despliegue manual en EC2
No hay despliegue automático desde GitHub Actions hacia EC2.

La arquitectura considera Front con acceso a internet y nodos Back/Data en subred privada sin internet directo.  
Por eso las imágenes se mueven manualmente desde Front hacia Back/Data usando:
- `docker pull`
- `docker save`
- `scp`
- `docker load`
- `docker run`

Desde Front:

```bash
docker pull USUARIO_DOCKERHUB/tienda-backend:latest
docker save USUARIO_DOCKERHUB/tienda-backend:latest -o tienda-backend.tar
scp -i ~/.ssh/LlavesEv2.pem tienda-backend.tar ubuntu@IP_PRIVADA_BACK:/home/ubuntu/
```

En Back:

```bash
docker load -i tienda-backend.tar
docker run -d --name tienda-backend --restart unless-stopped --env-file /home/ubuntu/.env -p 3001:3001 USUARIO_DOCKERHUB/tienda-backend:latest
```

Desde Front hacia Data:

```bash
docker pull USUARIO_DOCKERHUB/tienda-db:latest
docker save USUARIO_DOCKERHUB/tienda-db:latest -o tienda-db.tar
scp -i ~/.ssh/LlavesEv2.pem tienda-db.tar ubuntu@IP_PRIVADA_DATA:/home/ubuntu/
```

En Data:

```bash
docker load -i tienda-db.tar
docker run -d --name tienda-db --restart unless-stopped --env-file /home/ubuntu/.env -p 3306:3306 -v dbdata:/var/lib/mysql USUARIO_DOCKERHUB/tienda-db:latest
```

## Secrets en GitHub Actions
Solo se requieren estos secrets:
- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
