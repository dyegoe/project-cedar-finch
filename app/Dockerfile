FROM dunglas/frankenphp:1-php8.3-alpine

WORKDIR /app

COPY --chown=www-data:www-data index.php /app/index.php
COPY Caddyfile /etc/frankenphp/Caddyfile

RUN chown -R www-data:www-data /data/caddy /config/caddy

USER www-data

EXPOSE 8080
