# syntax=docker/dockerfile:1.7
FROM php:8.3-fpm-alpine3.23 AS base
RUN apk upgrade --no-cache
RUN apk add --no-cache icu-libs libcurl ca-certificates \
 && apk add --no-cache --virtual .build-deps $PHPIZE_DEPS icu-dev curl-dev \
 && docker-php-ext-install -j"$(getconf _NPROCESSORS_ONLN)" intl curl opcache \
 && apk del .build-deps

FROM base AS build
RUN apk upgrade --no-cache
RUN apk add --no-cache git unzip
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer
WORKDIR /app
ARG STREAK_REF=125aaaf200620410446ebfa51897fca3607e04fd
RUN git init . && git remote add origin https://github.com/DenverCoder1/github-readme-streak-stats.git \
 && git fetch --depth=1 origin "$STREAK_REF" && git checkout FETCH_HEAD
RUN composer install --no-dev --prefer-dist --no-interaction --optimize-autoloader \
 && composer check-platform-reqs --no-dev

FROM base AS runtime
RUN apk upgrade --no-cache
# GNU tar is only needed during extension/source builds.
RUN apk del --no-network tar
WORKDIR /app/src
COPY --from=build /app/src /app/src
COPY --from=build /app/vendor /app/vendor
RUN mkdir -p /app/cache && chown 82:82 /app/cache
COPY pool.conf /usr/local/etc/php-fpm.d/zz-app.conf
USER 82:82
EXPOSE 9000
CMD ["php-fpm", "-F"]
