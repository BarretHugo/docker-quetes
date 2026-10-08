# Quête 5 – Les volumes

Fil rouge, étape 2 : faire en sorte que les données de la base survivent à la suppression du conteneur.

- Script : [volumes_hugo_barret.sh](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/blob/main/volumes_hugo_barret.sh) dans le repo `BARRET_hugo_demo-api`

## Journal de bord

**1. Le principe**

PostgreSQL écrit ses données dans `/var/lib/postgresql/data`. Sans volume, ce dossier est dans la couche du conteneur et disparaît avec `docker rm`. Avec `-v demo_pgdata:/var/lib/postgresql/data`, il est stocké dans un named volume géré par Docker, en dehors du conteneur.

**2. Le script**

Il enchaîne : build de l'API, création du réseau `demo_net` et du volume `demo_pgdata`, lancement de `demo-db` (volume + `init.sql` monté en lecture seule), lancement de l'API, ajout de « Casquette Démo », puis `docker rm -f demo-db`, recréation sur le même volume, et on revérifie `/products`.

Je l'ai lancé depuis Git Bash.

**3. Le résultat**

```
=== 5. Ajout d'un produit
{"id":4,"name":"Casquette Démo","price_cents":1200,"created_at":"2026-10-08T09:54:40.531Z"}
--- Produits AVANT suppression de la base :
[{"id":4,"name":"Casquette Démo",...},{"id":3,"name":"T-shirt conteneur",...},{"id":2,"name":"Mug Docker",...},{"id":1,"name":"Sticker Demo",...}]
=== 6. Suppression du conteneur demo-db puis recréation sur le même volume
Attente de PostgreSQL OK
--- Produits APRÈS recréation de la base :
[{"id":4,"name":"Casquette Démo",...},{"id":3,"name":"T-shirt conteneur",...},{"id":2,"name":"Mug Docker",...},{"id":1,"name":"Sticker Demo",...}]
=== 7. Vérifications finales
local     demo_pgdata
OK : « Casquette Démo » a survécu à la suppression du conteneur
```

Deux choses que je remarque :
- Le `created_at` de la casquette est le même avant et après : c'est bien la même ligne, pas une nouvelle insertion.
- Il y a toujours 4 produits après la recréation, pas 7 : `init.sql` n'a **pas** été rejoué, parce que le volume n'était plus vide. C'est exactement le « piège » du cours, vu en vrai.

## Ce qui m'a posé problème

**L'accent de « Démo » cassé.** Au premier essai, l'API renvoyait `"Casquette D�mo"` et ma vérification finale échouait. Le problème venait de `curl` sous Windows : il reçoit ses arguments dans l'encodage Windows (CP1252) et pas en UTF-8, donc le `é` partait sur un mauvais octet. Solution : envoyer le JSON par un pipe, `printf '...' | curl --data-binary @-`, au lieu de `-d '...'`. J'ai supprimé le volume du premier essai (il gardait la casquette mal encodée, forcément… c'est le principe d'un volume) et relancé.

**Le `pg_isready` trop optimiste.** Au premier démarrage, l'image postgres lance un serveur temporaire pour jouer `init.sql`, puis le redémarre. Ce serveur temporaire écoute seulement sur le socket Unix, donc `pg_isready` sans option peut dire « prêt » trop tôt. Avec `pg_isready -h 127.0.0.1`, on teste en TCP et on attend le vrai serveur.

**Les chemins dans Git Bash.** Git Bash transforme les chemins qui commencent par `/` en chemins Windows, ce qui casse les `-v ...:/docker-entrypoint-initdb.d/...`. D'où `export MSYS_NO_PATHCONV=1` et `pwd -W` en haut du script (sans effet sous Linux).

## Nettoyage

Le script supprime les conteneurs et le réseau à la fin mais garde le volume, pour pouvoir prouver la persistance. Pour tout effacer :

```bash
docker volume rm demo_pgdata
```
