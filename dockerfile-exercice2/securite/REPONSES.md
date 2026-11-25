# Sécurisation d'un conteneur Docker : quiz-docker (exercice 2)

- Dépôt : https://github.com/adam-benyekkou/docker-introduction-adam (branche `docker-exercice-securite`)
- Image initiale : `cavydev/quiz-docker:initial` (anciennement `cavydev/dockerfile-exercice2:1.0.0`)
- Image sécurisée : `cavydev/quiz-docker:secure`
- Variante DHI : `cavydev/quiz-docker:dhi`
- Environnement : Docker 29.10.0, Trivy 0.76.0 (`aquasec/trivy`), amd64

---

## 1. Mission 1 : audit de l'image existante

### 1.1 Réponses aux questions

**L'image est-elle adaptée à une production ?**
Non. Elle exécute `npm run dev` (serveur de développement Vite) en root, sur une base
Node 18 dont le support est terminé. L'image finale contient toute la chaîne de build
(npm, devDependencies, sources) et aucune limite de ressources n'est posée.

**Quels composants sont réellement nécessaires ?**
Les fichiers produits par `vite build` (`dist/`) et un serveur HTTP statique pour les
servir. Node, npm, Vite, les sources et `node_modules` ne servent qu'à la compilation.

**Quels risques de sécurité ?**

| Risque | Constat sur `quiz-docker:initial` |
|---|---|
| Exécution en root | `docker run --rm quiz-docker:initial id` donne `uid=0(root)` |
| Serveur de développement exposé | `CMD npm run dev -- --host 0.0.0.0`, port 5173 |
| Base obsolète | Node `v18.20.8`, Alpine `3.21.3` (Trivy : « OS version no longer supported ») |
| Outils inutiles en production | `node`, `npm`, `apk` et `sh` présents dans l'image finale |
| Surface d'attaque | 264 vulnérabilités dont 5 CRITICAL et 83 HIGH |
| Tag de base non figé | `node:18-alpine` change selon la date de construction |
| `.dockerignore` minimal | 3 motifs (`Dockerfile.initial.dockerignore`) : un `.env` ou un `*.pem` entre dans l'image via `COPY . .` |
| Aucune limite à l'exécution | Ni mémoire, ni CPU, ni `pids-limit`, ni `cap-drop` |
| Socket Docker | Non monté, mais rien n'empêche de le monter |

**Améliorations prioritaires**
1. Build multi-étapes : Node pour compiler, runtime statique minimal.
2. Utilisateur non-root.
3. Base minimale, maintenue, à tag précis.
4. `.dockerignore` complet.
5. Limites de ressources et de privilèges à l'exécution.
6. Scan régulier des images.

### 1.2 Tableau d'état initial

| Critère | `quiz-docker:initial` |
|---|---|
| Image de base | `node:18-alpine` (Alpine 3.21.3) |
| Taille | 221 Mo |
| Utilisateur | `root` (uid 0) |
| Outils de développement | node 18.20.8, npm 10.8.2, apk |
| Mode d'exécution | serveur de développement Vite, port 5173 |
| Multi-stage | non |
| `.dockerignore` | 3 motifs |
| `COPY` / `ADD` | `COPY . .` |
| Limites CPU et mémoire | aucune |
| Capabilities | celles du moteur par défaut |
| Système de fichiers racine | inscriptible |
| Socket Docker | non monté |
| CVE CRITICAL | 5 |
| CVE HIGH | 83 |
| CVE MEDIUM | 130 |
| CVE LOW | 45 |
| CVE UNKNOWN | 1 |
| Total CVE | 264 |

Détail Trivy (`securite/scans/initial.txt`) : Alpine 3.21.3, 100 (2C/22H/41M/35L) ;
paquets Node, 75 (1C/32H/36M/6L) ; npm, 89 (2C/29H/53M/4L, 1 UNKNOWN).

---

## 2. Mission 2 : recommandations appliquées

| Recommandation | Niveau | Mise en œuvre | Preuve |
|---|---|---|---|
| Image de base minimale | Dockerfile | runtime `nginxinc/nginx-unprivileged:1.29-alpine3.23`, seul `dist/` est copié | `docker exec qs command -v node npm` : absent |
| Ne pas tourner en root | Dockerfile | `USER 101` | `docker exec qs id` : `uid=101(nginx)` |
| `.dockerignore` | Build | secrets, artefacts, `.git`, outillage et livrables exclus | `.dockerignore` (18 motifs) |
| Tags d'images précis | Dockerfile | `node:22.22-alpine3.24` et `nginxinc/nginx-unprivileged:1.29-alpine3.23` | `grep '^FROM' Dockerfile` |
| `COPY` plutôt que `ADD` | Dockerfile | `COPY` explicites uniquement | `grep -c '^ADD' Dockerfile` : 0 |
| Limiter ressources et privilèges | Exécution | `--memory 128m --cpus 0.5 --pids-limit 100 --cap-drop ALL --security-opt no-new-privileges --read-only` | `docker inspect` : `Memory=134217728`, `NanoCpus=500000000`, `PidsLimit=100`, `CapDrop=[ALL]`, `ReadonlyRootfs=true` |
| Build multi-étapes | Dockerfile | étape `build` (Node), puis étape runtime (nginx) | `docker history quiz-docker:secure` ne contient ni `npm ci` ni sources |
| Pas de socket Docker exposé | Exécution | aucun `-v /var/run/docker.sock`, scans via `docker save` puis `trivy image --input` | `docker inspect -f '{{len .Mounts}}' qs` : 0 |
| Vérifier régulièrement | Processus | scans Trivy versionnés dans `securite/scans/`, commande en §5 | `securite/scans/*.txt` |

Correctifs de paquets : la base épinglée peut être en retard sur des CVE déjà corrigées
en amont. Le Dockerfile exécute donc `apk upgrade --no-cache` dans l'étape runtime, avec
un `USER root` limité à cette couche. Effet mesuré : de 248 vulnérabilités à 0
(`securite/scans/secure-sans-upgrade.txt` et `secure.txt`). En production, il vaut mieux
reconstruire depuis une base déjà corrigée ; cet `upgrade` est un filet de sécurité.

### Décisions d'architecture

**Pourquoi nginx statique plutôt que Node ou le serveur de développement ?**
L'application est une SPA Vite : après `vite build`, il ne reste que des fichiers
statiques. Garder Node au runtime conserverait npm, `node_modules` et un serveur conçu
pour le développement, sans bénéfice fonctionnel. nginx sert `dist/` avec un utilisateur
non privilégié par défaut.

**Pourquoi le port 8080 ?**
Un processus non-root ne peut pas ouvrir un port inférieur à 1024. L'image
`nginx-unprivileged` écoute sur 8080, ce qui évite la capability `NET_BIND_SERVICE` et
permet `--cap-drop ALL`. Le port passe de 5173 à 8080 : `-p 127.0.0.1:8080:8080`.

**Alternatives écartées**
- Node au runtime avec `vite preview` : garde toute la chaîne Node et ses CVE.
- `nginx` standard avec `USER nginx` : il écrit dans `/var/cache/nginx` et `/var/run`
  au démarrage, l'image non privilégiée est déjà adaptée.
- Image DHI : étudiée en mission 4 (`Dockerfile.dhi`), plus légère (24,2 Mo) et
  équivalente sur les CVE. Conclusion en §4.3.

---

## 3. Mission 3 : comparaison avant et après

### 3.1 Commandes

```bash
# Image initiale (état de départ)
docker build -f Dockerfile.initial -t quiz-docker:initial .

# Image sécurisée
docker build -t quiz-docker:secure .

# Exécution durcie
docker run -d --name qs \
  --read-only --tmpfs /tmp --tmpfs /var/cache/nginx --tmpfs /var/run \
  --cap-drop ALL --security-opt no-new-privileges \
  --memory 128m --cpus 0.5 --pids-limit 100 \
  -p 127.0.0.1:8080:8080 quiz-docker:secure

# Publication
docker tag quiz-docker:initial cavydev/quiz-docker:initial
docker tag quiz-docker:secure  cavydev/quiz-docker:secure
docker push cavydev/quiz-docker:initial
docker push cavydev/quiz-docker:secure

# Variante Docker Hardened Image
docker build -f Dockerfile.dhi -t quiz-docker:dhi .
docker tag quiz-docker:dhi cavydev/quiz-docker:dhi
docker push cavydev/quiz-docker:dhi
```

### 3.2 Tableau comparatif

| Critère | Initial | Sécurisé | DHI |
|---|---|---|---|
| Image de base | `node:18-alpine` (Alpine 3.21.3) | `nginxinc/nginx-unprivileged:1.29-alpine3.23` (Alpine 3.23.4) | `dhi.io/nginx:1.30-alpine3.24` (Alpine 3.24) |
| Taille | 221 Mo | 118 Mo (-47 %) | 24,2 Mo (-89 %) |
| Utilisateur | root (uid 0) | nginx (uid 101) | 65532 |
| Outils de développement | node, npm, apk, sources, `node_modules` | absents (seul `dist/` est copié) | absents, busybox et un shell restent présents |
| Gestionnaire de paquets | apk | apk | absent |
| Serveur applicatif | serveur de développement Vite | nginx (statique) | nginx (statique) |
| Port | 5173 | 8080 | 8080 |
| `.dockerignore` | 3 motifs | 18 motifs | identique |
| Multi-stage | non | oui (2 étapes) | oui (2 étapes) |
| `ADD` | non | non | non |
| Tags précis | non (`18-alpine`) | oui | oui |
| Limites CPU et mémoire | aucune | 0,5 CPU, 128 Mo, 100 pids | 0,5 CPU, 128 Mo, 100 pids |
| Privilèges | par défaut | `--cap-drop ALL`, `no-new-privileges` | `--cap-drop ALL`, `no-new-privileges` |
| Racine en lecture seule | non | oui (`--read-only` et `tmpfs`) | oui (`--read-only` et 5 `tmpfs` appartenant à 65532) |
| Accès au socket Docker | non | non (`Mounts` = 0) | non (`Mounts` = 0) |
| CVE CRITICAL | 5 | 0 | 0 |
| CVE HIGH | 83 | 0 | 0 |
| CVE MEDIUM | 130 | 0 | 0 |
| CVE LOW | 45 | 0 | 0 |
| Total CVE | 264 | 0 | 0 |

Sources : `securite/scans/initial.txt`, `secure.txt` et `dhi.txt`. Valeurs intermédiaires :
248 (`secure-sans-upgrade.txt`) et 72 (`base129.txt`).

### 3.3 Preuves d'exécution

| Vérification | Résultat |
|---|---|
| `curl -sI localhost:8080` | `HTTP/1.1 200 OK`, `Server: nginx/1.29.8` |
| `curl -s localhost:8081` (DHI) | `HTTP=200` |
| État du conteneur `qs` après 15 s | `Up (healthy)` |
| `docker exec qs id` | `uid=101(nginx) gid=101(nginx)` |
| `docker exec qs command -v node npm` | absent |
| `docker inspect` (secure) | `ReadonlyRootfs=true`, `CapDrop=[ALL]`, `Memory=134217728`, `NanoCpus=500000000`, `PidsLimit=100`, `Mounts=0` |
| `docker inspect` (DHI) | `user=65532`, `CapDrop=[ALL]`, `Mounts=0`, `Memory=134217728` |
| `docker run --rm quiz-docker:secure ls /var/run/docker.sock` | `No such file or directory` |
| `docker run --rm --entrypoint sh quiz-docker:dhi -c 'ls /var/run/docker.sock'` | `absent` |
| `docker push` | `initial`, `secure` et `dhi` poussées (exit 0) |

### 3.4 Questions d'analyse

**Quelles modifications ont apporté les améliorations les plus significatives ?**
Le build multi-étapes, qui retire Node, npm et les sources de l'image finale : il
supprime à lui seul les 164 CVE des cibles Node et npm et réduit la taille de 47 %.
Viennent ensuite le passage en non-root avec `--cap-drop ALL` et `no-new-privileges`, qui
limitent l'impact d'une compromission, puis `apk upgrade`, qui ferme les CVE corrigeables
de la base (de 248 à 0). `--read-only` empêche enfin l'attaquant d'écrire dans le
conteneur.

**La taille de l'image a-t-elle diminué ? Pourquoi ?**
Oui, de 221 Mo à 118 Mo. L'image finale ne contient plus Node, npm, `node_modules`, les
sources ni les devDependencies, seulement les fichiers statiques de Vite et nginx sur
Alpine. `apk upgrade` ajoute 25 Mo de couche, compensés par la suppression de la chaîne
Node.

**Quelles recommandations concernent le Dockerfile et lesquelles concernent l'exécution ?**
- Dockerfile : base minimale, tag précis, multi-stage, `COPY` au lieu de `ADD`, `USER`
  non-root, `.dockerignore`, `apk upgrade`, `HEALTHCHECK`.
- Exécution : `--memory`, `--cpus`, `--pids-limit`, `--cap-drop ALL`,
  `no-new-privileges`, `--read-only` avec `tmpfs`, aucun montage du socket Docker, scan
  régulier et mise à jour des images.

**L'image est-elle totalement sécurisée ? Justifiez.**
Non. Toute image hérite d'une base et de paquets qui évoluent. Le scan est propre le jour
de la construction, mais :
- une nouvelle CVE peut apparaître le lendemain ;
- les dépendances npm de build ont des vulnérabilités connues (`npm-audit.txt`) : elles
  restent dans l'étape `build`, mais la chaîne de build n'est pas corrigée ;
- l'image n'est pas signée et ne porte pas de SBOM (BuildKit ajoute seulement une
  attestation de provenance SLSA) ;
- l'image runtime contient un shell (busybox), utile à un attaquant qui obtient une
  exécution de code ;
- le trafic est en HTTP, sans TLS, sans limitation de débit ni WAF ;
- la configuration du démon Docker et de l'hôte n'est pas couverte.

**Mesures complémentaires pour la production**
1. CI qui échoue sur les CVE CRITICAL et HIGH (Trivy avec `--exit-code`) et reconstruction
   automatique à la publication d'une base corrigée.
2. SBOM (Syft ou Trivy) et signature d'image (cosign), vérifiées à l'admission.
3. Runtime sans shell : l'image DHI testée en mission 4 embarque busybox et `/bin/sh`
   (§4.3), il faut une variante réellement distroless.
4. Équivalent Kubernetes : `runAsNonRoot`, `readOnlyRootFilesystem`,
   `allowPrivilegeEscalation: false`, `seccompProfile: RuntimeDefault`.
5. Politiques réseau, TLS en terminaison, limitation de débit, WAF.
6. Journalisation centralisée et détection au runtime (Falco).
7. Secrets hors de l'image (variables d'environnement, gestionnaire de secrets).

---

## 4. Mission 4 : Docker Hardened Image (DHI)

L'atelier https://docs.docker.com/guides/lab-dhi-node/ renvoie une erreur 404. La
migration a donc été faite directement sur l'application, avec `docker login dhi.io`.

### 4.1 Mise en œuvre

`Dockerfile.dhi` ne diffère de `Dockerfile` que par les bases :

- build : `dhi.io/node:22-alpine3.22-dev`. La variante `-dev` est la seule à fournir
  `npm` (l'image DHI Node standard ne l'embarque pas). Cette étape est jetée.
- runtime : `dhi.io/nginx:1.30-alpine3.24`, utilisateur `65532`, port 8080.
- pas d'`apk upgrade` : DHI n'a pas de gestionnaire de paquets. Un correctif impose de
  reconstruire sur un nouveau tag.

```bash
docker build -f Dockerfile.dhi -t quiz-docker:dhi .
docker run -d --name qdhi \
  --read-only \
  --tmpfs /tmp:uid=65532,gid=65532 \
  --tmpfs /var/cache/nginx:uid=65532,gid=65532 \
  --tmpfs /run/nginx:uid=65532,gid=65532 \
  --tmpfs /var/lib/nginx/tmp:uid=65532,gid=65532 \
  --tmpfs /var/lib/nginx/logs:uid=65532,gid=65532 \
  --cap-drop ALL --security-opt no-new-privileges \
  --memory 128m --cpus 0.5 --pids-limit 100 \
  -p 127.0.0.1:8081:8080 quiz-docker:dhi
```

Les `tmpfs` doivent appartenir à l'uid 65532, sinon nginx échoue en lecture seule
(`mkdir() "/var/lib/nginx/tmp/client_body" failed (13: Permission denied)`).

### 4.2 Comparaison DHI et image sécurisée

| Critère | `quiz-docker:secure` | `quiz-docker:dhi` |
|---|---|---|
| Taille | 118 Mo | 24,2 Mo |
| Base | nginx-unprivileged 1.29 (Alpine 3.23.4) | nginx DHI 1.30 (Alpine 3.24) |
| Utilisateur | 101 | 65532 |
| Shell | busybox et `/bin/sh` | busybox et `/bin/sh` |
| Gestionnaire de paquets | apk | absent |
| CVE (Trivy) | 0 | 0 |
| Architectures | non testé | amd64 et arm64 |
| Correctifs | `apk upgrade` à la construction | nouveau tag DHI à chaque publication |

### 4.3 Enseignements

- Gain principal, la taille : 24,2 Mo contre 118 Mo (-79 %), et -89 % par rapport à
  l'image initiale. La base DHI ne contient que nginx et ses dépendances.
- Aucun gain mesurable sur les CVE : les deux images sont à 0 le jour du scan. Le tag
  `1.29-alpine3.23` de DHI mesurait pourtant 72 vulnérabilités dont 11 HIGH
  (`base129.txt`), ce qui montre que la fraîcheur du tag compte plus que le label. La base
  1.30 seule est aussi à 0 (`base130.txt`).
- Pas de gain sur le shell : cette image DHI embarque busybox et un shell, contrairement
  à une image distroless.
- Contrainte opérationnelle : sans `apk`, il n'y a pas de correctif à chaud, il faut
  reconstruire à chaque publication DHI. Les images sont immuables, mais le pipeline doit
  suivre.
- Signature et SBOM : DHI annonce des images signées avec SBOM et provenance. Je n'ai pas
  pu le vérifier, `cosign` n'étant pas installé.

Conclusion : pour cette application, DHI est le meilleur candidat de production sur la
taille et sur la gestion des correctifs par l'éditeur, à condition d'automatiser la
reconstruction sur nouveau tag et de garder un scan en CI. `quiz-docker:secure` reste
la variante qui ne dépend pas d'un registre à authentification.

---

## 5. Risques résiduels et synthèse

Améliorations apportées : image deux fois plus légère, exécution non-root sans
capabilities, racine en lecture seule, 0 CVE au scan, aucun outil de développement dans
l'image finale, socket Docker jamais exposé, serveur statique seul exposé. La variante
DHI atteint 24,2 Mo avec le même résultat.

Risques résiduels :
1. Le scan n'est valable que le jour où il est fait : il faut reconstruire à chaque
   correctif amont.
2. Pas de signature ni de SBOM (`imagetools inspect --format '{{json .SBOM}}'` renvoie
   `{}`), seulement une provenance SLSA de BuildKit. `cosign` est absent.
3. Un shell est présent dans l'image runtime, y compris dans l'image DHI.
4. HTTP en clair, sans limitation de débit.
5. `npm audit` (`securite/scans/npm-audit.txt`) signale 6 vulnérabilités (5 high, 1
   moderate) dans les dépendances de build, dont `source-map-js`. Elles restent dans
   l'étape `build` et n'atteignent pas l'image finale. L'application n'est pas modifiée,
   conformément à la consigne.
6. `apk upgrade` rend la construction non strictement reproductible. La variante DHI
   n'a pas ce défaut mais impose une reconstruction à chaque nouveau tag.
7. Les bases ne sont pas épinglées par digest (`@sha256:...`) et
   `# syntax=docker/dockerfile:1` n'est pas figé. En production, épingler par digest
   (`docker buildx imagetools inspect <image>`).
8. Les identifiants Docker Hub sont stockés en base64 dans `~/.docker/config.json`
   (pas de `credsStore`). À remplacer par un credential helper.
9. L'image DHI n'a pas de `HEALTHCHECK` (pas de `wget`) : la supervision revient à
   l'orchestrateur.

Commandes de scan :

```bash
docker save quiz-docker:secure -o /tmp/secure.tar
docker run --rm -v /tmp:/in:ro aquasec/trivy:latest \
  image --input /in/secure.tar --scanners vuln --format table
```

Le scan passe par une archive en lecture seule et ne monte pas le socket Docker, comme
le demande le TP.

---

## 6. Livrables

| Livrable | Fichier |
|---|---|
| Dockerfile initial | `dockerfile-exercice2/Dockerfile.initial` (avec `Dockerfile.initial.dockerignore`, les 3 motifs d'origine) |
| Dockerfile sécurisé | `dockerfile-exercice2/Dockerfile` |
| Dockerfile DHI | `dockerfile-exercice2/Dockerfile.dhi` |
| `.dockerignore` | `dockerfile-exercice2/.dockerignore` |
| Scan avant | `dockerfile-exercice2/securite/scans/initial.txt` |
| Scan après | `dockerfile-exercice2/securite/scans/secure.txt` |
| Scan DHI | `dockerfile-exercice2/securite/scans/dhi.txt` |
| Scans intermédiaires | `dockerfile-exercice2/securite/scans/secure-sans-upgrade.txt`, `base129.txt`, `npm-audit.txt` |
| Scan de la base DHI seule | `dockerfile-exercice2/securite/scans/base130.txt` |
| Rapport | ce fichier |
| Images publiées | `cavydev/quiz-docker:initial`, `:secure`, `:dhi` |
| Dépôt Git | https://github.com/adam-benyekkou/docker-introduction-adam (branche `docker-exercice-securite`) |
