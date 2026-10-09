# Quête 8 – Analyse de vulnérabilité avec Trivy

Fil rouge, étape 7 : scanner `demo-api`, corriger, et mettre une gate Trivy dans la CI.

- Branche : [quete-8-trivy](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/tree/quete-8-trivy) du repo `BARRET_hugo_demo-api` (à partir de cette quête, une branche par quête)
- Scans : [scan-avant.txt](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/blob/quete-8-trivy/scans/scan-avant.txt) et [scan-apres.txt](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/blob/quete-8-trivy/scans/scan-apres.txt)
- CI : [.github/workflows/ci.yml](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/blob/quete-8-trivy/.github/workflows/ci.yml)

## Journal de bord

**1. Scan initial**

Avec l'image `aquasec/trivy` (version 0.75.0), sans rien installer, depuis Git Bash :

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache \
  aquasec/trivy:latest image --severity HIGH,CRITICAL demo-api:multi > scans/scan-avant.txt
```

Le volume `trivy-cache` évite de retélécharger la base de CVE à chaque scan.

```
demo-api:multi (alpine 3.20.3)
Total: 21 (HIGH: 19, CRITICAL: 2)

Node.js (node-pkg)
Total: 33 (HIGH: 30, CRITICAL: 3)
```

**54 failles HIGH/CRITICAL.** En regardant d'où elles venaient :
- **OS (Alpine 3.20.3)** : openssl (`libcrypto3` / `libssl3`, en CRITICAL), `musl`, `zlib`. Ma base `node:22.11-alpine` date de fin 2024, elle a pris presque deux ans de CVE.
- **Node**, et c'est là que j'ai été surpris : la plupart ne sont **pas** dans mon API mais dans **npm lui-même**, livré avec l'image node (`/usr/local/lib/node_modules/npm` : `tar`, `glob`, `cross-spawn`, `sigstore`…). Plus un peu de corepack/yarn.
- Et quelques-unes dans mes vraies dépendances (`/app/node_modules`) : `proxy-addr` (CRITICAL), `body-parser`, `send`.

**2. Corrections**

- **Base** : `node:22.11-alpine` → `node:22.23.3-alpine3.24` (le Node 22 le plus récent). J'ai mis le tag dans un `ARG NODE_IMAGE` en haut du Dockerfile, utilisé par les deux `FROM`, pour n'avoir qu'une ligne à changer.
- **Dépendances** : `npm audit fix` dans `api/` → `package-lock.json` mis à jour, `npm audit` : `found 0 vulnerabilities`.
- **npm / corepack / yarn retirés de l'image finale** : l'API n'en a pas besoin pour tourner, `npm ci` est fait dans l'étape `deps`. C'est vraiment là que le multi-étapes de la quête 4 sert :

```dockerfile
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/corepack \
      /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack \
      /opt/yarn-* /usr/local/bin/yarn /usr/local/bin/yarnpkg
```

J'ai aussi passé le `Dockerfile` principal (celui de Compose) sur la nouvelle base.

**3. Re-scan**

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache \
  aquasec/trivy:latest image --severity HIGH,CRITICAL --ignore-unfixed demo-api:multi > scans/scan-apres.txt
```

**0 partout**, sur les 84 cibles scannées (l'OS en alpine 3.24.2 et chaque paquet de `/app/node_modules`). Et même **sans** `--ignore-unfixed`, il n'y a plus rien. Plus de ligne `usr/local/lib/node_modules/npm` dans le rapport : npm n'est plus dans l'image. Pas de `.trivyignore` nécessaire.

L'image tourne toujours (`/health` → `{"status":"UP"}`) et `which npm` répond `npm absent`.

Par contre l'image est passée de 228 à 244 MB : la nouvelle base est un peu plus grosse, et surtout le `rm` est dans sa propre couche, donc il **masque** les fichiers sans les retirer des couches de la base (le « whiteout » du cours de la quête 4). Ici le but était la sécurité, pas la taille.

**4. La CI**

`.github/workflows/ci.yml` : à chaque push (branches et tags `v*`), build de `api/Dockerfile.multi` puis `aquasecurity/trivy-action` avec `severity: CRITICAL,HIGH`, `ignore-unfixed: true`, `exit-code: "1"`.

Premier run : **rouge en 6 secondes**, avant même le build : `Unable to resolve action aquasecurity/trivy-action@0.28.0, unable to find version 0.28.0`. La version du cours n'existe plus : les tags sont maintenant préfixés par `v` (`v0.28.0` … `v0.36.0`). J'ai pris la dernière, `v0.36.0`, et je l'ai épinglée par **SHA de commit** plutôt que par tag : un tag peut être déplacé vers un autre code, un SHA non. Pour une action de sécurité, ça me paraît le minimum.

**5. Démonstration de la gate**

| Tag | Base | CI |
|---|---|---|
| `v1.0.0` | `node:22.23.3-alpine3.24` | ✅ [Success](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/actions/runs/37947556759) |
| `v1.0.1` | `node:22.11-alpine` (régression volontaire) | ❌ [Failure](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/actions/runs/37947585184) |
| `v1.0.2` | retour à `node:22.23.3-alpine3.24` | ✅ [Success](https://github.com/ynov-x-anthony/BARRET_hugo_demo-api/actions/runs/37947933366) |

Sur le run rouge, j'ai vérifié que c'est bien la gate qui bloque et pas autre chose : `Build (local, pour scanner)` → success, `Scan Trivy` → **failure**.

Intéressant : même en régression, npm reste retiré de l'image. J'ai reproduit la régression en local (`--build-arg NODE_IMAGE=node:22.11-alpine`) pour voir ce qui bloque : `alpine 3.20.3 — Total: 21 (HIGH: 19, CRITICAL: 2)` et **rien** côté Node. La CI échoue donc uniquement sur l'OS de la vieille base (openssl, musl, zlib). Ça montre que les corrections se complètent : le retrait de npm règle la partie Node, mais seule la mise à jour de la base règle l'OS.
