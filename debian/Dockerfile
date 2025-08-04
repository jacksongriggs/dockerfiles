ARG PHP=8.3

# Modified prepare-app stage to build from fork instead of downloading release
FROM node:20-bookworm AS prepare-app

WORKDIR /build

# Install PHP and required extensions for composer
RUN apt-get update && apt-get install -y \
    php-cli \
    php-bcmath \
    php-gd \
    php-mbstring \
    php-xml \
    php-curl \
    php-zip \
    git \
    unzip

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Clone your forked repository
RUN git clone --depth 1 -b v5-stable https://github.com/jacksongriggs/invoiceninja-backend.git /var/www/html

WORKDIR /var/www/html

# Install composer dependencies
RUN composer install --no-dev --optimize-autoloader

# Install MCP SDK separately
RUN composer require logiscape/mcp-sdk-php --no-scripts --optimize-autoloader

# Download UI release from your fork (built by GitHub Actions)
RUN curl -L -o ui.zip https://github.com/jacksongriggs/ui/releases/latest/download/invoiceninja-react.zip && \
    unzip -q ui.zip && \
    cp -r dist/* public/ && \
    mv public/index.html resources/views/react/index.blade.php && \
    rm -rf ui.zip dist/

# Build main app assets
RUN npm ci && npm run production

# Clean up
RUN rm -rf node_modules ui .git

# Create necessary links and prepare public directory
RUN ln -s /var/www/html/resources/views/react/index.blade.php /var/www/html/public/index.html && \
    php artisan storage:link && \
    # Workaround for application updates (same as original)
    mv /var/www/html/public /tmp/public

# ==================
# EVERYTHING BELOW THIS LINE IS IDENTICAL TO THE ORIGINAL DOCKERFILE
# ==================
FROM php:${PHP}-fpm

# PHP modules
ARG php_require="bcmath gd mbstring pdo_mysql zip"
ARG php_suggest="exif imagick intl pcntl saxon soap"
ARG php_extra="opcache"

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    mariadb-client \
    gpg \
    supervisor \
    # Unicode support for PDF
    fonts-noto-cjk-extra \
    fonts-wqy-microhei \
    fonts-wqy-zenhei \
    xfonts-wqy \
    # Install google-chrome-stable(amd64)/chromium(arm64)
    && if [ "$(dpkg --print-architecture)" = "amd64" ]; then \
    mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://dl.google.com/linux/linux_signing_key.pub | \
    gpg --dearmor -o /etc/apt/keyrings/google.gpg \
    && echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/google.gpg] https://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends google-chrome-stable; \
    elif [ "$(dpkg --print-architecture)" = "arm64" ]; then \
    apt-get install -y --no-install-recommends \
    chromium; \        
    fi \
    # Create config directory for chromium/google-chrome-stable
    && mkdir /var/www/.config \
    && chown www-data:www-data /var/www/.config \
    # Cleanup
    && apt-get purge -y gpg \
    && apt-get autoremove -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
COPY --from=ghcr.io/mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/

RUN install-php-extensions \
    ${php_require} \
    ${php_suggest} \
    ${php_extra}

# Configure PHP
RUN ln -s "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

COPY php/php.ini /usr/local/etc/php/conf.d/invoiceninja.ini

COPY php/php-fpm.conf /usr/local/etc/php-fpm.d/invoiceninja.conf

# Setup supervisor
COPY supervisor/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Setup InvoiceNinja
COPY --from=prepare-app /var/www/html /var/www/html
COPY --from=prepare-app /tmp/public /tmp/public

# Fix ownership for www-data
RUN chown -R www-data:www-data /var/www/html && \
    chmod -R 755 /var/www/html/bootstrap/cache && \
    chmod -R 755 /var/www/html/storage

# Add initialization script
COPY --chmod=0755 scripts/init.sh /usr/local/bin/init.sh

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD pgrep -f "php-fpm: master process"

ENTRYPOINT ["/usr/local/bin/init.sh"]
CMD ["supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]