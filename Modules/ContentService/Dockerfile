FROM mlocati/php-extension-installer:2.7@sha256:61ec62196646d4299a959ebdf0f6753f71a7e9db0eebafe1e4f3bf0f323f50be AS php

FROM composer:lts@sha256:dced28fe342974778e0dece19bcfbdc65976b2642ee62f39b87dc064f75561ea AS composer

FROM nginx:1.27.4-alpine@sha256:4ff102c5d78d254a6f0da062b3cf39eaf07f01eec0927fd21e219d0af8bc0591 AS nginx

FROM nextcloud:31.0.1-fpm-alpine@sha256:6d309a25f29a102bbaac7acfe8528744bcec1ec0daa11b41c75d524818693366

# To use a common group for the shared volume with the certificates
RUN addgroup viroo --gid 12345 && \
    addgroup www-data viroo && \
    apk update && \
    apk upgrade && \
    apk --no-cache add \
    envsubst \
    grep \
    imagemagick-dev \
    jq \
    less \
    nginx

# fpm version only
COPY fpm.conf /usr/local/etc/php-fpm.d/www.viroo.conf

# https://www.reddit.com/r/nginx/comments/16ih337/running_nginx_as_nonroot_unprivileged_inside/
RUN touch /var/run/nginx.pid && \
    chown -R www-data:www-data /var/run/nginx.pid && \
    # https://stackoverflow.com/a/64391912
    mkdir -p /var/lib/nginx/tmp /var/log/nginx && \
    chown -R www-data:www-data /var/lib/nginx /var/log/nginx && \
    chmod -R 755 /var/lib/nginx /var/log/nginx

# Allow to use nginx templates
COPY --chown=www-data:www-data nginx.conf /etc/nginx/templates/default.conf.template
COPY --from=nginx --chown=www-data:www-data /docker-entrypoint.d/20-envsubst-on-templates.sh /etc/nginx/templates/20-envsubst-on-templates.sh

RUN mkdir /etc/nginx/conf.d && \
    chown -R www-data:www-data /etc/nginx/conf.d && \
    chmod -R 755 /etc/nginx/conf.d && \
    # Set ownership to nginx folder
    chown -R www-data:www-data /etc/nginx

# Install python/pip
ENV PYTHONUNBUFFERED=1

# hadolint ignore=DL3013
RUN apk add --update --no-cache python3 && ln -sf python3 /usr/bin/python && \
    rm /usr/lib/python3*/EXTERNALLY-MANAGED && \
    python3 -m ensurepip && \
    pip3 install --no-cache-dir --upgrade pip setuptools && \
    # Install yacron
    # hadolint ignore=DL3013
    pip3 install --no-cache-dir yacron

COPY cron /viroo/content_service/cron

# https://github.com/nextcloud/docker/issues/763
RUN touch /usr/local/etc/php/conf.d/redis-session.ini && \
    chown www-data:www-data /usr/local/etc/php/conf.d/redis-session.ini && \
    chmod 600 /usr/local/etc/php/conf.d/redis-session.ini

COPY --from=php /usr/bin/install-php-extensions /usr/bin/

RUN install-php-extensions yaml

COPY --from=composer /usr/bin/composer /usr/bin/composer

RUN mkdir -p /mnt/viroo/shared && \
    chown -R 1000 /mnt/viroo/shared && \
    chmod -R 777 /mnt/viroo/shared

# hadolint ignore=DL3022
COPY --from=shared shared/scripts /viroo/scripts

# Inject entrypoint.sh and std scripts and functions
RUN mv /entrypoint.sh /entrypoint.std.sh
COPY *.sh /

# Fix entrypoint.sh permissions
RUN chown -R www-data:www-data /*.sh && \
    chmod +x /*.sh

#### CUSTOM Layers
# Add custom files
COPY --chown=www-data:www-data customizations /viroo/content_service/customizations

RUN composer install --no-dev --working-dir=/viroo/content_service/customizations/custom_apps/viroo

COPY scripts /viroo/content_service/scripts

# Prepare default folder structure for new users (skeleton)
COPY default_content /viroo/content_service/default_content

RUN chown -R www-data:www-data /viroo/content_service/default_content && \
    # Download custom apps
    VIROO_REQUIRED_APPS=/viroo/content_service/required_apps && \
    mkdir -p "$VIROO_REQUIRED_APPS" && \
    curl -k --retry 5 --retry-delay 5 -sSLo /tmp/groupquota.tar.gz "https://github.com/nextcloud-releases/groupquota/releases/download/v0.2.2/groupquota-v0.2.2.tar.gz" && \
    tar -xzf /tmp/groupquota.tar.gz -C "$VIROO_REQUIRED_APPS" && rm /tmp/groupquota.tar.gz && \
    curl -k --retry 5 --retry-delay 5 -sSLo /tmp/user_oidc.tar.gz "https://github.com/nextcloud-releases/user_oidc/releases/download/v7.0.0/user_oidc-v7.0.0.tar.gz" && \
    tar -xzf /tmp/user_oidc.tar.gz -C "$VIROO_REQUIRED_APPS" && rm /tmp/user_oidc.tar.gz && \
    curl -k --retry 5 --retry-delay 5 -sSLo /tmp/user_saml.tar.gz "https://github.com/nextcloud-releases/user_saml/releases/download/v6.5.0/user_saml-v6.5.0.tar.gz" && \
    tar -xzf /tmp/user_saml.tar.gz -C "$VIROO_REQUIRED_APPS" && rm /tmp/user_saml.tar.gz && \
    curl -k --retry 5 --retry-delay 5 -sSLo /tmp/theming_customcss.tar.gz "https://github.com/nextcloud/theming_customcss/releases/download/v1.18.0/theming_customcss.tar.gz" && \
    tar -xzf /tmp/theming_customcss.tar.gz -C "$VIROO_REQUIRED_APPS" && rm /tmp/theming_customcss.tar.gz && \
    curl -k --retry 5 --retry-delay 5 -sSLo /tmp/welcome.tar.gz "https://github.com/nextcloud-releases/welcome/releases/download/v1.2.1/welcome-v1.2.1.tar.gz" && \
    tar -xzf /tmp/welcome.tar.gz -C "$VIROO_REQUIRED_APPS" && rm /tmp/welcome.tar.gz && \
    chown -R www-data:www-data "$VIROO_REQUIRED_APPS"

HEALTHCHECK CMD ["/bin/sh", "-c", "curl --fail -v http://localhost:8080/${VIROO_NEXTCLOUD_BASEPATH}/status.php | grep '\"installed\":true' | grep '\"maintenance\":false' | grep '\"needsDbUpgrade\":false' || exit 1"]

USER www-data
