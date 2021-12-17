# the different stages of this Dockerfile are meant to be built into separate images
# https://docs.docker.com/develop/develop-images/multistage-build/#stop-at-a-specific-build-stage
# https://docs.docker.com/compose/compose-file/#target


# https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact
ARG PHP_VERSION=7.4


# "php" stage
FROM php:${PHP_VERSION}-apache

RUN apt-get -y update
RUN apt-get -y install git libzip-dev postgresql-client zlib1g unzip libmcrypt-dev libpq-dev
RUN docker-php-ext-configure pgsql -with-pgsql=/usr/local/pgsql \
    && docker-php-ext-install pgsql pdo_pgsql

# "php" enable mode rewrite
RUN a2enmod rewrite

# "php" copy site configuration
COPY docker/default.conf /etc/apache2/sites-available/000-default.conf

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Workaround to allow using PHPUnit 8 with Symfony 4.3
ENV SYMFONY_PHPUNIT_VERSION=8.3

# https://getcomposer.org/doc/03-cli.md#composer-allow-superuser
ENV COMPOSER_ALLOW_SUPERUSER=1
# install Symfony Flex globally to speed up download of Composer packages (parallelized prefetching)
RUN set -eux; \
	composer global require "symfony/flex" --prefer-dist --no-progress --classmap-authoritative; \
	composer clear-cache
ENV PATH="${PATH}:/root/.composer/vendor/bin"

WORKDIR /var/www/html/

# build for production
# ARG APP_ENV=prod

# prevent the reinstallation of vendors at every changes in the source code
COPY composer.json composer.lock symfony.lock ./
RUN set -eux; \
	composer install --prefer-dist --no-scripts --no-progress; \
	composer clear-cache

# do not use .env files in production
COPY .env ./

# copy only specifically what we need
COPY bin bin/
COPY config config/
COPY public public/
COPY migrations migrations
COPY templates templates/
COPY src src/

EXPOSE 80