# Runbook — Déploiement Strapi v5 sur VPS Infomaniak

## Contexte

- **Projet** : Classe moyenne éditions
- **CMS** : Strapi v5 (migré depuis v3)
- **Hébergeur** : Infomaniak VPS Lite
- **Domaine Strapi** : strapi.cmeditions.fr
- **Médias** : Cloudinary (pas de gestion locale des fichiers)
- **Base de données** : PostgreSQL

---

## Prérequis locaux

- Node.js via nvm
- Git + GitHub CLI (`gh`)
- Heroku CLI (pour dump de la base source)
- PostgreSQL 17 en local (pour restaurer le dump)

---

## 1. Récupérer la base de données source (Heroku)

```bash
heroku pg:backups:capture --app nom-de-lapp
heroku pg:backups:download --app nom-de-lapp
# Génère un fichier latest.dump

createdb nom-db-locale
pg_restore --no-owner --no-privileges -d nom-db-locale latest.dump
```

> Si erreur de version pg_restore : installer PostgreSQL 17 via Homebrew et l'ajouter au PATH.

---

## 2. Migration Strapi v3 → v5

Effectuée via Claude Code. Résultat : une app Strapi v5 fraîche avec les données migrées.

### Transfer des données (v5 local → v5 VPS)

```bash
# Depuis le dossier du projet Strapi v5 local
npm run strapi transfer -- --to https://strapi.cmeditions.fr/admin
```

> Générer un token de transfer dans l'admin Strapi du VPS avant de lancer.
> Les assets échoueront (Cloudinary gère les médias) — les entités passeront.

---

## 3. Migration API REST → GraphQL (frontend Next.js)

### Installer le plugin GraphQL sur Strapi v5

```bash
npm install @strapi/plugin-graphql
```

Déclarer dans `config/plugins.ts` :

```typescript
graphql: {
  enabled: true,
}
```

### Réécrire utils/api.js

Remplacer les appels REST par GraphQL via une fonction `fetchGraphQL` :

```javascript
const GRAPHQL_URL = `${process.env.NEXT_PUBLIC_STRAPI_API_URL}/graphql`

async function fetchGraphQL(query, variables = {}) {
  const response = await fetch(GRAPHQL_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query, variables }),
  })
  const data = await response.json()
  return data.data
}
```

> En Strapi v5, pas de `data.attributes` — les champs sont directement à plat.
> Utiliser `documentId` à la place de `id` numérique.
> Ajouter `pagination: { pageSize: 100 }` pour éviter la limite de 10 résultats par défaut.

---

## 4. Provisionner le VPS

### Connexion SSH

```bash
ssh -i ~/.ssh/id_infomaniak ubuntu@IP_DU_VPS
```

### Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

### Installer nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/$(curl -s https://api.github.com/repos/nvm-sh/nvm/releases/latest | grep '"tag_name"' | cut -d'"' -f4)/install.sh | bash
source ~/.bashrc
nvm --version
```

### Installer Node.js 22 LTS

```bash
nvm install 22
nvm alias default 22
node --version
```

### Installer PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib -y
sudo systemctl status postgresql
```

### Créer la base de données et l'utilisateur

```bash
sudo -u postgres psql
```

```sql
CREATE USER strapi WITH PASSWORD 'mot-de-passe-solide';
CREATE DATABASE strapi_cme OWNER strapi;
\q
```

> Eviter les caractères spéciaux dans le mot de passe (problèmes d'échappement dans le .env).

### Installer PM2

```bash
npm install -g pm2
```

### Installer Nginx

```bash
sudo apt install nginx -y
sudo systemctl status nginx
```

---

## 5. Déployer Strapi sur le VPS

### Pousser le code sur GitHub

```bash
git remote set-url origin https://github.com/user/repo.git
git push -u origin main
```

### Cloner sur le VPS

```bash
git clone https://github.com/user/repo.git
cd nom-du-repo
npm install
```

### Configurer le .env

```bash
nano .env
```

```env
HOST=0.0.0.0
PORT=1337
APP_KEYS=...
API_TOKEN_SALT=...
ADMIN_JWT_SECRET=...
TRANSFER_TOKEN_SALT=...
JWT_SECRET=...

DATABASE_CLIENT=postgres
DATABASE_HOST=127.0.0.1
DATABASE_PORT=5432
DATABASE_NAME=strapi_cme
DATABASE_USERNAME=strapi
DATABASE_PASSWORD=mot-de-passe-solide

CLOUDINARY_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...

SMTP_HOST=pro1.mail.ovh.net
SMTP_PORT=587
SMTP_USERNAME=adresse@domaine.fr
SMTP_PASSWORD=mot-de-passe-email
SMTP_FROM=adresse@domaine.fr
```

### Builder Strapi

```bash
NODE_ENV=production NODE_OPTIONS="--max-old-space-size=1536" npm run build
```

> Le build peut prendre 5-10 minutes sur un VPS avec 2 Go de RAM.
> Si erreur out of memory : ajouter un swap de 2 Go (voir section Swap).

### Démarrer avec PM2

```bash
NODE_ENV=production pm2 start npm --name "strapi-cme" -- start
pm2 startup
pm2 save
```

---

## 6. Ajouter un swap (si out of memory au build)

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Pour rendre le swap permanent au reboot :

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 7. Configurer Nginx

### Créer la configuration

```bash
sudo nano /etc/nginx/sites-available/strapi-cme
```

```nginx
server {
    listen 80;
    server_name strapi.cmeditions.fr;

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

### Activer la configuration

```bash
sudo ln -s /etc/nginx/sites-available/strapi-cme /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

## 8. Configurer le DNS

Dans l'interface OVH, ajouter un enregistrement DNS de type `A` :

- Sous-domaine : `strapi`
- Cible : IP du VPS
- TTL : valeur par défaut

---

## 9. Configurer le SSL (Let's Encrypt)

### Ouvrir les ports 80 et 443 dans le firewall Infomaniak

Dans le manager Infomaniak → Firewall du VPS → ajouter les ports 80 et 443.

### Installer Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Générer le certificat

```bash
sudo certbot --nginx -d strapi.cmeditions.fr
```

> Certbot met à jour automatiquement la config Nginx pour rediriger HTTP → HTTPS.

---

## 10. Configurer l'email (SMTP OVH)

### Installer le provider nodemailer

```bash
npm install @strapi/provider-email-nodemailer
```

### Configurer dans config/plugins.ts

```typescript
email: {
  config: {
    provider: 'nodemailer',
    providerOptions: {
      host: env('SMTP_HOST'),
      port: env('SMTP_PORT'),
      auth: {
        user: env('SMTP_USERNAME'),
        pass: env('SMTP_PASSWORD'),
      },
    },
    settings: {
      defaultFrom: env('SMTP_FROM'),
      defaultReplyTo: env('SMTP_FROM'),
    },
  },
},
```

---

## 11. Mettre à jour le code sur le VPS

```bash
cd ~/nom-du-repo
git fetch origin
git reset --hard origin/main
npm install
NODE_ENV=production NODE_OPTIONS="--max-old-space-size=1536" npm run build
pm2 restart strapi-cme
```

---

## Commandes utiles

```bash
# Voir les logs Strapi
pm2 logs strapi-cme --lines 50

# Statut des processus
pm2 status

# Redémarrer Strapi
pm2 restart strapi-cme

# Tester Nginx
sudo nginx -t

# Renouvellement SSL (automatique, mais si besoin manuel)
sudo certbot renew
```

