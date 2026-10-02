FROM nginx:alpine-slim

RUN apk add --no-cache shadow && \
    usermod -u 82 nginx && \
    usermod -aG www-data nginx

COPY nginx.conf /etc/nginx/nginx.conf

#test