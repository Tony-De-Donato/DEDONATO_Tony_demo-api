# demo-api

Le fil rouge des quêtes Docker : une mini-API "catalogue" que tu vas
conteneuriser, faire persister, mettre en réseau, orchestrer et sécuriser,
une quête à la fois.

Le métier est volontairement trivial (`Node` + `Express` + `PostgreSQL`,
un catalogue de produits) : toute la difficulté est sur **Docker**, jamais
sur le code applicatif.

## Démarrage avec Compose

Depuis la racine du dépôt, copier la configuration puis modifier le mot de
passe local :

```sh
cp .env.example .env
```

Lancer la stack :

```sh
docker compose up -d --build
```

L'API est disponible sur `http://localhost:${API_PORT:-8080}` et Adminer sur
`http://localhost:${ADMINER_PORT:-8081}`. PostgreSQL n'est pas publié sur
l'hôte. Dans Adminer, utiliser `db` comme serveur et les valeurs de
`POSTGRES_USER`, `POSTGRES_DB` et `DB_PASSWORD` définies dans `.env`.

## Ce que contient le repo

| Fichier | Rôle |
|---|---|
| `api/server.js` | l'API Express (`/`, `/version`, `/health`, `/ready`, `/products`) |
| `api/db.js` | connexion PostgreSQL, entièrement pilotée par des variables d'environnement |
| `api/package.json`, `api/package-lock.json` | dépendances (`express`, `pg`) |
| `db/init.sql` | création de la table `products` + quelques données de démo |
| `compose.yml` | services `api`, `db` et `adminer`, volume et secret |
| `.env.example` | variables de configuration à copier dans `.env` |

## Les routes de l'API

| Méthode | Route | Effet |
|---|---|---|
| `GET` | `/` | infos application + version |
| `GET` | `/version` | numéro de version courant |
| `GET` | `/health` | liveness, ne touche pas la base |
| `GET` | `/ready` | readiness, teste la connexion à la base |
| `GET` | `/products` | liste des produits |
| `POST` | `/products` | crée un produit : `{ "name": "...", "price_cents": 1234 }` |

## Variables et secrets

Les variables sont définies dans `.env`, qui est ignoré par Git. Le mot de
passe est monté dans les conteneurs par le secret Compose `db_password` :
PostgreSQL le lit via `POSTGRES_PASSWORD_FILE` et l'API via
`PGPASSWORD_FILE`. Ne jamais commiter `.env` ni remplacer ce mécanisme par
un mot de passe en clair dans `compose.yml`.

## Vérification

Commande testée :

```sh
docker compose up -d --build
docker compose ps
curl -s localhost:8080/products
curl -s -X POST -H 'content-type: application/json' \
  -d '{"name":"Gourde","price_cents":900}' localhost:8080/products
docker compose down
docker compose up -d
curl -s localhost:8080/products
```

Extrait attendu de `docker compose ps` :

```text
NAME        IMAGE             SERVICE     STATUS
...         ...               db          Up ... (healthy)
...         ...               api         Up ...
```

Le produit `Gourde` reste présent après `down` puis `up` grâce au volume
nommé `pgdata`. `db/init.sql` n'est exécuté qu'au premier démarrage d'un
volume vide.

## Repartir de zéro

Pour supprimer les données PostgreSQL et rejouer `db/init.sql` :

```sh
docker compose down -v
docker compose up -d --build
```

## Prérequis machine (macOS / Linux / Windows)

- **Docker Engine + Compose v2** : le plugin intégré, invoqué en deux mots
  `docker compose` (pas l'ancien binaire autonome `docker-compose` v1).
  `docker compose version` doit répondre `v2.x` ou une version supérieure
  (v3, v4, v5…). Ce qui compte, c'est que ce ne soit pas du v1 legacy.
- macOS / Windows : **Docker Desktop** (ou Colima / Rancher Desktop).
  Sous Windows, backend **WSL 2** : travaille depuis un terminal **WSL**.
- `git`, `curl`. Node est nécessaire **seulement** si tu régénères
  `package-lock.json` (`cd api && npm install`, déjà commité ici).
