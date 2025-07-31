# Multi-stage build para optimizar el tamaño final
FROM php:8.2-apache as base

# Variables de entorno
ENV COMPOSER_ALLOW_SUPERUSER=1
ENV COMPOSER_HOME="/tmp"
ENV DEBIAN_FRONTEND=noninteractive
ENV NODE_VERSION=18

# Actualizar sistema e instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    # Herramientas básicas
    git \
    curl \
    wget \
    nano \
    vim \
    unzip \
    supervisor \
    cron \
    # Dependencias para extensiones PHP
    libzip-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libonig-dev \
    libxml2-dev \
    libicu-dev \
    libmagickwand-dev \
    libmemcached-dev \
    libpq-dev \
    libc-client-dev \
    libkrb5-dev \
    libldap2-dev \
    libtidy-dev \
    libgmp-dev \
    libbz2-dev \
    libssl-dev \
    libevent-dev \
    librabbitmq-dev \
    libsqlite3-dev \
    # Herramientas para imágenes y PDFs
    imagemagick \
    ghostscript \
    # Limpiar cache de apt
    && rm -rf /var/lib/apt/lists/*

# Instalar Node.js y npm para build tools
RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - \
    && apt-get install -y nodejs

# Configurar extensiones PHP complejas
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-configure imap --with-kerberos --with-imap-ssl \
    && docker-php-ext-configure ldap --with-libdir=lib/x86_64-linux-gnu/

# Instalar extensiones PHP principales
RUN docker-php-ext-install -j$(nproc) \
    # Base de datos
    pdo_mysql \
    mysqli \
    pdo_pgsql \
    pgsql \
    pdo_sqlite \
    # Gráficos e imágenes
    gd \
    exif \
    # Archivos y compresión
    zip \
    bz2 \
    # Internacionalización
    intl \
    mbstring \
    gettext \
    # Performance
    opcache \
    # Matemáticas
    bcmath \
    gmp \
    # Servicios web
    soap \
    # XML/HTML
    xml \
    tidy \
    # Email
    imap \
    # LDAP
    ldap \
    # Networking
    sockets \
    ftp \
    # System V IPC
    sysvmsg \
    sysvsem \
    sysvshm \
    # Process control
    pcntl \
    posix \
    # Calendario
    calendar

# Instalar extensiones PECL
RUN pecl install \
    # Cache
    redis \
    apcu \
    memcached \
    # Imágenes
    imagick \
    # Desarrollo (opcional - comentar en producción)
    xdebug \
    && docker-php-ext-enable \
    redis \
    apcu \
    memcached \
    imagick \
    xdebug

# Instalar Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Instalar herramientas de desarrollo PHP globales
RUN composer global require \
    phpunit/phpunit \
    friendsofphp/php-cs-fixer \
    phpstan/phpstan \
    && ln -s /tmp/vendor/bin/* /usr/local/bin/

# Habilitar módulos de Apache
RUN a2enmod rewrite \
    && a2enmod ssl \
    && a2enmod headers \
    && a2enmod expires \
    && a2enmod deflate

# Configurar supervisor
RUN mkdir -p /var/log/supervisor

# Crear directorios necesarios
RUN mkdir -p /var/log/php \
    && mkdir -p /var/www/html/storage/logs \
    && mkdir -p /var/www/html/bootstrap/cache

# Configurar permisos y usuario de seguridad
RUN groupadd -r phpuser && useradd -r -g phpuser phpuser \
    && chown -R www-data:www-data /var/www \
    && chown -R www-data:www-data /var/log/php \
    && chmod -R 755 /var/www \
    && chmod -R 755 /var/log/php

# Configurar cron para tareas programadas
RUN touch /var/log/cron.log \
    && chmod 644 /var/log/cron.log

# Establecer directorio de trabajo
WORKDIR /var/www/html

# Crear archivo de configuración de supervisor
RUN echo '[supervisord]' > /etc/supervisor/conf.d/supervisord.conf \
    && echo 'nodaemon=true' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'user=root' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo '' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo '[program:apache2]' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'command=/usr/sbin/apache2ctl -D FOREGROUND' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'autostart=true' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'autorestart=true' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'stderr_logfile=/var/log/apache2/error.log' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'stdout_logfile=/var/log/apache2/access.log' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo '' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo '[program:cron]' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'command=/usr/sbin/cron -f' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'autostart=true' >> /etc/supervisor/conf.d/supervisord.conf \
    && echo 'autorestart=true' >> /etc/supervisor/conf.d/supervisord.conf

# Copiar configuraciones personalizadas (si existen)
# COPY ./config/apache.conf /etc/apache2/sites-available/000-default.conf
# COPY ./config/php.ini /usr/local/etc/php/conf.d/custom.ini

# Limpiar archivos temporales y cache
RUN apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && rm -rf /var/tmp/*

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# Exponer puertos
EXPOSE 80 443

# Comando por defecto usando supervisor
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]

# Información de la imagen
LABEL maintainer="Tu Nombre <tu@email.com>"
LABEL description="PHP 8.2 con Apache - Entorno completo para aplicaciones modernas"
LABEL version="1.0"