FROM composer/composer:2-bin AS composer

FROM php:7.4-apache AS base

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libicu-dev \
    libxml2-dev \
    libzip-dev \
    libonig-dev \
    && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
    gd \
    intl \
    mbstring \
    pdo_mysql \
    zip \
    opcache \
    && docker-php-ext-enable opcache

# Configure Apache
RUN a2enmod rewrite headers

# Set working directory
WORKDIR /app

# Copy composer
COPY --from=composer /composer /usr/bin/composer

#
# Development
#

FROM base AS dev

ARG USER_ID=1000
ARG GROUP_ID=1000

# Match host user UID/GID for correct file ownership on mounted volumes
RUN usermod -u ${USER_ID} www-data && groupmod -g ${GROUP_ID} www-data

# Suppress git "dubious ownership" warning for mounted /app
RUN git config --system --add safe.directory /app

# Set environment variables
ENV APACHE_DOCUMENT_ROOT=/app/www

# Configure Apache document root
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
    && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf

# Apache needs root to bind port 80; it drops to www-data for worker processes

#
# Production
#

FROM base AS prod-builder

COPY . /app

RUN --mount=type=cache,target=/tmp/composer-cache \
    composer install --no-dev --no-progress --no-interaction --optimize-autoloader --classmap-authoritative \
    && php ./init --env=Production --overwrite=All \
    && chmod +x yii \
    && mkdir -p runtime www/assets \
    && chown -R www-data:www-data /app

FROM base AS prod

ENV APACHE_DOCUMENT_ROOT=/app/www

RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
    && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf

COPY --from=prod-builder --chown=www-data:www-data /app /app

# Apache needs root to bind port 80; it drops to www-data for worker processes
