# CI/CD avec GitHub Actions + Docker (Github)

Ce projet démontre l'intégration et le déploiement continus (CI/CD) d'une application React + Express utilisant GitHub Actions et Docker.

## Pré-requis

✅ Compte GitHub  
✅ Docker installé (WSL ou Linux/macOS)  
✅ Projet Node.js / React prêt (et containerisé)  
✅ Serveur VPS Linux (ex: debian)  
✅ Clé SSH pour se connecter au VPS  
✅ GHCR Token + compte  

## Structure du Projet
```
project/
├── backend/               # Express
│   └── Dockerfile
├── frontend/             # React (Vite)
│   └── Dockerfile
├── docker-compose.yml     # Déploiement
└── .github/
    └── workflows/
        └── deploy.yml     # CI/CD + déploiement
```

## Configuration des Dockerfiles

### Backend (Express)
```dockerfile
FROM node:22
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 5000
CMD ["npm", "start"]
```

### Frontend (React + Vite)
```dockerfile
FROM node:22 as build
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

## Configuration du Déploiement

### 1. Configuration du Serveur

Sur votre serveur, créez un dossier pour le déploiement :

```bash
mkdir -p /home/debian/deploy
```

Créez le fichier docker-compose.yml :

```yaml
services:
  frontend:
    image: ghcr.io/<github_username>/myapp-frontend
    ports:
      - "80:80"
    restart: always

  backend:
    image: ghcr.io/<github_username>/myapp-backend
    ports:
      - "5000:5000"
    restart: always
```

> Remplacez <github_username> par votre nom d'utilisateur ou d'organisation GitHub.

### 2. Configuration des Secrets GitHub

Dans votre dépôt GitHub > **Settings > Secrets and variables > Actions**, ajoutez :

| Nom | Description |
| --- | --- |
| `SERVER_HOST` | IP publique de votre serveur |
| `SERVER_USER` | Nom de l'utilisateur (ex. `debian`) |
| `SSH_PRIVATE_KEY` | Clé privée SSH pour connexion au VPS |

### 3. Configuration du Workflow GitHub Actions

Créez le fichier `.github/workflows/deploy.yml` :

```yaml
name: Build, Push & Deploy to VPS

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read
  packages: write

env:
  GHCR_IMAGE_FRONTEND: ghcr.io/${{ github.repository_owner }}/myapp-frontend
  GHCR_IMAGE_BACKEND: ghcr.io/${{ github.repository_owner }}/myapp-backend

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Frontend
        run: |
          docker build -t $GHCR_IMAGE_FRONTEND ./frontend
          docker push $GHCR_IMAGE_FRONTEND

      - name: Build and Push Backend
        run: |
          docker build -t $GHCR_IMAGE_BACKEND ./backend
          docker push $GHCR_IMAGE_BACKEND

      - name: Deploy on VPS via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin
            cd /home/debian/deploy
            docker compose pull
            docker compose up -d
```

> **Note**: Dans votre dépôt GitHub, sous Settings > Actions > General, assurez-vous que l'option Workflow permissions est configurée sur "Read and write permissions".

## Déploiement

Une fois tout configuré :
1. Committez et poussez vos changements sur la branche main
2. GitHub Actions construira les images Docker
3. Les images seront poussées sur GitHub Container Registry
4. Le serveur sera mis à jour automatiquement avec les nouvelles versions
