# Quête 3 – Dockerfile et sécurité

Fil rouge : durcir l'image `demo-api` et la façon de la lancer.

- Repo : [BARRET_hugo_demo-api](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api) (commit `c27f940`)

## Journal de bord

**1. Durcir le Dockerfile**

```dockerfile
FROM node:22.11-alpine

WORKDIR /app

COPY --chown=node:node package.json package-lock.json ./
RUN npm ci --omit=dev

COPY --chown=node:node server.js db.js ./

HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

USER node

EXPOSE 3000

CMD ["node", "server.js"]
```

Ce qui change par rapport à la quête 2 :
- `node:22.11-alpine` au lieu de `node:22-alpine` : `22` bouge à chaque nouvelle version mineure, `22.11` non.
- `--chown=node:node` sur les `COPY`, puis `USER node` juste avant le `CMD`. L'ordre compte : une fois en `node`, on ne peut plus modifier ce qui appartient à root.
- Un `HEALTHCHECK` sur `/health`, en Node directement puisqu'il n'y a pas `curl` dans l'image.

Le `.dockerignore` de la quête 2 excluait déjà `.git`, `.env*`, `node_modules` et `*.md`, je n'y ai pas touché.

**2. Vérifier le non-root**

```
$ docker run --rm demo-api:hardened id
uid=1000(node) gid=1000(node) groups=1000(node),1000(node)
```

**3. Lancer l'API durcie**

Il faut d'abord un réseau et une base dessus, sinon `--network demo_net` échoue :

```bash
docker network create demo_net
docker run -d --name demo-db --network demo_net -e POSTGRES_USER=demo -e POSTGRES_PASSWORD=demo -e POSTGRES_DB=demo postgres:16-alpine
```

Puis l'API, sur une seule ligne car les `\` de fin de ligne ne marchent pas dans PowerShell :

```bash
docker run -d --name api -p 8080:3000 --read-only --tmpfs /tmp:size=16m --cap-drop ALL --security-opt no-new-privileges --pids-limit 200 --memory 256m --cpus 1 --network demo_net -e PGHOST=demo-db demo-api:hardened
```

Pas besoin de passer user / mot de passe / base : `db.js` prend `demo` par défaut, comme le postgres.

**4. Les preuves**

```
$ curl -s localhost:8080/health
{"status":"UP"}

$ curl -s localhost:8080/ready
{"status":"READY"}

$ docker exec api sh -c "touch /app/x 2>&1 || echo 'rootfs read-only OK'"
touch: /app/x: Read-only file system
rootfs read-only OK

$ docker inspect -f 'readonly={{.HostConfig.ReadonlyRootfs}} capdrop={{.HostConfig.CapDrop}}' api
readonly=true capdrop=[ALL]

$ docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS                    PORTS                                         NAMES
d056d99fce98   demo-api:hardened    "docker-entrypoint.s…"   16 seconds ago   Up 15 seconds (healthy)   0.0.0.0:8080->3000/tcp, [::]:8080->3000/tcp   api
0061e811a880   postgres:16-alpine   "docker-entrypoint.s…"   34 seconds ago   Up 34 seconds             5432/tcp                                      demo-db
```

`/ready` en `READY` montre que l'API joint bien la base par son nom `demo-db` sur le réseau, même avec toutes les restrictions.

Pour le test d'écriture, j'ai fait attention au message : `Read-only file system` veut dire que c'est bien le `--read-only` qui bloque. Si ça avait été `Permission denied`, ça aurait juste été l'utilisateur `node` qui n'a pas les droits sur `/app`.

**Galère PowerShell** : la commande de la quête, `sh -c 'touch ... || echo "rootfs read-only OK"'`, n'affichait que `rootfs`. PowerShell 5.1 casse les guillemets doubles à l'intérieur des simples quand il les passe à `docker`. En inversant (doubles dehors, simples dedans), ça passe.
