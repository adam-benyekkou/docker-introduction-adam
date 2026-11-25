# syntax=docker/dockerfile:1
# Image sécurisée : build Node, puis nginx non privilégié (voir securite/REPONSES.md)

# Étape 1 : build (tag précis)
FROM node:22.22-alpine3.24 AS build

WORKDIR /app

# Manifestes d'abord pour garder la couche d'installation en cache
COPY package.json package-lock.json ./
RUN npm ci

COPY index.html main.js style.css vite.config.js javascript.svg ./
COPY questions_defaut.js questions_defaut_2.js ./
COPY images ./images
COPY public ./public

RUN npm run build

# Étape 2 : runtime (nginx non-root, tag précis)
FROM nginxinc/nginx-unprivileged:1.29-alpine3.23

# Correctifs des paquets Alpine déjà publiés (root limité à cette couche)
USER root
RUN apk upgrade --no-cache && rm -rf /var/cache/apk/*

# Seul dist/ est copié, ni sources ni node_modules
COPY --from=build /app/dist /usr/share/nginx/html

# Un utilisateur non-root ne peut pas ouvrir un port inférieur à 1024
EXPOSE 8080

USER 101

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/ >/dev/null || exit 1

CMD ["nginx", "-g", "daemon off;"]
