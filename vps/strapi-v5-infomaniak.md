# Héberger Strapi v5 sur un VPS Linux

## Stack

- **OS** : Ubuntu 24.04 LTS
- **Runtime** : Node.js 22 LTS (via nvm)
- **Base de données** : PostgreSQL
- **Process manager** : PM2
- **Reverse proxy** : Nginx
- **SSL** : Let's Encrypt (Certbot)

---

## 1. Connexion au VPS

```bash
ssh -i ~/.ssh/ma-cle ubuntu@IP_DU_VPS
```

---

## 2. Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Installer nvm et Node.js

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/$(curl -s https://api.github.com/repos/nvm-sh/nvm/releases/latest | grep '"tag_name"' | cut -d'"' -f4)/install.sh | bash
source ~/.bashrc

nvm install 22
nvm alias default 22
node --version
```

> Strapi v5 supporte Node.js v20, v22 et v24 (LTS uniquement).

---

## 4. Installer PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib -y
sudo systemctl status postgresql
```

### Créer un utilisateur et une base de données

```bash
sudo -u postgres psql
```

```sql
CREATE USER mon_user WITH PASSWORD 'mot-de-passe';
CREATE DATABASE ma_db OWNER mon_user;
\q
```

> Éviter les caractères spéciaux dans le mot de passe — ils peuvent causer des problèmes d'échappement dans le .env.

---

## 5. Installer PM2 et Nginx

```bash
npm install -g pm2
sudo apt install nginx -y
```

---

## 6. Ajouter un swap (recommandé si RAM ≤ 2 Go)

Le build Strapi est gourmand en mémoire. Un swap évite les erreurs out of memory.

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Rendre le swap permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 7. Déployer Strapi

### Cloner le repo

```bash
git clone https://github.com/user/repo.git
cd repo
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
DATABASE_NAME=ma_db
DATABASE_USERNAME=mon_user
DATABASE_PASSWORD=mot-de-passe
```

> Les valeurs APP_KEYS, secrets et salts se trouvent dans le .env local du projet.

### Builder Strapi

```bash
NODE_ENV=production NODE_OPTIONS="--max-old-space-size=1536" npm run build
```

> Le build peut prendre plusieurs minutes sur un petit VPS.

### Démarrer avec PM2

```bash
NODE_ENV=production pm2 start npm --name "strapi" -- start
pm2 startup
pm2 save
```

---

## 8. Configurer Nginx comme reverse proxy

```bash
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

## 9. Configurer le DNS

Chez le registrar, créer un enregistrement de type `A` :

- Sous-domaine : `strapi` (ou `api`, etc.)
- Cible : IP du VPS
- TTL : valeur par défaut

---

## 10. Configurer le SSL

### Ouvrir les ports 80 et 443 dans le firewall du VPS

### Installer Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d mon-domaine.com
```

> Certbot met à jour automatiquement la config Nginx pour rediriger HTTP → HTTPS et renouvelle le certificat automatiquement.

---

## 11. Configurer l'email (SMTP)

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

### Variables .env correspondantes

```env
SMTP_HOST=smtp.mon-provider.com
SMTP_PORT=587
SMTP_USERNAME=adresse@domaine.com
SMTP_PASSWORD=mot-de-passe
SMTP_FROM=adresse@domaine.com
```

---

## 12. Mettre à jour le code sur le VPS

```bash
cd ~/repo
git fetch origin
git reset --hard origin/main
npm install
NODE_ENV=production NODE_OPTIONS="--max-old-space-size=1536" npm run build
pm2 restart strapi
```

---

## Commandes utiles

```bash
# Logs
pm2 logs strapi --lines 50

# Statut
pm2 status

# Redémarrer
pm2 restart strapi

# Vérifier la config Nginx
sudo nginx -t

# Renouveler le certificat SSL manuellement
sudo certbot renew
```
