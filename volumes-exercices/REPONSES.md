# TP Volumes Docker — Compte rendu

Toutes les sorties ci-dessous sont réelles (25/11/2026). Version détaillée :
`REPONSES-detail.md`.

## L'essentiel en trois lignes

| Montage       | Où vivent les données                             | Survivent à `docker rm` |
| ------------- | ------------------------------------------------- | ----------------------- |
| Aucun montage | couche inscriptible du conteneur                  | **non**                 |
| Volume nommé  | `/var/lib/docker/volumes/<nom>` (géré par Docker) | **oui**                 |
| Bind Mount    | un dossier de ton choix sur l'hôte                | **oui**                 |

---

## Exercices 1 à 4 — volumes

| Ex  | Commande clé                                         | Résultat observé                             |
| --- | ---------------------------------------------------- | -------------------------------------------- |
| 1   | écrire dans `/tmp`, puis supprimer le conteneur      | `Donnee-temporaire`, puis **`ABSENT`**       |
| 2   | `docker volume create tp-volume`                     | volume créé, `Driver=local`, `Scope=local`   |
| 3   | écrire, `docker rm`, relire depuis un conteneur neuf | **`Donnee-persistante`**                     |
| 4   | écrivain + lecteur `readonly` sur `tp-partage`       | `Message-du-conteneur-A` → `Message-modifie` |

**1.** Le conteneur supprimé emporte sa couche inscriptible. Même image ≠ même
conteneur : l'image est en lecture seule et partagée, la couche est propre à
chacun. Problématique dès que la donnée n'est pas reproductible : uploads, base,
journaux.

**2.** Le volume existe sans conteneur. `local` = stockage sur le disque du
démon ; `Mountpoint` = le chemin réel, ici
`/var/lib/docker/volumes/tp-volume/_data` ; `Scope=local` = limité à cette
machine. Sur Docker Desktop, cet emplacement est *dans la VM Linux*, donc
invisible depuis le Finder ou l'Explorateur.

**3.** C'est le **volume** qui porte la durée de vie, pas le conteneur.

**4.** Les deux conteneurs lisent le même fichier en temps réel. `readonly`
protège le volume d'un lecteur compromis — vérifié avec
`sh: can't create /data/x.txt: Read-only file system` (exit 1). Écritures
simultanées = écrasement mutuel, lecture partielle, corruption : deux PostgreSQL
sur un même `PGDATA` se refusent.

---

## Exercices 5 et 6 — bind mount

```sh
docker run -d --name tp-nginx-bind -p 8085:80 \
  --mount "type=bind,source=$(pwd),target=/usr/share/nginx/html,readonly" nginx:alpine
```

- Avant édition : `Version 1 — Bonjour Docker !`
- Après édition du fichier sur l'hôte, la page affiche
  `Version 2 — Fichier modifié sur mon ordinateur`, **sans recréer le
  conteneur**.
- `docker inspect` : `Type=bind`, `RW=false`. Écriture depuis le conteneur →
  `Read-only file system`.
- Après `docker rm -f`, `index.html` est toujours sur l'hôte.

La page change parce que Nginx lit *ton* dossier : il n'y a qu'un exemplaire des
fichiers. Sans montage (port 8086), Nginx sert les fichiers de l'image :
`Welcome to nginx!`.

| Situation                           | Choix             | Pourquoi                                  |
| ----------------------------------- | ----------------- | ----------------------------------------- |
| Persister une base de données       | **Volume nommé**  | processus interne, chemin géré par Docker |
| Voir les modifs d'un site local     | **Bind Mount**    | sources sur l'hôte, aucun rebuild         |
| Application temporaire sans données | **Aucun montage** | la couche inscriptible suffit             |
| Garder les données à la recréation  | **Volume nommé**  | donnée découplée du conteneur             |

**Volume ou bind mount ?** Même durabilité ; la différence est qui possède le
chemin — Docker, ou toi.

**Persistance ≠ sauvegarde.** Une seule copie, sur un seul disque, sans
historique. `docker volume rm`, un disque mort ou un `DROP TABLE` effacent tout.
Une sauvegarde = copie séparée + restauration testée.

---

## Exercice 7 — suppression ciblée

```sh
docker volume rm tp-nettoyage
docker volume inspect tp-nettoyage   # Error: no such volume   (exit 1, attendu)
docker volume create tp-nettoyage    # volume NEUF et VIDE : important.txt a disparu
```

Le nom est une étiquette, pas une identité. `docker volume prune --all` est
risqué parce qu'il vise **tous** les volumes non montés, tous projets confondus —
dont un volume de production dont le conteneur est arrêté pour maintenance.
Avant de supprimer : sauvegarde testée, conteneurs consommateurs identifiés,
arrêt propre, export des données.

---

## Atelier — PostgreSQL

```sh
docker volume create tp-postgres-data
docker run -d --name tp-postgres \
  -e POSTGRES_USER=tpuser -e POSTGRES_PASSWORD='TpDocker2026!' -e POSTGRES_DB=tpdb \
  --mount type=volume,source=tp-postgres-data,target=/var/lib/postgresql/data \
  postgres:17                        # aucun port publié
```

```sh
docker exec tp-postgres psql -U tpuser -d tpdb -c "CREATE TABLE apprenants (...);"
docker exec tp-postgres psql -U tpuser -d tpdb -c "INSERT INTO apprenants ...;"
```

**Maintenance.** `docker rm -f tp-postgres`, le volume `tp-postgres-data` reste,
puis recréation avec les mêmes paramètres :

```text
 id |  nom  | formation
----+-------+-----------
  1 | Alice | Docker
  2 | Bilal | DevOps
  3 | Chloe | Cloud
(3 rows)
```

**Réussite : les trois lignes ont survécu.**

1. **Pourquoi ?** `/var/lib/postgresql/data` est un point de montage vers le
   volume. `docker rm` ne touche pas au volume.
2. **Volume vide neuf ?** `initdb` recrée un cluster vierge : la base et la table
   disparaissent, **sans aucune erreur**. Perte silencieuse, le pire cas.
3. **Pourquoi pas une sauvegarde ?** Une seule copie, un seul support :
   suppression, corruption ou panne disque sont définitives. Il faut `pg_dump`
   vers un stockage distinct, avec rétention.
4. **Changement de version majeure ?** Le format de `PGDATA` change : PostgreSQL
   refuse de démarrer (protection, pas un bug). Il faut `pg_upgrade` ou un
   dump/restore, et le retour arrière est impossible. Attention aussi au point de
   montage déplacé (`/var/lib/postgresql`) sur les images récentes.
5. **Bind mount à la place ?** Quand l'hôte doit lire les fichiers : édition de
   `postgresql.conf`, import d'un dump, script de sauvegarde local, NFS existant.
   Le prix : UID (999 pour `postgres`), chemin absolu non portable, risque
   d'écraser le contenu.

**Piège observé.** Au premier démarrage, `pg_isready` répond
`rejecting connections` pendant environ 7 s, le temps que `initdb` redémarre le
serveur. Un test unique au démarrage conclut à tort à un échec ; une boucle de
retry est nécessaire.

---

## Nettoyage (non exécuté — détruit les données)

```sh
docker rm -f tp-postgres
docker volume rm tp-volume tp-partage tp-nettoyage tp-postgres-data
```
