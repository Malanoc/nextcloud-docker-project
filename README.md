# nextcloud-docker-project

## Introduction

Ce projet a pour objectif d'héberger une instance de **Nextcloud 35** à l'aide de **Docker Compose**. Nextcloud permet de stocker, synchroniser et partager des fichiers depuis une interface web.

Le déploiement sépare les rôles entre quatre conteneurs : **Nginx** pour le serveur web, **Nextcloud PHP-FPM** pour l'application, **MariaDB** pour la base de données et **Redis** pour la mise en cache. Les paramètres sensibles sont externalisés dans un fichier `.env`.

## Architecture

```text
Navigateur (machine hôte)
          |
          | http://localhost:8080
          v
   Nginx (web : 80)
          |   Réseau frontend
          |   FastCGI vers app:9000
          v
 Nextcloud (app : PHP-FPM)
          |
          |   Réseau backend (interne)
          +---------------------+
          |                     |
          v                     v
 MariaDB (db : 3306)     Redis (redis : 6379)
```

| Service | Image / construction | Rôle |
| --- | --- | --- |
| `web` | Image Nginx construite depuis `./nginx` | Sert les fichiers statiques et transmet les requêtes PHP à Nextcloud. |
| `app` | `nextcloud:35-fpm-alpine` | Exécute Nextcloud via PHP-FPM. |
| `db` | `mariadb:lts` | Stocke les données relationnelles de Nextcloud. |
| `redis` | `redis:alpine` | Fournit un cache accessible avec mot de passe. |

Deux volumes nommés assurent la persistance :

- `nextcloud_data` : fichiers de Nextcloud montés dans `/var/www/html` par `app` et `web` ; il inclut les données applicatives et, selon la configuration de Nextcloud, les fichiers utilisateurs.
- `mariadb_data` : données MariaDB dans `/var/lib/mysql`.

Redis ne possède pas de volume persistant dans cette configuration.

Le conteneur Nginx est personnalisé par un `Dockerfile` basé sur `nginx:alpine-slim`. Il installe `shadow`, configure l'UID de l'utilisateur `nginx` à `82`, l'ajoute au groupe `www-data`, puis copie `nginx.conf`. Cette adaptation vise à faciliter l'accès aux fichiers partagés avec Nextcloud.

## Instructions de lancement

**Prérequis :** Docker Engine et le plugin Docker Compose, ainsi que les quatre fichiers transmis.

**1. Organiser les fichiers.** Le chemin de construction `./nginx` dans `compose.yaml` implique cette structure :

```text
projet-nextcloud/
├── compose.yaml
├── .env.example
├── .env                 # à créer, ne pas partager
└── nginx/
    ├── Dockerfile
    └── nginx.conf
```

**2. Préparer les variables d'environnement.** Depuis la racine du projet :


Créer un fichier .env et copier le contenu de .env.example dedans.

Modifier `.env` et remplacer tous les `CHANGE_ME` par des mots de passe robustes et distincts. Vérifier également les domaines autorisés dans `NEXTCLOUD_TRUSTED_DOMAINS`.


**3. Vérifier et lancer le conteneur.**

```bash
# Vérifier la syntaxe et la résolution de la configuration Compose.
docker compose config --quiet

# Construire et démarrer le conteneur et tous les services.
docker compose up -d --build

# Vérifier l'état du conteneur et les contrôles de santé.
docker compose ps

# Consulter les journaux de tous les services.
docker compose logs -f
```

Ouvrir ensuite **http://localhost:8080** sur la machine qui héberge Docker. Les variables `NEXTCLOUD_ADMIN_USER` et `NEXTCLOUD_ADMIN_PASSWORD` servent à définir le compte administrateur lors de l'initialisation de l'instance.

**Commandes de gestion :**

```bash
# Arrêter les services sans supprimer les données.
docker compose down

# Redémarrer les services.
docker compose up -d

# Afficher les journaux de Nginx uniquement.
docker compose logs -f web
```

**Attention :** `docker compose down -v` supprime aussi les volumes nommés gérés par ce projet et risque donc d'effacer les données Nextcloud et MariaDB.

## Informations sur le réseau, la sécurité, les variables

### Réseaux et ports

| Élément | Configuration | Conséquence |
| --- | --- | --- |
| Réseau `frontend` | `bridge` | Relie uniquement Nginx et Nextcloud. |
| Réseau `backend` | `bridge`, `internal: true` | Relie Nextcloud à MariaDB et Redis, sans exposition directe vers l'extérieur. |
| Nginx | `127.0.0.1:8080:80` | Le port 8080 est publié uniquement sur l'interface loopback de l'hôte. |
| Nextcloud PHP-FPM | `app:9000` | Accessible depuis Nginx sur le réseau frontend ; port non publié sur l'hôte. |
| MariaDB / Redis | Ports internes usuels `3306` / `6379` | Aucun port publié sur l'hôte. |

La connexion **Nginx → Nextcloud** utilise FastCGI (`fastcgi_pass php-handler`, avec `app:9000`). La connexion **Nextcloud → MariaDB/Redis** utilise les noms de services `db` et `redis`, résolus sur le réseau Docker.

### Sécurité

- Les mots de passe sont injectés depuis `.env` plutôt que saisis directement dans `compose.yaml`. Le fichier `.env` doit rester privé et être exclu du dépôt Git (par exemple avec `.gitignore`).
- MariaDB possède un utilisateur applicatif dédié en plus du compte `root`.
- Redis exige un mot de passe grâce à `--requirepass`.
- MariaDB et Redis disposent de contrôles de santé exécutés toutes les 10 secondes (délai maximal de 5 secondes, 10 tentatives). Nextcloud ne démarre qu'une fois ces deux dépendances déclarées saines.
- Tous les services utilisent `restart: unless-stopped`.
- Nginx renvoie une erreur `404` sur plusieurs chemins sensibles (`config`, `data`, fichiers cachés, etc.), et configure les redirections CardDAV/CalDAV.
- La taille maximale du corps d'une requête configurée dans Nginx est de **512 Mo** (`client_max_body_size 512M`). Cela ne garantit pas à lui seul une limite effective de téléversement identique dans PHP/Nextcloud.
- Le réseau interne Docker réduit l'exposition des services, mais ne remplace pas la gestion des correctifs, les sauvegardes ni une configuration sécurisée de Nextcloud.

### Variables d'environnement

| Variable | Utilisation |
| --- | --- |
| `DB_ROOT_PASSWORD` | Mot de passe du compte administrateur MariaDB. |
| `DB_NAME` | Nom de la base Nextcloud. |
| `DB_USER` | Utilisateur MariaDB de Nextcloud. |
| `DB_PASSWORD` | Mot de passe de cet utilisateur. |
| `REDIS_PASSWORD` | Mot de passe Redis, partagé avec Nextcloud. |
| `NEXTCLOUD_ADMIN_USER` | Nom du premier compte administrateur Nextcloud. |
| `NEXTCLOUD_ADMIN_PASSWORD` | Mot de passe de ce compte lors de l'initialisation. |
| `NEXTCLOUD_TRUSTED_DOMAINS` | Hôtes autorisés par Nextcloud, `localhost` dans le fichier exemple. |

La directive MariaDB `--transaction-isolation=READ-COMMITTED` définit le niveau d'isolation des transactions : les lectures ne voient pas les modifications non validées des autres transactions.

## Tests effectués et résultats

| Test | Commande ou méthode | Résultat
| --- | --- | --- |
| Validation Compose | `docker compose config --quiet` | Aucune erreur de syntaxe/configuration. |
| Construction Nginx | `docker compose build web` | Image construite sans erreur. |
| Démarrage des conteneurs | `docker compose up -d` puis `docker compose ps` | Les quatre services sont démarrés. |
| Santé MariaDB et Redis | `docker compose ps` | `db` et `redis` sont marqués `healthy`. |
| Accès Nextcloud | Ouvrir `http://localhost:8080` | Interface de Nextcloud accessible depuis l'hôte. |
| Authentification | Se connecter avec l'administrateur configuré | Connexion réussie après initialisation. |
| Persistance | Créer un fichier, redémarrer les conteneurs, vérifier le fichier | Fichier conservé. |
| Isolation du réseau | Contrôler les ports publiés avec `docker compose ps` | Seul `127.0.0.1:8080` est publié. |

## Problèmes rencontrés

### Erreurs 403 & Problèmes CSS

Pour résoudre définitivement le conflit de permissions (`403 Forbidden`) et garantir le chargement correct des fichiers graphiques (`.css`, `.js`), appliquez les modifications suivantes dans le dossier `./nginx` avant le déploiement.

**Modification du `Dockerfile` (`./nginx/Dockerfile`)**

Il est nécessaire d'harmoniser l'UID de l'utilisateur Nginx avec celui de Nextcloud Alpine (**UID 82**) pour autoriser la lecture du volume partagé.

```dockerfile
FROM nginx:alpine-slim

# Alignement de l'UID sur 82 (www-data de Nextcloud) et ajout au groupe
RUN apk add --no-cache shadow && \
    usermod -u 82 nginx && \
    usermod -aG www-data nginx

COPY nginx.conf /etc/nginx/nginx.conf
```

**Modification de la configuration Nginx (`./nginx/nginx.conf`)**

**Tout en haut du fichier**, déclarez l'utilisateur système :
  ```nginx
  user nginx;
  worker_processes auto;
  ```

**Tout en bas du fichier**, modifiez le bloc `location /` en supprimant le paramètre `$uri/`. Cela évite que Nginx ne s'enferme dans les sous-dossiers physiques de Nextcloud et préserve le chargement du CSS :
  ```nginx
  location / {
      try_files $uri /index.php$request_uri;
  }
  ```


**Après modification :** N'oubliez pas de reconstruire l'image avec la commande `docker compose up --build web` pour appliquer les changements.
