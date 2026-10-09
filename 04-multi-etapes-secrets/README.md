# Quête 4 – Builds multi-étapes et gestion des secrets

Fil rouge, étape 6 : passer `demo-api` en multi-étapes et prouver qu'un secret de build ne fuit pas.

- Fichiers : [api/Dockerfile.multi](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/blob/main/api/Dockerfile.multi) et [api/Dockerfile.naive](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/blob/main/api/Dockerfile.naive) (commit `4b2d14f`)

## Journal de bord

**1. Deux Dockerfile**

`Dockerfile.naive`, le repère volontairement lourd :

```dockerfile
FROM node:22
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]
```

`Dockerfile.multi`, la version en deux étapes :

```dockerfile
# syntax=docker/dockerfile:1

FROM node:22.11-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci --omit=dev

FROM node:22.11-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node server.js db.js package.json ./
HEALTHCHECK ... (même sonde /health que la quête 3)
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

L'étape `runtime` repart d'un `FROM` neuf : elle ne récupère de `deps` que `node_modules`. La ligne du secret, la quête disait de l'« ajouter », mais je l'ai mise directement à la place du `RUN npm ci` pour ne pas installer deux fois. Sans `--secret` au build, elle marche quand même (le secret est optionnel par défaut).

**2. Comparaison**

```
$ docker image ls demo-api
IMAGE               ID             DISK USAGE   CONTENT SIZE
demo-api:multi      40a4bbc55a9d        228MB         55.1MB
demo-api:naive      db3029481344       1.65GB          413MB
```

| Image | Disque | Contenu |
|---|---|---|
| naive | 1.65 GB | 413 MB |
| multi | 228 MB | 55.1 MB |
| Ratio | ÷ 7,2 | ÷ 7,5 |

Le gros du gain vient de la base (`node:22` Debian complet contre `alpine`), puis des dev-deps et du `COPY . .`.

Par contre, `multi` fait quasiment la même taille que mon `demo-api:hardened` de la quête 3 (232 MB) : ce Dockerfile était déjà en alpine avec `--omit=dev` et une copie minimale. Le multi-étapes sert surtout quand il y a une étape de build (TypeScript, compilation…), ce qui n'est pas le cas de demo-api. C'est d'ailleurs pour ça que la quête fait comparer à un `naive`.

**3. Le secret au build**

J'ai créé un faux fichier avec `//registry.npmjs.org/:_authToken=FAKE-123`. Pas dans `~/.npmrc` comme le propose la quête : un faux token là-dedans risquait de casser mes vrais `npm install` plus tard. Je l'ai mis dans un fichier à part, supprimé à la fin.

```powershell
docker build --no-cache -f api/Dockerfile.multi --secret id=npmrc,src="$HOME\fake.npmrc" -t demo-api:multi ./api
```

Le `--no-cache` est important : sans lui, l'étape `npm ci` était reprise du cache (`CACHED`), donc le secret n'était même pas monté et la preuve n'aurait rien prouvé. Avec, on voit `[deps 4/4] RUN --mount=type=secret,...` qui tourne vraiment (2 s).

**4. Les preuves**

```
$ docker history --no-trunc demo-api:multi | Select-String "FAKE-123"
(aucune ligne)

$ docker run --rm -u root demo-api:multi sh -c 'cat /root/.npmrc 2>&1'
cat: can't open '/root/.npmrc': No such file or directory
```

Le `-u root`, c'est parce que l'image tourne en `node` : sans lui, on obtient `Permission denied` (`/root` est en `drwx------`), ce qui ne prouve pas que le fichier n'existe pas.

**5. L'image tourne**

```
$ curl -s localhost:8080/health
{"status":"UP"}
```

Au premier essai, `curl` ne renvoyait rien : je l'avais lancé juste après le `docker run`, Node n'avait pas fini de démarrer. Avec 3 secondes d'attente, c'est bon.

Petit souci en route : mon PC a encore planté pendant la quête, il a fallu relancer Docker Desktop. Les images et les fichiers étaient toujours là.
