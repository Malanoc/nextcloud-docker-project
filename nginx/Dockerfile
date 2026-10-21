# Utilise une image Nginx légère basée sur Alpine Linux
FROM nginx:alpine-slim

# Installe l'utilitaire "usermod" (via le paquet shadow), puis :
# - change l'UID de l'utilisateur nginx en 82 ;
# - ajoute nginx au groupe www-data pour faciliter les accès partagés.
# L'objectif est de rapprocher les permissions de Nginx de celles utilisées
# par l'image PHP-FPM de Nextcloud sur le volume commun afin de garantir la compatibilité des autorisations.
# "--no-cache" évite de conserver le cache des paquets Alpine dans l'image.
RUN apk add --no-cache shadow && \
    usermod -u 82 nginx && \
    usermod -aG www-data nginx

# Remplace la configuration Nginx par celle prévue pour Nextcloud.
# Le fichier nginx.conf doit se trouver dans le contexte de build ./nginx.
COPY nginx.conf /etc/nginx/nginx.conf