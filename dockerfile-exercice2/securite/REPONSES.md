# Sécurisation d'un conteneur Docker — quiz-docker (exercice 2)

- Dépôt : https://github.com/adam-benyekkou/docker-introduction-adam (branche `docker-exercice-securite`)
- Image initiale : `cavydev/quiz-docker:initial` (ex-`cavydev/dockerfile-exercice2:1.0.0`)
- Image sécurisée : `cavydev/quiz-docker:secure`
- Variante DHI : `cavydev/quiz-docker:dhi`
- Environnement : Docker 29.10.0, Trivy 0.76.0 (`aquasec/trivy`), build multi-étapes sur amd64

---

## 1. Mission 1 — Audit de l'image existante

### 1.1 Réponses aux questions

**L'image est-elle adaptée à une production ?**
Non. Elle exécute `npm run dev` (serveur de développement Vite) en **root**, sur une base
Node 18 dont le support est terminé, avec l'intégralité de la chaîne de build (npm,
devDependencies, sources) dans l'image finale, sans aucune limite de ressources.

**Quels composants sont réellement nécessaires ?**
Uniquement les fichiers produits par `vite build` (`dist/`) et un serveur HTTP statique
capable de les servir. Node, npm, Vite, les sources et `node_modules` ne sont nécessaires
qu'à la compilation.

**Quels risques de sécurité ?**

| Risque | Constat mesuré sur `quiz-docker:initial` |
|---|---|
| Exécution en root | `docker run --rm quiz-docker:initial id` → `uid=0(root)` |
| Serveur de développement exposé | `CMD npm run dev -- --host 0.0.0.0`, port 5173 |
| Base obsolète | Node `v18.20.8`, Alpine `3.21.3` (Trivy : « OS version no longer supported ») |
| Chaîne d'outils inutile en runtime | `node`, `npm`, `apk`, `sh` présents dans l'image finale |
| Surface d'attaque élevée | 264 vulnérabilités dont **5 CRITICAL / 83 HIGH** |
| Tag de base flottant | `node:18-alpine`, contenu variable selon les rebuilds |
| `.dockerignore` minimal | 3 motifs seulement (`Dockerfile.initial.dockerignore`) : un `.env` ou `*.pem` entre dans l'image via `COPY . .` |
| Aucune limite runtime | Ni mémoire, ni CPU, ni `pids-limit`, ni `cap-drop` |
| Socket Docker | Non monté ici, mais aucun garde-fou documenté contre ce montage |

**Améliorations prioritaires**
1. Build multi-étapes : Node seulement pour compiler, runtime statique minimal.
2. Utilisateur non-root.
3. Base minimale et tag précis, maintenue.
4. `.dockerignore` complet (secrets, artefacts, outillage).
5. Limites de ressources et de privilèges à l'exécution.
6. Scan régulier des images.

### 1.2 Tableau d'état initial

| Critère | `quiz-docker:initial` |
|---|---|
| Image de base | `node:18-alpine` (Alpine 3.21.3) |
| Taille | **221 MB** |
| Utilisateur | `root` (uid 0) |
| Outils de dev présents | node 18.20.8, npm 10.8.2, apk |
| Mode d'exécution | serveur de dev Vite, port 5173 |
| Multi-stage | non |
| `.dockerignore` | 3 motifs |
| `COPY` / `ADD` | `COPY . .` |
| Limites CPU/mémoire | aucune |
| Capabilities | par défaut (toutes celles du moteur) |
| Système de fichiers racine | inscriptible |
| Socket Docker | non monté |
| CVE CRITICAL | **5** |
| CVE HIGH | **83** |
| CVE MEDIUM | **130** |
| CVE LOW | **45** |
| CVE UNKNOWN | **1** |
| Total CVE | **264** |

Détail Trivy (`securite/scans/initial.txt`) : Alpine 3.21.3 → 100 (2C/22H/41M/35L),
paquets Node → 75 (1C/32H/36M/6L), npm → 89 (2C/29H/53M/4L).

---

## 2. Mission 2 — Recommandations appliquées

| Recommandation | Niveau | Mise en œuvre | Preuve |
|---|---|---|---|
| Image de base minimale | Dockerfile | runtime `nginxinc/nginx-unprivileged:1.29-alpine3.23`, seulement `dist/` copié | `docker exec qs command -v node npm` → absent |
| Ne pas tourner en root | Dockerfile | `USER 101` (image non privilégiée) | `docker exec qs id` → `uid=101(nginx)` |
| `.dockerignore` | Build | secrets, artefacts, `.git`, outillage, livrables exclus | `.dockerignore` (18 motifs) |
| Tags d'images précis | Dockerfile | `node:22.22-alpine3.24` et `nginxinc/nginx-unprivileged:1.29-alpine3.23` | `grep '^FROM' Dockerfile` |
| `COPY` plutôt que `ADD` | Dockerfile | uniquement des `COPY` explicites, aucune URL ni archive | `grep -c '^ADD' Dockerfile` → 0 |
| Limiter ressources et privilèges | Exécution | `--memory 128m --cpus 0.5 --pids-limit 100 --cap-drop ALL --security-opt no-new-privileges --read-only` | `docker inspect` : `Memory=134217728`, `NanoCpus=500000000`, `PidsLimit=100`, `CapDrop=[ALL]`, `ReadonlyRootfs=true` |
| Build multi-étapes | Dockerfile | étape `build` (Node) puis étape runtime (nginx) | `docker history quiz-docker:secure` ne contient ni `npm ci` ni sources |
| Pas de socket Docker exposé | Exécution | aucun `-v /var/run/docker.sock`, scans via `docker save` + `trivy image --input` | `docker inspect -f '{{len .Mounts}}' qs` → `0` |
| Vérifier régulièrement | Processus | scans Trivy versionnés dans `securite/scans/`, commande documentée §5 | `securite/scans/*.txt` |

**Correctifs de paquets.** La base épinglée peut être en retard sur des CVE déjà corrigées
en amont. Le Dockerfile applique donc `apk upgrade --no-cache` dans l'étape runtime
(bloc `USER root` limité à cette couche). Effet mesuré : **248 → 0 vulnérabilité** (`securite/scans/secure-sans-upgrade.txt` contre `secure.txt`). En
production, la pratique de référence reste de reconstruire depuis une base patchée ; cet
upgrade est un filet de sécurité, pas un substitut au cycle de mise à jour.

### Décisions d'architecture

**Pourquoi nginx statique plutôt que Node ou le serveur de dev ?**
L'application est une SPA Vite. Après `vite build`, seuls des fichiers statiques
subsistent. Conserver Node en runtime maintiendrait npm, `node_modules` et un serveur de
développement non conçu pour la production, pour aucun gain fonctionnel. nginx sert
`dist/` avec un modèle de sécurité mature et un utilisateur non privilégié par défaut.

**Pourquoi le port 8080 ?**
Un processus non-root ne peut pas ouvrir un port < 1024. L'image
`nginx-unprivileged` écoute sur 8080 par défaut, ce qui évite toute capability
`NET_BIND_SERVICE` et permet `--cap-drop ALL`. Le port passe donc de 5173 (dev) à 8080 :
`-p 8080:8080`.

**Alternatives écartées**
- Node runtime + `vite preview` : garde toute la chaîne Node, donc les CVE npm.
- `nginx` standard + `USER nginx` : nginx standard écrit dans `/var/cache/nginx` et
  `/var/run` en root au démarrage ; l'image non privilégiée est déjà adaptée.
- Image DHI : étudiée en mission 4 (`Dockerfile.dhi`). Meilleure sur la taille (24,2 MB),
  équivalente sur les CVE ; voir §4.3 pour la conclusion et ses contreparties.

---

## 3. Mission 3 — Comparaison avant / après

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
| Taille | 221 MB | 118 MB (−47 %) | **24,2 MB (−89 %)** |
| Utilisateur | root (uid 0) | nginx (uid 101) | 65532 |
| Outils de développement | node, npm, apk, sources, `node_modules` | absents (seul `dist/` est copié) | absents ; busybox et un shell minimal restent présents |
| Gestionnaire de paquets | apk | apk | absent (pas de correctif à chaud possible) |
| Serveur applicatif | serveur de dev Vite | nginx (statique) | nginx (statique) |
| Port | 5173 | 8080 | 8080 |
| `.dockerignore` | 3 motifs | secrets, artefacts, `.git`, outillage, livrables exclus | identique |
| Multi-stage | non | oui (2 étapes) | oui (2 étapes) |
| `ADD` | non | non (`COPY` uniquement) | non (`COPY` uniquement) |
| Tags précis | non (`18-alpine`) | oui (versions Node/Alpine/nginx fixées) | oui (tags DHI versionnés) |
| Limitation CPU/mémoire | aucune | 0.5 CPU, 128 Mo, 100 pids | 0.5 CPU, 128 Mo, 100 pids |
| Restrictions de privilèges | aucune | `--cap-drop ALL`, `no-new-privileges` | `--cap-drop ALL`, `no-new-privileges` |
| Racine en lecture seule | non | oui (`--read-only` + `tmpfs`) | oui (`--read-only` + 5 `tmpfs` détenus par 65532) |
| Accès socket Docker | non | non, prouvé (`Mounts` = 0) | non, prouvé (`Mounts` = 0) |
| CVE CRITICAL | 5 | 0 | 0 |
| CVE HIGH | 83 | 0 | 0 |
| CVE MEDIUM | 130 | 0 | 0 |
| CVE LOW | 45 | 0 | 0 |
| Total CVE | 264 | **0** | **0** |

Sources : `securite/scans/initial.txt`, `securite/scans/secure.txt`, `securite/scans/dhi.txt`.
Chiffres intermédiaires : 248 (`secure-sans-upgrade.txt`), 72 (`base129.txt`).

### 3.3 Preuves d'exécution

| Vérification | Résultat |
|---|---|
| `curl -sI localhost:8080` | `HTTP/1.1 200 OK`, `Server: nginx/1.29.8` |
| `curl -s localhost:8081` (DHI) | `HTTP=200` |
| Santé du conteneur après 15 s | `Up (healthy)` (secure) |
| `docker exec qs id` | `uid=101(nginx) gid=101(nginx)` |
| `docker exec qs command -v node npm` | `absent` |
| `docker inspect` (secure) | `ReadonlyRootfs=true`, `CapDrop=[ALL]`, `Memory=134217728`, `NanoCpus=500000000`, `PidsLimit=100`, `Mounts=0` |
| `docker inspect` (DHI) | `user=65532`, `CapDrop=[ALL]`, `Mounts=0`, `Memory=134217728` |
| `docker run --rm quiz-docker:secure ls /var/run/docker.sock` | `No such file or directory` |
| `docker run --rm --entrypoint sh quiz-docker:dhi -c 'ls /var/run/docker.sock'` | `absent` |
| `docker push` | `initial`, `secure` et `dhi` poussées (exit 0) |

### 3.4 Questions d'analyse

**Quelles modifications ont apporté les améliorations les plus significatives ?**
Le build multi-étapes, qui supprime Node, npm et les sources de l'image finale : il
élimine à lui seul les 164 CVE des cibles Node/npm et réduit la taille de 47 %. Vient
ensuite le passage non-root avec `--cap-drop ALL` et `no-new-privileges`, qui limite
l'impact d'une compromission applicative. L'`apk upgrade` au build ferme les CVE
corrigeables de la base (248 → 0). Enfin `--read-only` empêche la persistance d'un
attaquant dans le conteneur.

**La taille de l'image a-t-elle diminué ? Pourquoi ?**
Oui : 221 MB → 118 MB. L'image finale ne contient plus Node, npm, `node_modules`, les
sources ni les devDependencies ; elle ne contient que les fichiers statiques produits
par Vite et nginx sur Alpine. Les paquets Alpine patchés par `apk upgrade` ajoutent
quelques mégaoctets, largement compensés par la suppression de la chaîne Node.

**Quelles recommandations concernent le Dockerfile et lesquelles concernent l'exécution ?**
- Dockerfile : base minimale, tag précis, multi-stage, `COPY` au lieu de `ADD`, `USER`
  non-root, `.dockerignore`, `apk upgrade`, `HEALTHCHECK`.
- Exécution : `--memory`, `--cpus`, `--pids-limit`, `--cap-drop ALL`,
  `no-new-privileges`, `--read-only` avec `tmpfs`, absence de montage du socket Docker,
  scan régulier et politique de mise à jour des images.

**L'image est-elle totalement sécurisée ? Justifiez.**
Non. Une image ne peut pas l'être « totalement » : toute image hérite d'une base et de
paquets qui évoluent. Ici, le scan est propre au jour du build, mais :
- une nouvelle CVE peut apparaître le lendemain ; le résultat n'est valable qu'à date ;
- les dépendances npm ne sont plus dans l'image finale, mais leur chaîne de build n'est
  pas vérifiée (pas de `npm audit`, pas de provenance) ;
- l'image n'est pas signée et ne porte pas de SBOM (l'attestation de provenance SLSA
  générée par BuildKit est présente, mais ne remplace ni l'un ni l'autre) ;
- l'image runtime contient un shell (busybox), utile à un attaquant qui obtient une
  exécution de code ;
- l'exposition réseau reste HTTP en clair, sans TLS, sans limitation de débit ni WAF ;
- la configuration du démon Docker et de l'hôte n'est pas couverte.

**Mesures complémentaires pour un déploiement en production**
1. CI qui bloque sur CVE CRITICAL/HIGH (Trivy + `--exit-code`), avec rebuild
   automatique lors de la publication d'une base patchée.
2. SBOM (Syft/Trivy) et signature d'image (cosign), vérification à l'admission.
3. Runtime sans shell : l'image DHI testée en mission 4 embarque busybox et `/bin/sh`
   (§4.3) ; il faut une variante réellement distroless pour supprimer le shell.
4. `securityContext` Kubernetes équivalent : non-root, `readOnlyRootFilesystem`,
   `allowPrivilegeEscalation: false`, `seccompProfile: RuntimeDefault`.
5. Politiques réseau, TLS en terminaison, limitation de débit, WAF.
6. Journalisation centralisée et détection runtime (Falco).
7. Gestion des secrets hors image (variables d'environnement, secret manager).

---

## 4. Mission 4 — Docker Hardened Image (DHI)

**Statut : réalisée** (`docker login dhi.io` effectué, images DHI accessibles).

### 4.1 Mise en œuvre

`Dockerfile.dhi` ne diffère de `Dockerfile` que par les bases :

- build : `dhi.io/node:22-alpine3.22-dev`. La variante `-dev` est la seule à fournir
  `npm` (l'image DHI Node standard ne l'embarque pas). Cette étape est jetée.
- runtime : `dhi.io/nginx:1.30-alpine3.24`, utilisateur `65532`, port 8080.
- Aucun `apk upgrade` : DHI ne fournit pas de gestionnaire de paquets, et les bases sont
  patchées par l'éditeur. Un correctif impose donc de reconstruire sur un nouveau tag.

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

### 4.2 Comparaison DHI / image sécurisée

| Critère | `quiz-docker:secure` | `quiz-docker:dhi` |
|---|---|---|
| Taille | 118 MB | **24,2 MB** |
| Base | nginx-unprivileged 1.29 (Alpine 3.23.4) | nginx DHI 1.30 (Alpine 3.24) |
| Utilisateur | 101 | 65532 |
| Shell | busybox + `/bin/sh` | busybox + `/bin/sh` (vérifié : `docker exec qdhi sh -c 'echo shell_present'` répond) |
| Gestionnaire de paquets | apk | absent |
| CVE (Trivy, tableau complet) | 0 | 0 |
| Multi-architecture | non testé | index amd64 + arm64, annotation `com.docker.dhi.build.id` |
| Correctifs | `apk upgrade` au build | nouveau tag DHI à chaque publication |

### 4.3 Enseignements

- **Gain principal : la taille.** 24,2 MB contre 118 MB, soit −79 % par rapport à l'image
  sécurisée et −89 % par rapport à l'initiale. La base DHI ne contient que nginx et ses
  dépendances strictes.
- **Aucun gain CVE mesurable ici.** Les deux images sont à 0 au moment du scan. Le tag
  DHI `1.29-alpine3.23` mesurait, lui, 72 vulnérabilités (11 HIGH) pendant l'audit :
  la fraîcheur du tag compte plus que le label « hardened ». Le scan de la base DHI
  1.30 seule (`securite/scans/base130.txt`) confirme 0 vulnérabilité avant même
  l'ajout de l'application.
- **Pas de gain sur le shell.** Contrairement à l'idée reçue « distroless », cette image
  DHI embarque busybox et un shell. Un attaquant qui obtient une exécution de code
  conserve un shell minimal.
- **Contrepartie opérationnelle.** Sans `apk`, impossible de patcher à chaud : il faut
  rebuilder à chaque publication DHI. C'est un avantage de gouvernance (images
  immuables, signature annoncée par l'éditeur mais non vérifiée ici) mais une contrainte de pipeline.
- **Signature et SBOM.** DHI annonce des images signées avec SBOM/provenance. Non
  vérifié ici : `cosign` n'est pas installé dans cet environnement, donc aucune preuve
  locale n'est avancée.

**Conclusion.** Pour cette application, DHI est le meilleur candidat production sur la
taille et la fraîcheur gérée par l'éditeur, à condition d'automatiser la reconstruction
sur nouveau tag et de conserver un scan en CI. `quiz-docker:secure` reste la variante
sans dépendance à un registre privé.

Aucune allégation de signature n'est faite sans vérification outillée.

---

## 5. Risques résiduels et synthèse

**Améliorations apportées.** Image deux fois plus légère, exécution non-root sans
capabilities, racine en lecture seule, aucune CVE détectée au build, aucun outil de
développement dans l'image finale, socket Docker jamais exposé, surface réseau réduite
à un serveur statique. La variante DHI descend à 24,2 MB pour le même résultat.

**Risques résiduels.**
1. Résultat de scan daté : la base doit être reconstruite à chaque correctif amont.
2. Pas de signature d'image ni de SBOM (`imagetools inspect --format '{{json .SBOM}}'`
   renvoie `{}`) ; seule une provenance SLSA BuildKit est attachée. `cosign` absent.
3. Shell présent dans l'image runtime, y compris dans l'image DHI.
4. HTTP en clair, sans limitation de débit.
5. Chaîne de build npm : `npm audit` (`securite/scans/npm-audit.txt`) signale des
   vulnérabilités dans les dépendances de build (Vite, `source-map-js`). Elles restent dans
   l'étape `build`, jetée, et n'atteignent pas l'image finale ; l'application n'est pas
   modifiée, conformément à la consigne.
6. `apk upgrade` rend le build non strictement reproductible ; à remplacer par une base
   patchée en production. La variante DHI n'a pas ce défaut mais impose un rebuild à
   chaque publication de tag.
7. Bases non épinglées par digest (`@sha256:…`) et `# syntax=docker/dockerfile:1` flottant :
   un tag peut être republié. En production, épingler par digest
   (`docker buildx imagetools inspect <image>`).
8. Identifiants Docker Hub stockés en base64 dans `~/.docker/config.json` (aucun
   `credsStore`) : à remplacer par un credential helper. Hors dépôt, rien n'a fui.
9. L'image DHI n'a pas de `HEALTHCHECK` (pas de `wget`) ; la supervision doit venir de
   l'orchestrateur.

**Commandes de scan (reproductibles).**

```bash
docker save quiz-docker:secure -o /tmp/secure.tar
docker run --rm -v /tmp:/in:ro aquasec/trivy:latest \
  image --input /in/secure.tar --scanners vuln --format table
```

Le scan ne monte pas le socket Docker : l'archive est passée en lecture seule, ce qui
respecte la recommandation du TP.

---

## 6. Livrables

| Livrable | Fichier |
|---|---|
| Dockerfile initial | `dockerfile-exercice2/Dockerfile.initial` (+ `Dockerfile.initial.dockerignore`, les 3 motifs d'origine) |
| Dockerfile sécurisé | `dockerfile-exercice2/Dockerfile` |
| Dockerfile DHI | `dockerfile-exercice2/Dockerfile.dhi` |
| `.dockerignore` | `dockerfile-exercice2/.dockerignore` |
| Scans avant | `dockerfile-exercice2/securite/scans/initial.txt` |
| Scans après | `dockerfile-exercice2/securite/scans/secure.txt` |
| Scan DHI | `dockerfile-exercice2/securite/scans/dhi.txt` |
| Scans intermédiaires | `dockerfile-exercice2/securite/scans/secure-sans-upgrade.txt`, `base129.txt`, `npm-audit.txt` |
| Scan base DHI seule | `dockerfile-exercice2/securite/scans/base130.txt` |
| Rapport | ce fichier |
| Images publiées | `cavydev/quiz-docker:initial`, `:secure`, `:dhi` |
| Dépôt Git | https://github.com/adam-benyekkou/docker-introduction-adam (branche `docker-exercice-securite`) |
