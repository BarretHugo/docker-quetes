# Quête 1 – Découverte de Docker

Premier challenge du fil rouge `demo-api` : pas encore de code, juste une base PostgreSQL qui tourne dans un conteneur.

## Journal de bord

**1. Lancer la base**

J'ai récupéré l'image officielle puis lancé le conteneur en arrière-plan avec les trois variables d'environnement demandées :

```bash
docker pull postgres:16-alpine
docker run -d --name demo-db -e POSTGRES_USER=demo -e POSTGRES_PASSWORD=demo -e POSTGRES_DB=demo postgres:16-alpine
```

J'avais déjà une ancienne version de `postgres:16-alpine` sur ma machine (d'un autre projet), le `pull` a récupéré la plus récente.

**2. Vérifier qu'elle tourne**

```
$ docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS         PORTS      NAMES
7ae96b9a0bfa   postgres:16-alpine   "docker-entrypoint.s…"   8 seconds ago   Up 8 seconds   5432/tcp   demo-db
```

Dans `docker logs demo-db`, j'ai remarqué que la ligne `database system is ready to accept connections` apparaît **deux fois**. En regardant les logs de plus près : au premier démarrage, l'image lance un serveur temporaire pour initialiser la base (création de l'utilisateur `demo` et de la base `demo`), l'arrête, puis démarre le vrai serveur. C'est la deuxième ligne (PID 1) qui compte.

**3. Entrer dans la base avec psql**

```bash
docker exec -it demo-db psql -U demo -d demo
```

Pas besoin de `-p` : on passe par `docker exec`, donc directement dans le conteneur, pas par le réseau. Et pas besoin d'installer psql sur Windows, il est déjà dans l'image.

**4. Créer la table et insérer une ligne**

```sql
CREATE TABLE products (id serial primary key, name text, price_cents int);
INSERT INTO products (name, price_cents) VALUES ('Sticker Démo', 150);
SELECT * FROM products;
\dt
```

**5. Nettoyer**

```bash
docker stop demo-db && docker rm demo-db
```

Rien n'étant monté en volume, la table disparaît avec le conteneur. Je suppose que c'est ce qu'on corrigera dans la quête sur les volumes.

## Sorties demandées

`\dt` :

```
         List of relations
 Schema |   Name   | Type  | Owner
--------+----------+-------+-------
 public | products | table | demo
(1 row)
```

`SELECT * FROM products;` :

```
 id |     name     | price_cents
----+--------------+-------------
  1 | Sticker Démo |         150
(1 row)
```

3 dernières lignes de `docker logs demo-db` :

```
2026-10-06 07:06:42.557 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2026-10-06 07:06:42.560 UTC [57] LOG:  database system was shut down at 2026-10-06 07:06:42 UTC
2026-10-06 07:06:42.564 UTC [1] LOG:  database system is ready to accept connections
```
