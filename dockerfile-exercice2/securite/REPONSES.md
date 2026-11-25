# Sécurisation d'un conteneur Docker — quiz-docker (exercice 2)

- Dépôt : https://github.com/adam-benyekkou/docker-introduction-adam (branche `docker-exercice-securite`)
- Image initiale : `cavydev/quiz-docker:initial` (ex-`cavydev/dockerfile-exercice2:1.0.0`)
- Image sécurisée : `cavydev/quiz-docker:secure`
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
| `.dockerignore` minimal | 3 motifs seulement : un `.env` ou `*.pem` entre dans l'image via `COPY . .` |
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
| System de fichiers racine | inscriptible |
| Socket Docker | non monté |
| CVE CRITICAL | **5** |
| CVE HIGH | **83** |
| CVE MEDIUM | **130** |
| CVE LOW | **45** |
| Total CVE | **264** |

Détail Trivy (`securite/scans/initial.txt`) : Alpine 3.21.3 → 100 (2C/22H/41M/35L),
paquets Node → 75 (1C/32H/36M/6L), npm → 89 (2C/29H/53M/4L).

---

## 2. Mission 2 — Recommandations appliquées

| Recommandation | Niveau | Mise en œuvre | Preuve |
|---|---|---|---|
| Image de base minimale | Dockerfile | runtime `nginxinc/nginx-unprivileged:1.29-alpine3.23`, seulement `dist/` copié | `docker exec qs command -v node npm` → absent |
| Ne pas tourner en root | Dockerfile | `USER 101` (image non privilégiée) | `docker exec qs id` → `uid=101(nginx)` |
| `.dockerignore` | Build | secrets, artefacts, `.git`, outillage, livrables exclus | `.dockerignore` (11 groupes de motifs) |
| Tags d'images précis | Dockerfile | `node:22.22-alpine3.24` et `nginxinc/nginx-unprivileged:1.29-alpine3.23` | `grep '^FROM' Dockerfile` |
| `COPY` plutôt que `ADD` | Dockerfile | uniquement des `COPY` explicites, aucune URL ni archive | `grep -c '^ADD' Dockerfile` → 0 |
| Limiter ressources et privilèges | Exécution | `--memory 128m --cpus 0.5 --pids-limit 100 --cap-drop ALL --security-opt no-new-privileges --read-only` | `docker inspect` : `Memory=134217728`, `NanoCpus=500000000`, `PidsLimit=100`, `CapDrop=[ALL]`, `ReadonlyRootfs=true` |
| Build multi-étapes | Dockerfile | étape `build` (Node) puis étape runtime (nginx) | `docker history quiz-docker:secure` ne contient ni `npm ci` ni sources |
| Pas de socket Docker exposé | Exécution | aucun `-v /var/run/docker.sock`, scans via `docker save` + `trivy image --input` | `docker inspect -f '{{len .Mounts}}' qs` → `0` |
| Vérifier régulièrement | Processus | scans Trivy versionnés dans `securite/scans/`, commande documentée §5 | `securite/scans/*.txt` |

**Correctifs de paquets.** La base épinglée peut être en retard sur des CVE déjà corrigées
en amont. Le Dockerfile applique donc `apk upgrade --no-cache` dans l'étape runtime
(bloc `USER root` limité à cette couche). Effet mesuré : **248 → 0 vulnérabilité**. En
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
- Image DHI : étudiée en mission 4 et non retenue comme base par défaut ici, car elle
  exige un accès au registre `dhi.io`.

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
  -p 8080:8080 quiz-docker:secure

# Publication
docker tag quiz-docker:initial cavydev/quiz-docker:initial
docker tag quiz-docker:secure  cavydev/quiz-docker:secure
docker push cavydev/quiz-docker:initial
docker push cavydev/quiz-docker:secure
```

### 3.2 Tableau comparatif

| Critère | Initial | Sécurisé |
|---|---|---|
| Image de base | `node:18-alpine` (Alpine 3.21.3) | `nginxinc/nginx-unprivileged:1.29-alpine3.23` (Alpine 3.23.4) |
| Taille | 221 MB | **118 MB** (−47 %) |
| Utilisateur | root (uid 0) | nginx (uid 101) |
| Outils de développement | node, npm, apk, sources, `node_modules` | absents (seul `dist/` est copié) |
| Serveur applicatif | serveur de dev Vite | nginx (statique) |
| Port | 5173 | 8080 |
| `.dockerignore` | 3 motifs | secrets, artefacts, `.git`, outillage, livrables exclus |
| Multi-stage | non | oui (2 étapes) |
| `ADD` | non | non (`COPY` uniquement) |
| Tags précis | non (`18-alpine`) | oui (versions Node/Alpine/nginx fixées) |
| Limitation CPU/mémoire | aucune | 0.5 CPU, 128 Mo, 100 pids |
| Restrictions de privilèges | aucune | `--cap-drop ALL`, `no-new-privileges` |
| Racine en lecture seule | non | oui (`--read-only` + `tmpfs`) |
| Accès socket Docker | non | non, prouvé (`Mounts` = 0) |
| CVE CRITICAL | 5 | **0** |
| CVE HIGH | 83 | **0** |
| CVE MEDIUM | 130 | **0** |
| CVE LOW | 45 | **0** |
| Total CVE | 264 | **0** |

Sources : `securite/scans/initial.txt`, `securite/scans/secure.txt`.

### 3.3 Preuves d'exécution

| Vérification | Résultat |
|---|---|
| `curl -sI localhost:8080` | `HTTP/1.1 200 OK`, `Server: nginx/1.29.8` |
| Santé du conteneur après 15 s | `Up (healthy)` |
| `docker exec qs id` | `uid=101(nginx) gid=101(nginx)` |
| `docker exec qs command -v node npm` | `absent` |
| `docker inspect` | `ReadonlyRootfs=true`, `CapDrop=[ALL]`, `Memory=134217728`, `NanoCpus=500000000`, `PidsLimit=100`, `Mounts=0` |
| `docker run --rm quiz-docker:secure ls /var/run/docker.sock` | `No such file or directory` |
| `docker push` | `initial` et `secure` poussées (exit 0) |

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
- l'image n'est ni signée ni accompagnée d'un SBOM vérifiable ;
- l'image runtime contient un shell (busybox), utile à un attaquant qui obtient une
  exécution de code ;
- l'exposition réseau reste HTTP en clair, sans TLS, sans limitation de débit ni WAF ;
- la configuration du démon Docker et de l'hôte n'est pas couverte.

**Mesures complémentaires pour un déploiement en production**
1. CI qui bloque sur CVE CRITICAL/HIGH (Trivy + `--exit-code`), avec rebuild
   automatique lors de la publication d'une base patchée.
2. SBOM (Syft/Trivy) et signature d'image (cosign), vérification à l'admission.
3. Runtime sans shell (distroless ou Docker Hardened Image, cf. mission 4).
4. `securityContext` Kubernetes équivalent : non-root, `readOnlyRootFilesystem`,
   `allowPrivilegeEscalation: false`, `seccompProfile: RuntimeDefault`.
5. Politiques réseau, TLS en terminaison, limitation de débit, WAF.
6. Journalisation centralisée et détection runtime (Falco).
7. Gestion des secrets hors image (variables d'environnement, secret manager).

---

## 4. Mission 4 — Docker Hardened Image (DHI)

**Statut : à compléter après `docker login dhi.io`.**

Atelier de référence : https://docs.docker.com/guides/lab-dhi-node/

Étapes prévues une fois l'accès au registre obtenu :

1. Construire une variante `quiz-docker:dhi` en remplaçant l'étape de build par
   `dhi.io/node:<tag>` et l'étape runtime par `dhi.io/nginx:<tag>` (ou l'image DHI
   statique correspondante), sans modifier l'application.
2. Vérifier : utilisateur par défaut, absence de shell, taille, entrées SBOM/provenance.
3. Scanner avec la même méthode (`docker save` + `trivy image --input`) et reporter les
   compteurs par sévérité dans le tableau §3.2.
4. Comparer les propriétés : base distroless, CVE, signature, SBOM, politique de
   support, coût de migration, compatibilité avec l'écosystème npm.
5. Conclure sur l'opportunité d'utiliser DHI en production pour cette application.

Aucun résultat DHI n'est présenté ici : rien n'a été mesuré, donc rien n'est affirmé.

---

## 5. Risques résiduels et synthèse

**Améliorations apportées.** Image deux fois plus légère, exécution non-root sans
capabilities, racine en lecture seule, aucune CVE détectée au build, aucun outil de
développement dans l'image finale, socket Docker jamais exposé, surface réseau réduite
à un serveur statique.

**Risques résiduels.**
1. Résultat de scan daté : la base doit être reconstruite à chaque correctif amont.
2. Pas de signature d'image ni de SBOM vérifiable.
3. Shell présent dans l'image runtime.
4. HTTP en clair, sans limitation de débit.
5. Chaîne de build npm non auditée (dépendances de build uniquement).
6. `apk upgrade` rend le build non strictement reproductible ; à remplacer par une base
   patchée en production.

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
| Dockerfile initial | `dockerfile-exercice2/Dockerfile.initial` |
| Dockerfile sécurisé | `dockerfile-exercice2/Dockerfile` |
| `.dockerignore` | `dockerfile-exercice2/.dockerignore` |
| Scans avant | `dockerfile-exercice2/securite/scans/initial.txt` |
| Scans après | `dockerfile-exercice2/securite/scans/secure.txt` |
| Rapport | ce fichier |
| Images publiées | `cavydev/quiz-docker:initial`, `cavydev/quiz-docker:secure` |
| Dépôt Git | https://github.com/adam-benyekkou/docker-introduction-adam (branche `docker-exercice-securite`) |
