# Image de base Node.js légère
FROM node:18-alpine

# Répertoire de travail de l'application
WORKDIR /app

# Copie des manifestes d'abord pour profiter du cache Docker
COPY package.json package-lock.json ./

# Installation des dépendances à partir du lockfile
RUN npm ci

# Copie du reste du code source
COPY . .

# Port du serveur de développement Vite
EXPOSE 5173

# Écoute sur toutes les interfaces réseau du conteneur
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
