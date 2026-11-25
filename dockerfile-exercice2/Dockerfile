# syntax=docker/dockerfile:1
# Dockerfile sécurisé — quiz-docker (voir securite/REPONSES.md)
#
# Deux étapes :
#   1. build  : Node compile l'application Vite en fichiers statiques (dist/)
#   2. runtime: nginx non privilégié sert dist/ — Node, npm et les sources
#               de l'application ne sont pas présents dans l'image finale.

# ---------------------------------------------------------------------------
# Étape 1 : build
# ---------------------------------------------------------------------------
# Tag précis : version Node LTS + version Alpine fixées.
FROM node:22.22-alpine3.24 AS build

WORKDIR /app

# Manifestes d'abord : la couche d'installation reste en cache tant que
# package-lock.json ne change pas.
COPY package.json package-lock.json ./

# npm ci installe exactement le lockfile (reproductible).
RUN npm ci

# COPY, jamais ADD : pas de décompression ni d'URL distante implicite.
COPY index.html main.js style.css vite.config.js javascript.svg ./
COPY questions_defaut.js questions_defaut_2.js ./
COPY images ./images
COPY public ./public

RUN npm run build

# ---------------------------------------------------------------------------
# Étape 2 : runtime
# ---------------------------------------------------------------------------
# Image nginx dont l'utilisateur par défaut est non-root (uid 101) et dont
# les chemins d'écriture sont déjà adaptés à un utilisateur non privilégié.
# Tag précis : version nginx + version Alpine fixées.
FROM nginxinc/nginx-unprivileged:1.29-alpine3.23

# Application des correctifs des paquets Alpine au moment du build :
# la base épinglée peut être en retard sur les CVE déjà corrigées en amont.
# Le USER root est limité à cette seule couche, puis on revient à 101.
# En production, la bonne pratique reste de reconstruire depuis une base
# patchée ; cet upgrade est un filet de sécurité pour les CVE « fixed ».
USER root
RUN apk upgrade --no-cache && rm -rf /var/cache/apk/*

# Seul le résultat du build est copié : pas de sources, pas de node_modules.
COPY --from=build /app/dist /usr/share/nginx/html

# nginx-unprivileged écoute sur 8080 : un utilisateur non-root ne peut pas
# ouvrir un port < 1024.
EXPOSE 8080

# Non-root explicite, même si l'image de base le fait déjà.
USER 101

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/ >/dev/null || exit 1

CMD ["nginx", "-g", "daemon off;"]
