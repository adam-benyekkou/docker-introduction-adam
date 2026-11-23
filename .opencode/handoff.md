# Handoff

## Done (verified)
- `dockerfile-exercice2/Dockerfile` réécrit avec commentaires (node:18-alpine, npm ci, CMD vite --host 0.0.0.0), plus `dockerfile-exercice2/.dockerignore`.
- `docker build -t cavydev/dockerfile-exercice2:1.0.0 .` → exit 0
- `docker run -p 8082:5173` + `curl localhost:8082/` → HTTP 200 (exit 0), `/main.js` → HTTP 200
- Tags `1.0.0`, `1.0`, `latest` poussés : `docker push --all-tags cavydev/dockerfile-exercice2` → exit 0, digest `sha256:fa0887b7...ad1b`
- Déploiement depuis le registre : `docker rmi` des tags locaux puis `docker pull cavydev/dockerfile-exercice2:1.0.0` → exit 0, run sur 8083 + `curl` → HTTP 200
- Commit `e7e5fd9` sur la branche `dockerfile-exercice2`.

## Blocked
- `git push -u origin dockerfile-exercice2` → « Blocked by configuration policy » (deux tentatives). Le commit reste local ; la branche n'existe pas encore sur origin.

## Next
- Pousser la branche depuis un terminal non restreint : `git push -u origin dockerfile-exercice2`.

## Files
- dockerfile-exercice2/Dockerfile
- dockerfile-exercice2/.dockerignore

## Landmines
- Host port 8082/8083 choisis à `docker run -p`, pas dans le Dockerfile (5173 exposé dans l'image).
- Image finale ~221 Mo : serveur Vite de dev, pas de build statique (choix « déploiement simple »).
- Conteneur `ex2c` laissé tournant sur le port 8083.

## Open questions
- None.
