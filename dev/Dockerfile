FROM php:7.0-fpm
MAINTAINER "Rus0 <andonid88@gmail.com>"

ENV PHP_EXTRA_CONFIGURE_ARGS="--enable-fpm --with-fpm-user=magento2 --with-fpm-group=magento2"
ENV NODE_VERSION="6.x"

RUN apt-get update && apt-get install -y \
    apt-utils \
    sudo \
    wget \
    unzip \
    cron \
    curl \
    libmcrypt-dev \
    libicu-dev \
    libxml2-dev libxslt1-dev \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    git \
    vim \
    nano \
    openssh-server \
    supervisor \
    mysql-client \
    ocaml \
    expect

RUN curl -L https://github.com/bcpierce00/unison/archive/2.48.4.tar.gz | tar zxv -C /tmp && \
             cd /tmp/unison-2.48.4 && \
             sed -i -e 's/GLIBC_SUPPORT_INOTIFY 0/GLIBC_SUPPORT_INOTIFY 1/' src/fsmonitor/linux/inotify_stubs.c && \
             make && \
             cp src/unison src/unison-fsmonitor /usr/local/bin && \
             cd /root && rm -rf /tmp/unison-2.48.4 

RUN docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-configure hash --with-mhash \
    && docker-php-ext-install -j$(nproc) mcrypt intl xsl gd zip pdo_mysql pcntl opcache soap bcmath json iconv 

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer 

RUN mkdir /var/run/sshd \
    && echo "StrictHostKeyChecking no" >> /etc/ssh/ssh_config

RUN apt-get update \
    && apt-get install -y apache2

RUN a2enmod rewrite \
    && a2enmod proxy \
    && a2enmod proxy_fcgi 

RUN rm -f /etc/apache2/sites-enabled/000-default.conf 

RUN useradd -m -d /home/magento2 -s /bin/bash magento2 && adduser magento2 sudo \
    && touch /etc/sudoers.d/privacy \
    && echo "Defaults        lecture = never" >> /etc/sudoers.d/privacy \
    && mkdir /home/magento2/magento2 && mkdir /home/magento2/state 

RUN curl -sS https://accounts.magento.cloud/cli/installer -o /home/magento2/installer \
    && rm -r /usr/local/etc/php-fpm.d/* \
    && sed -i 's/www-data/magento2/g' /etc/apache2/envvars 

RUN pecl install xdebug-stable \
    && echo "zend_extension=$(find /usr/local/lib/php/extensions/ -name xdebug.so)" > /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.remote_enable=on" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.remote_autostart=off" >> /usr/local/etc/php/conf.d/xdebug.ini

# PHP config
ADD conf/php.ini /usr/local/etc/php

# SSH config
COPY conf/sshd_config /etc/ssh/sshd_config
RUN chown magento2:magento2 /etc/ssh/ssh_config

# supervisord config
ADD conf/supervisord.conf /etc/supervisord.conf

# php-fpm config
ADD conf/php-fpm-magento2.conf /usr/local/etc/php-fpm.d/php-fpm-magento2.conf

# apache config
ADD conf/apache-default.conf /etc/apache2/sites-enabled/apache-default.conf

# unison script
ADD conf/.unison/magento2.prf /home/magento2/.unison/magento2.prf

ADD conf/unison.sh /usr/local/bin/unison.sh
ADD conf/entrypoint.sh /usr/local/bin/entrypoint.sh
ADD conf/check-unison.sh /usr/local/bin/check-unison.sh
RUN chmod +x /usr/local/bin/unison.sh && chmod +x /usr/local/bin/entrypoint.sh \
    && chmod +x /usr/local/bin/check-unison.sh

ENV USE_SHARED_WEBROOT 1
ENV SHARED_CODE_PATH /var/www/html
ENV WEBROOT_PATH /var/www/html
ENV MAGENTO_ENABLE_SYNC_MARKER 0

## Node
RUN apt-get update && apt-get install -my wget gnupg
WORKDIR /tmp/grunt
RUN curl -sL https://deb.nodesource.com/setup_${NODE_VERSION} | bash - \
&& apt-get update \
    && apt-get install -y build-essential nodejs \
    && npm install -g express && npm install -g grunt-cli

RUN chown -R magento2:magento2 /home/magento2 && \
    chown -R magento2:magento2 /var/www/html

# Delete user password to connect with ssh with empty password
RUN passwd magento2 -d

EXPOSE 80 22 5000 44100
WORKDIR /var/www/html

USER root

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
