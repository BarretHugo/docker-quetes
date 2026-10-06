# Quête 2 – Le Dockerfile

Fil rouge, étape 1 : écrire le Dockerfile de `demo-api` pour que l'API tourne enfin dans un conteneur.

- Repo : [BARRET_hugo_demo-api](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api)
- Image publiée : [hugobarret/demo-api:1.0](https://hub.docker.com/r/hugobarret/demo-api) sur Docker Hub

## Journal de bord

**1. Récupérer le starter**

J'ai cloné le starter du formateur sous le nom `BARRET_hugo_demo-api` et supprimé le remote `origin` qui pointait encore vers le starter.

**2. Écrire le Dockerfile**

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY server.js db.js ./

EXPOSE 3000

CMD ["node", "server.js"]
```

J'ai pris `node:22-alpine` : version épinglée et image légère. Les dépendances sont copiées et installées **avant** le code, comme ça `npm ci` reste en cache tant que je ne touche qu'au code.

Pour le `.dockerignore`, j'exclus `node_modules`, `.git`, les `.md`, les `.env*`, les logs et le Dockerfile lui-même. Le plus important pour moi : ne pas envoyer un `node_modules` local ou un `.env` dans le contexte de build.

**3. Build et test**

```bash
docker build -t demo-api:1.0 ./api
docker run -d --name api -p 8080:3000 demo-api:1.0
```

```
$ curl -s localhost:8080/health
{"status":"UP"}
$ curl -s localhost:8080/
{"ok":true,"app":"demo-api","version":"dev"}
```

`/products` répond 503 (après environ 3 secondes, le temps que la connexion échoue), ce qui est normal puisqu'il n'y a pas encore de base. On le voit aussi dans `docker logs api` :

```
{"level":"info","msg":"demo-api started","port":3000,"version":"dev"}
{"level":"info","method":"GET","path":"/health","status":200,"ms":4}
{"level":"info","method":"GET","path":"/","status":200,"ms":0}
{"level":"info","method":"GET","path":"/products","status":503,"ms":3005}
```

**4. Vérifier le cache**

J'ai ajouté une ligne de commentaire dans `server.js` puis relancé le build avec `--progress=plain`. Seule la dernière couche (`COPY server.js db.js`) est refaite, `npm ci` est bien en cache :

```
#6 [2/5] WORKDIR /app
#6 CACHED
#7 [3/5] COPY package.json package-lock.json ./
#7 CACHED
#8 [4/5] RUN npm ci --omit=dev
#8 CACHED
#9 [5/5] COPY server.js db.js ./
#9 DONE 0.0s
```

**5. Taille de l'image**

```
$ docker image ls demo-api
IMAGE          ID             DISK USAGE   CONTENT SIZE   EXTRA
demo-api:1.0   1d302373bdae        252MB         63.2MB
```

63 Mo compressés, 252 Mo une fois décompressés sur le disque. L'essentiel vient de l'image Node elle-même, il y a de quoi gagner avec un build multi-étapes (prévu dans une prochaine quête).

**6. Publication**

J'ai choisi Docker Hub. Après m'être connecté avec mon compte dans Docker Desktop :

```bash
docker tag demo-api:1.0 hugobarret/demo-api:1.0
docker push hugobarret/demo-api:1.0
```

```
The push refers to repository [docker.io/hugobarret/demo-api]
e2de96513ba9: Mounted from library/nginx
d39db1cf9caa: Pushed
...
1.0: digest: sha256:1d302373bdae629dcee4ffb52ba3ebe329745b6719ab2ae9394724d8ed6456b9 size: 856
```

Marrant : une couche a été « Mounted from library/nginx » au lieu d'être envoyée. Il s'agit sûrement de la couche de base Alpine, la même que celle de `nginx:alpine` (j'ai retrouvé le même identifiant `e2de96513ba9` quand j'avais pull nginx dans la quête 1). Docker Hub l'avait déjà, donc pas besoin de la renvoyer. C'est le partage de layers vu dans le cours.

Le repo est poussé dans l'organisation : `ynov-x-anthony/BARRET_hugo_demo-api`.

Petit souci en route : mon PC a planté pendant la quête. Il a fallu relancer Docker Desktop, mais l'image `demo-api:1.0` et mes fichiers étaient toujours là.
