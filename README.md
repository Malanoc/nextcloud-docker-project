# Corrections requises sur la branche `main` (Erreurs 403 & Problèmes CSS)

Pour résoudre définitivement le conflit de permissions (`403 Forbidden`) et garantir le chargement correct des fichiers graphiques (`.css`, `.js`), appliquez les modifications suivantes dans le dossier `./nginx` avant le déploiement.

### Modification du `Dockerfile` (`./nginx/Dockerfile`)
Il est nécessaire d'harmoniser l'UID de l'utilisateur Nginx avec celui de Nextcloud Alpine (**UID 82**) pour autoriser la lecture du volume partagé.

```dockerfile
FROM nginx:alpine-slim

# Alignement de l'UID sur 82 (www-data de Nextcloud) et ajout au groupe
RUN apk add --no-cache shadow && \
    usermod -u 82 nginx && \
    usermod -aG www-data nginx

COPY nginx.conf /etc/nginx/nginx.conf
```

### Modification de la configuration Nginx (`./nginx/nginx.conf`)
* **Tout en haut du fichier**, déclarez l'utilisateur système :
  ```nginx
  user nginx;
  worker_processes auto;
  ```

* **Tout en bas du fichier**, modifiez le bloc `location /` en supprimant le paramètre `$uri/`. Cela évite que Nginx ne s'enferme dans les sous-dossiers physiques de Nextcloud et préserve le chargement du CSS :
  ```nginx
  location / {
      try_files $uri /index.php$request_uri;
  }
  ```


**Après modification :** N'oubliez pas de reconstruire l'image avec la commande `docker compose up -d --build web` pour appliquer les changements.
