# 1 - Découverte de Docker

## Challenge : PostgreSQL dans un conteneur (`demo-db`)

### Commandes

```bash
docker pull postgres:16-alpine
docker run -d --name demo-db \
  -e POSTGRES_USER=demo \
  -e POSTGRES_PASSWORD=demo \
  -e POSTGRES_DB=demo \
  postgres:16-alpine

docker ps
docker logs demo-db

docker exec -it demo-db psql -U demo -d demo
```

```sql
CREATE TABLE products (id serial primary key, name text, price_cents int);
INSERT INTO products (name, price_cents) VALUES ('Sticker Démo', 150);
SELECT * FROM products;
\dt
\q
```

```bash
docker stop demo-db && docker rm demo-db
```

### Résultats

`docker ps` :

```
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS         PORTS      NAMES
7ae96b9a0bfa   postgres:16-alpine   "docker-entrypoint.s…"   8 seconds ago   Up 8 seconds   5432/tcp   demo-db
```

`SELECT * FROM products;` :

```
 id |     name     | price_cents
----+--------------+-------------
  1 | Sticker Démo |         150
(1 row)
```

`\dt` :

```
         List of relations
 Schema |   Name   | Type  | Owner
--------+----------+-------+-------
 public | products | table | demo
(1 row)
```

3 dernières lignes de `docker logs demo-db` :

```
2026-10-06 07:06:42.557 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2026-10-06 07:06:42.560 UTC [57] LOG:  database system was shut down at 2026-10-06 07:06:42 UTC
2026-10-06 07:06:42.564 UTC [1] LOG:  database system is ready to accept connections
```
