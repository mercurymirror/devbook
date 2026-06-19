# Héberger Strapi v5 sur un VPS Linux avec Docker

## Stack

- **OS** : Ubuntu 24.04 LTS
- **Runtime** : Node.js 22 LTS (dans le conteneur Docker)
- **Base de données** : PostgreSQL 17 (dans un conteneur Docker)
- **Orchestration** : Docker + Docker Compose
- **Reverse proxy** : Nginx
- **SSL** : Let's Encrypt (Certbot)
- **CI/CD** : GitHub Actions + GitHub Container Registry (GHCR)

---

## Architecture

```
Internet → Nginx (port 80/443) → Conteneur Strapi (port 1337) → Conteneur PostgreSQL
```

Le build de l'image Docker se fait sur GitHub Actions, pas sur le VPS. Le VPS pull l'image déjà buildée et la lance.

---

## Fichiers à créer dans le projet Strapi

### Dockerfile

```dockerfile
FROM node:22-alpine
RUN apk update && apk add --no-cache build-base gcc autoconf automake zlib-dev libpng-dev bash vips-dev git
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

WORKDIR /opt/
COPY package.json package-lock.json ./
RUN npm install -g node-gyp
RUN npm config set fetch-retry-maxtimeout 600000 -g && npm ci
ENV PATH=/opt/node_modules/.bin:$PATH

WORKDIR /opt/app
COPY . .
RUN npm run build
RUN chown -R node:node /opt/app
USER node
EXPOSE 1337
CMD ["npm", "run", "start"]
```

### docker-compose.yml (développement local)

```yaml
services:
  strapi:
    container_name: strapi
    build: .
    image: strapi:latest
    restart: unless-stopped
    env_file: .env
    ports:
      - "1337:1337"
    networks:
      - strapi
    depends_on:
      strapiDB:
        condition: service_healthy

  strapiDB:
    container_name: strapiDB
    restart: unless-stopped
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: ${DATABASE_USERNAME}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
      POSTGRES_DB: ${DATABASE_NAME}
    volumes:
      - strapi-data:/var/lib/postgresql/data/
    ports:
      - "5432:5432"
    networks:
      - strapi
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DATABASE_USERNAME} -d ${DATABASE_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  strapi-data:

networks:
  strapi:
    name: strapi
    driver: bridge
```

### docker-compose.prod.yml (production)

```yaml
services:
  strapi:
    container_name: strapi
    image: ghcr.io/GITHUB_USERNAME/NOM_DU_REPO:latest
    restart: unless-stopped
    env_file: .env
    ports:
      - "1337:1337"
    networks:
      - strapi
    depends_on:
      strapiDB:
        condition: service_healthy

  strapiDB:
    container_name: strapiDB
    restart: unless-stopped
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: ${DATABASE_USERNAME}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
      POSTGRES_DB: ${DATABASE_NAME}
    volumes:
      - strapi-data:/var/lib/postgresql/data/
    networks:
      - strapi
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DATABASE_USERNAME} -d ${DATABASE_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  strapi-data:

networks:
  strapi:
    name: strapi
    driver: bridge
```

> Remplacer `GITHUB_USERNAME` et `NOM_DU_REPO` par les valeurs réelles.

### .env (variables importantes)

```env
HOST=0.0.0.0
PORT=1337
APP_KEYS=...
API_TOKEN_SALT=...
ADMIN_JWT_SECRET=...
TRANSFER_TOKEN_SALT=...
JWT_SECRET=...

DATABASE_CLIENT=postgres
DATABASE_HOST=strapiDB  # Nom du conteneur PostgreSQL, pas 127.0.0.1
DATABASE_PORT=5432
DATABASE_NAME=strapi_cme
DATABASE_USERNAME=strapi
DATABASE_PASSWORD=mot-de-passe

CLOUDINARY_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...

SMTP_HOST=smtp.mon-provider.com
SMTP_PORT=587
SMTP_USERNAME=adresse@domaine.com
SMTP_PASSWORD=mot-de-passe
SMTP_FROM=adresse@domaine.com
```

> `DATABASE_HOST` doit être le nom du service PostgreSQL dans docker-compose, pas `127.0.0.1`.

---

## Workflow GitHub Actions

```yaml
name: Deploy to VPS

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/GITHUB_USERNAME/NOM_DU_REPO:latest

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd ~/NOM_DU_REPO
            git pull origin main
            docker pull ghcr.io/GITHUB_USERNAME/NOM_DU_REPO:latest
            docker compose -f docker-compose.prod.yml up -d
```

### Secrets GitHub à configurer

Dans le repo GitHub → Settings → Secrets and variables → Actions :

- `VPS_SSH_KEY` : clé SSH privée dédiée au déploiement
- `VPS_HOST` : IP du VPS
- `VPS_USER` : `ubuntu`

> `GITHUB_TOKEN` est automatique — pas besoin de le créer.

---

## 1. Provisionner le VPS

### Connexion SSH

```bash
ssh -i ~/.ssh/ma-cle ubuntu@IP_DU_VPS
```

### Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

### Installer Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo apt install docker-compose-plugin -y
sudo usermod -aG docker ubuntu
newgrp docker
```

Vérifier que Docker démarre au boot :

```bash
sudo systemctl is-enabled docker
```

---

## 2. Déployer Strapi

### Cloner le repo

```bash
git clone https://github.com/GITHUB_USERNAME/NOM_DU_REPO.git
cd NOM_DU_REPO
```

### Configurer le .env sur le VPS

```bash
nano .env
```

> Ne pas oublier `DATABASE_HOST=strapiDB` (nom du conteneur, pas `127.0.0.1`).

### Premier déploiement manuel

```bash
docker compose -f docker-compose.prod.yml up -d
```

---

## 3. Configurer Nginx

```bash
sudo apt install nginx -y
sudo nano /etc/nginx/sites-available/strapi
```

```nginx
server {
    listen 80;
    server_name mon-domaine.com;

    location / {
        proxy_pass http://localhost:1337;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/strapi /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

## 4. Configurer le DNS

Chez le registrar, créer un enregistrement de type `A` :

- Sous-domaine : `strapi` (ou `api`, etc.)
- Cible : IP du VPS
- TTL : valeur par défaut

---

## 5. Configurer le SSL

Ouvrir les ports 80 et 443 dans le firewall du VPS, puis :

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d mon-domaine.com
```

---

## 6. Migrer les données PostgreSQL

Si une base de données existante doit être migrée vers le conteneur Docker :

```bash
# Dump de l'ancienne base
pg_dump -U strapi -h 127.0.0.1 -d strapi_cme -F c -f /tmp/strapi_backup.dump

# Vider la base Docker (Strapi crée ses tables au premier démarrage)
docker exec -i strapiDB psql -U strapi -d strapi_cme -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

# Restaurer le dump dans le conteneur
cat /tmp/strapi_backup.dump | docker exec -i strapiDB pg_restore -U strapi -d strapi_cme -F c
```

---

## Commandes utiles

```bash
# Voir les conteneurs en cours
docker ps

# Logs Strapi
docker logs strapi

# Logs PostgreSQL
docker logs strapiDB

# Redémarrer les conteneurs
docker compose -f docker-compose.prod.yml restart

# Arrêter les conteneurs
docker compose -f docker-compose.prod.yml down

# Vérifier la config Nginx
sudo nginx -t

# Renouveler le certificat SSL manuellement
sudo certbot renew
```
