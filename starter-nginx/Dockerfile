# the different stages of this Dockerfile are meant to be built into separate images
# https://docs.docker.com/develop/develop-images/multistage-build/#stop-at-a-specific-build-stage
# https://docs.docker.com/compose/compose-file/#target


# https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact
ARG PHP_VERSION=7.4
ARG NGINX_VERSION=1.17
ARG VARNISH_VERSION=6.2


# "php" stage
FROM php:${PHP_VERSION}-fpm-alpine AS api_platform_php

# persistent / runtime deps
RUN apk add --no-cache \
	acl \
	file \
	gettext \
	git \
	nginx \
	;

ARG APCU_VERSION=5.1.17
RUN set -eux; \
	apk add --no-cache --virtual .build-deps \
	$PHPIZE_DEPS \
	icu-dev \
	libzip-dev \
	postgresql-dev \
	zlib-dev \
	; \
	\
	docker-php-ext-configure zip; \
	docker-php-ext-install -j$(nproc) \
	intl \
	pdo_pgsql \
	zip \
	; \
	pecl install \
	apcu-${APCU_VERSION} \
	; \
	pecl clear-cache; \
	docker-php-ext-enable \
	apcu \
	opcache \
	; \
	\
	runDeps="$( \
	scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
	| tr ',' '\n' \
	| sort -u \
	| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --no-cache --virtual .api-phpexts-rundeps $runDeps; \
	\
	apk del .build-deps

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
RUN ln -s $PHP_INI_DIR/php.ini-production $PHP_INI_DIR/php.ini
COPY docker/api-platform.prod.ini $PHP_INI_DIR/conf.d/api-platform.ini
COPY docker/default.conf /etc/nginx/http.d/default.conf

# Workaround to allow using PHPUnit 8 with Symfony 4.3
ENV SYMFONY_PHPUNIT_VERSION=8.3

# https://getcomposer.org/doc/03-cli.md#composer-allow-superuser
ENV COMPOSER_ALLOW_SUPERUSER=1
# install Symfony Flex globally to speed up download of Composer packages (parallelized prefetching)
RUN set -eux; \
	composer global require "symfony/flex" --prefer-dist --no-progress --classmap-authoritative; \
	composer clear-cache
ENV PATH="${PATH}:/root/.composer/vendor/bin"

WORKDIR /srv/api

# build for production
ARG APP_ENV=prod

# prevent the reinstallation of vendors at every changes in the source code
COPY composer.json composer.lock symfony.lock ./
RUN set -eux; \
	composer install --prefer-dist --no-dev --no-scripts --no-progress; \
	composer clear-cache

# do not use .env files in production
COPY .env ./
RUN composer dump-env prod; \
	rm .env

# copy only specifically what we need
COPY bin bin/
COPY config config/
COPY public public/
COPY migrations migrations
COPY templates templates
COPY src src/

RUN set -eux; \
	mkdir -p var/cache var/log; \
	composer dump-autoload --classmap-authoritative --no-dev; \
	composer run-script --no-dev post-install-cmd; \
	chmod +x bin/console; sync
VOLUME /srv/api/var

CMD ["php-fpm"]