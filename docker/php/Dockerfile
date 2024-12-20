FROM php:8.1-fpm AS rootless

ARG DEBIAN_FRONTEND=noninteractive
ARG NODE_VERSION=20

RUN apt-get update \
    && apt-get install -y \
      ca-certificates \
      curl \
      gnupg \
      lsb-release \
      openssl \
      software-properties-common

RUN curl 'https://deb.nodesource.com/gpgkey/nodesource.gpg.key' | apt-key add - \
    && echo "deb https://deb.nodesource.com/node_${NODE_VERSION}.x $(lsb_release -cs) main" > /etc/apt/sources.list.d/nodesource.list

RUN apt-get update && apt-get install -y \
        libmcrypt-dev \
        libicu-dev \
        libpq-dev \
        libxml2-dev \
        libpng-dev \
        libjpeg-dev \
        libwebp-dev \
        libsqlite3-dev \
        imagemagick \
        libmagickwand-dev \
        libtidy-dev \
        libonig-dev \
        libzip-dev \
        libfreetype6-dev \
        zlib1g-dev \
        git \
        build-essential \
        gettext-base \
        nodejs \
    && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp
RUN docker-php-ext-install -j "$(nproc)" \
        bcmath \
        gd \
        gettext \
        iconv \
        intl \
        mbstring \
        opcache \
        pdo \
        pdo_mysql \
        pdo_pgsql \
        pdo_sqlite \
        sockets \
        tidy \
        zip

RUN pecl install redis && \
    pecl install imagick && \
    pecl install xdebug-3.1.6 && \
    docker-php-ext-enable \
        redis \
        imagick \
        xdebug

RUN version=$(php -r "echo PHP_MAJOR_VERSION.PHP_MINOR_VERSION;") \
    && architecture=$(uname -m) \
    && curl -A "Docker" -o /tmp/blackfire-probe.tar.gz -D - -L -s https://blackfire.io/api/v1/releases/probe/php/linux/$architecture/$version \
    && mkdir -p /tmp/blackfire \
    && tar zxpf /tmp/blackfire-probe.tar.gz -C /tmp/blackfire \
    && mv /tmp/blackfire/blackfire-*.so $(php -r "echo ini_get ('extension_dir');")/blackfire.so \
    && printf "extension=blackfire.so\nblackfire.agent_socket=tcp://blackfire:8307\n" > $PHP_INI_DIR/conf.d/blackfire.ini \
    && rm -rf /tmp/blackfire /tmp/blackfire-probe.tar.gz

RUN mkdir -p /tmp/blackfire \
    && architecture=$(uname -m) \
    && curl -A "Docker" -L https://blackfire.io/api/v1/releases/cli/linux/$architecture | tar zxp -C /tmp/blackfire \
    && mv /tmp/blackfire/blackfire /usr/bin/blackfire \
    && rm -Rf /tmp/blackfire

RUN npm install -g yarn

WORKDIR /var/www/html

COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

COPY docker/php/entrypoint.sh /entrypoint.sh
COPY docker/php/config/ /opt/wallabag/config/

RUN chmod +x /entrypoint.sh && \
    mkdir -p \
        /var/www/html/app/config/ \
        /var/www/html/var/cache \
        /var/www/html/web/assets \
        /var/www/html/data \
        /var/www/html/data/db \
        /var/www/.cache && \
    chown -R www-data:www-data /var/www/html

# Copy application files
COPY --chown=www-data:www-data . /var/www/html/

# Set environment variables
ENV SYMFONY_ENV=prod \
    DATABASE_DRIVER=pdo_sqlite \
    DATABASE_PATH="%kernel.project_dir%/data/db/wallabag.sqlite" \
    DOMAIN_NAME=http://localhost:8000 \
    SECRET=ch4n63m31fy0uc4n \
    PHP_SESSION_HANDLER=redis \
    PHP_SESSION_SAVE_PATH=tcp://redis:6379?database=2 \
    TRUSTED_PROXIES=0.0.0.0/0

# Install dependencies and build assets
USER www-data
RUN composer install --no-dev --no-interaction --optimize-autoloader --prefer-dist && \
    yarn install && \
    yarn build:dev

# Initialize wallabag
RUN mkdir -p data/db && \
    php bin/console doctrine:database:create --if-not-exists && \
    php bin/console wallabag:install --no-interaction

VOLUME ["/var/www/html/data", "/var/www/html/web/assets", "/var/www/.cache"]

USER root
ENTRYPOINT ["/entrypoint.sh"]
CMD ["php-fpm"]

FROM rootless AS default

ARG USER_UID=1000
