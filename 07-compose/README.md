# Quête 7 – Compose

Fil rouge, étape 4 : toute la stack (api + db + adminer) dans un `compose.yml`.

- Repo : [BARRET_hugo_demo-api](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api) (commit `72db3d3`) : `compose.yml`, `.env.example`, `README.md`

## Journal de bord

**1. Ce que j'ai repris des quêtes d'avant**

Compose, c'est en fait tout ce qu'on a fait à la main dans les quêtes 5 et 6, mais dans un seul fichier :
- le volume `pgdata` (quête 5) ;
- les réseaux `front` / `back`, avec la base seulement sur `back` et sans `ports:` (quête 6). La quête n'en demandait pas, le réseau par défaut aurait suffi, mais je les ai gardés ;
- le bind mount de `db/init.sql` ;
- le `pg_isready -h 127.0.0.1` de la quête 5, cette fois dans le `healthcheck`.

**2. Le mot de passe en secret (le bonus)**

C'est la partie qui m'a demandé le plus de réflexion. Pour postgres c'est simple, l'image comprend `POSTGRES_PASSWORD_FILE=/run/secrets/db_password`. Mais l'API (`db.js`) attend `PGPASSWORD` en variable d'environnement, et je ne voulais pas toucher au code de l'API (le but est que tout se règle côté Docker).

Solution : surcharger la commande de l'API pour qu'elle lise le secret au démarrage :

```yaml
command: ["sh", "-c", "PGPASSWORD=\"$$(cat /run/secrets/db_password)\" exec node server.js"]
```

- `$$` : sinon Compose essaie d'interpoler `$(...)` lui-même.
- `exec` : `sh` est remplacé par `node`, qui reste PID 1 et reçoit bien le SIGTERM au `docker compose down`. Vérifié : `cat /proc/1/cmdline` donne `node server.js`.

Pour être sûr que le secret est vraiment utilisé, j'ai mis un mot de passe aléatoire de 24 caractères dans `secrets/db_password.txt` au lieu de `demo`. Si l'API se connecte, c'est qu'elle a lu le secret et pas sa valeur par défaut `demo`. Et `/ready` répond bien `READY`.

Le mot de passe n'apparaît nulle part dans `docker inspect` :

```
POSTGRES_USER=demo
POSTGRES_PASSWORD_FILE=/run/secrets/db_password
POSTGRES_DB=demo
```

`.env` et `secrets/` sont dans le `.gitignore`. Seul `.env.example` est commité (sans mot de passe, puisqu'il est dans le secret).

**3. Lancement**

```
$ docker compose up -d --build
 Container demo-api-db-1  Started
 Container demo-api-db-1  Waiting
 Container demo-api-db-1  Healthy
 Container demo-api-api-1  Starting
 Container demo-api-api-1  Started
```

On voit le `depends_on: condition: service_healthy` en action : l'API attend que la base soit `Healthy` avant de démarrer.

```
$ docker compose ps
NAME                 IMAGE                SERVICE   STATUS                    PORTS
demo-api-adminer-1   adminer:4            adminer   Up 22 seconds             0.0.0.0:8081->8080/tcp
demo-api-api-1       demo-api-api         api       Up 16 seconds (healthy)   0.0.0.0:8080->3000/tcp
demo-api-db-1        postgres:16-alpine   db        Up 22 seconds (healthy)   5432/tcp
```

`api` est même `healthy` grâce au `HEALTHCHECK` de mon Dockerfile de la quête 3.

**4. La Gourde survit au down / up**

```
$ curl -s -X POST -H 'content-type: application/json' -d '{"name":"Gourde","price_cents":900}' localhost:8080/products
{"id":4,"name":"Gourde","price_cents":900,"created_at":"2026-10-09T08:07:20.294Z"}

$ docker compose down
$ docker compose up -d

$ curl -s localhost:8080/products
[{"id":4,"name":"Gourde","price_cents":900,"created_at":"2026-10-09T08:07:20.294Z"},{"id":3,"name":"T-shirt conteneur",...},...]
```

Même `created_at`, donc la même ligne. `down` supprime les conteneurs et les réseaux mais garde le volume. Avec `down -v`, la Gourde aurait disparu.

Adminer répond aussi : `curl -s localhost:8081` → `<title>Login - Adminer</title>`.

## Remarques

- `name: demo-api` en tête du `compose.yml` : sans ça, le projet aurait pris le nom du dossier (`barret_hugo_demo-api`), et les conteneurs auraient eu des noms à rallonge.
- Ici le curl avec `-d '{"name":"Gourde",...}'` passe sans souci depuis Git Bash, pas d'accent dans « Gourde » (contrairement à la casquette de la quête 5…).
