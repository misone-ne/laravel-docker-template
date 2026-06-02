FROM php:8.3-fpm

ARG UID=1000
ARG GID=1000

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    zip \
    default-mysql-client \
    libzip-dev \
    libicu-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install \
    pdo_mysql \
    zip \
    intl \
    bcmath \
    gd \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

RUN groupmod -g ${GID} www-data \
    && usermod -u ${UID} -g ${GID} www-data

WORKDIR /var/www