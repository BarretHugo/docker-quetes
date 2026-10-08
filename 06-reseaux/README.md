# Quête 6 – Les réseaux

Fil rouge, étape 3 : la base n'est plus joignable que par l'API.

- Script : [reseaux_hugo_barret.sh](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/blob/main/reseaux_hugo_barret.sh) dans le repo `BARRET_hugo_demo-api`

## Journal de bord

**1. L'idée**

Deux réseaux perso au lieu du bridge par défaut :

| Conteneur | demo_front | demo_back | Port publié |
|---|---|---|---|
| demo-api | ✅ | ✅ | 8080 |
| demo-db | ❌ | ✅ | aucun |

L'API fait le pont. La base n'a ni port publié ni pied sur `demo_front`, donc rien d'autre que l'API ne peut la joindre.

Je lance l'API sur `demo_front` puis je l'ajoute à `demo_back` avec `docker network connect`. Les versions récentes de Docker acceptent plusieurs `--network` dans le `docker run`, mais `network connect` marche partout et c'est ce qu'on a vu en cours.

**2. Les preuves (sortie du script)**

L'API voit la base par son nom, grâce au DNS interne de Docker sur les réseaux perso :

```
=== 5. demo-api résout demo-db par son nom
172.21.0.2        demo-db  demo-db
```

Un conteneur jetable branché seulement sur `demo_front` ne la voit pas :

```
--- par son nom :
nc: bad address 'demo-db'
--- par son IP (172.21.0.2) :
nc: 172.21.0.2 (172.21.0.2:5432): Operation timed out
```

J'ai ajouté le test par IP moi-même : `bad address` montre seulement que le nom n'est pas résolu. Avec l'IP, on voit que même en connaissant l'adresse, on ne passe pas. Ce n'est pas un pare-feu, c'est juste qu'il n'y a aucun réseau en commun.

Les IP, une par réseau :

```
demo-db :
  demo_back : 172.21.0.2

demo-api :
  demo_back : 172.21.0.3
  demo_front : 172.20.0.2
```

On voit bien que l'API a deux adresses (une par réseau) et la base une seule. Les deux réseaux ont chacun leur sous-réseau (`172.20.x` et `172.21.x`).

Et l'API répond toujours :

```
[{"id":3,"name":"T-shirt conteneur",...},{"id":2,"name":"Mug Docker",...},{"id":1,"name":"Sticker Demo",...}]
```

Ce sont les 3 produits de `init.sql` : je n'ai pas réutilisé le volume de la quête 5, la base repart de zéro.

## Remarques

- `docker ps` montre `5432/tcp` pour demo-db sans `0.0.0.0:...->` : le port existe dans le conteneur mais n'est pas publié sur ma machine.
- Mon premier essai utilisait `alpine` sans tag et a téléchargé `alpine:latest`. Pas très cohérent avec la quête sécurité, j'ai épinglé `alpine:3.20`.
- Le script fait le nettoyage dans un `trap ... EXIT`, comme ça les conteneurs et réseaux sont supprimés même si une étape plante au milieu.
