#========================================================================
# Copyright Universidade Federal do Espirito Santo (Ufes)
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <https://www.gnu.org/licenses/>.
# 
# This program is released under license GNU GPL v3+ license.
#
#========================================================================

# Build on base image (default: ubuntu:jammy)
# Snap did not build boca-jail properly on ubuntu:bionic 
# error: cannot list snaps: cannot communicate with server: Get http://localhost/v2/snaps: dial unix /run/snapd.socket: connect: no such file or directory
# https://groups.google.com/g/boca-users/c/QrgnJl-KAKw/m/uSuRO64_CQAJ
ARG BASE_IMAGE=ubuntu:jammy

# --- Stage: bocarepo (clones BOCA's repository) ---
# hadolint ignore=DL3006
FROM --platform=${BUILDPLATFORM:-linux/amd64} ${BASE_IMAGE} AS bocarepo
# Git URL of the BOCA repository to clone from (default: cassiopc/boca)
ARG REPOSITORY=https://github.com/KGMats/boca.git
# The branch, tag or SHA to point to in the cloned BOCA repository
ARG REF=master

# hadolint ignore=DL3008,DL3015
RUN apt-get -y update \
    && apt-get -y install git ca-certificates \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /
RUN git clone "${REPOSITORY}"
WORKDIR /boca
RUN git checkout "${REF}"

# --- Stage: base (shared base for web and jail) ---
# The efficient way to publish multi-arch containers from GitHub Actions
# https://actuated.dev/blog/multi-arch-docker-github-actions
# hadolint ignore=DL3006
FROM --platform=${BUILDPLATFORM:-linux/amd64} ${BASE_IMAGE} AS base

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

LABEL authors="Joao Vitor Alves Fazolo, Rodrigo Laiola Guimaraes"

# Environment settings
ENV DEBIAN_FRONTEND=noninteractive \
    DEBCONF_NONINTERACTIVE_SEEN=true \
    # Full list at https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
    TZ=America/Sao_Paulo \
    # Locale and encoding settings
    LANG=en_US.UTF-8 \
    LC_ALL=C.UTF-8 \
    # Apache settings
    APACHE_RUN_USER=www-data \
    APACHE_RUN_GROUP=www-data

# Redundant but to ensure we are not going to break anything
# https://github.com/cassiopc/boca/tree/master/doc
# https://github.com/cassiopc/boca/tree/master/tools
# hadolint ignore=DL3002
USER root

# Install dependencies
# hadolint ignore=DL3005,DL3008,DL3015
RUN apt-get -y update \
    && apt-get -y dist-upgrade \
    && apt-get -y install --no-install-recommends \
        # Miscellaneous
        locales \
        tzdata \
        # Package: boca-common
        # https://github.com/cassiopc/boca/blob/master/debian/control
        # Pre-Depends:
        debconf \
        ca-certificates \
        makepasswd \
        sharutils \
        # Depends:
        libany-uri-escape-perl \
        openssl \
        php-cli \
        php-gd \
        php-pgsql \
        php-xml \
        php-zip \
        postgresql-client \
        wget \
    && rm -rf /var/lib/apt/lists/* \
    && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo $TZ > /etc/timezone \
    && dpkg-reconfigure -f noninteractive tzdata

# Copy BOCA repository from bocarepo stage
# https://github.com/cassiopc/boca/tree/master/doc
# https://github.com/cassiopc/boca/tree/master/tools
COPY --from=bocarepo --chown="$APACHE_RUN_USER:$APACHE_RUN_GROUP" /boca/doc /var/www/boca/doc
COPY --from=bocarepo --chown="$APACHE_RUN_USER:$APACHE_RUN_GROUP" /boca/src /var/www/boca/src
COPY --from=bocarepo --chown="$APACHE_RUN_USER:$APACHE_RUN_GROUP" /boca/tools /var/www/boca/tools

# Copy local config files
COPY --chown="$APACHE_RUN_USER:$APACHE_RUN_GROUP" boca/src/private/conf.php /var/www/boca/src/private/
COPY --chown="$APACHE_RUN_USER:$APACHE_RUN_GROUP" boca/src/private/createdb.php /var/www/boca/src/private/

WORKDIR /var/www/boca
RUN \
    # install-bocacommon
    # https://github.com/cassiopc/boca/blob/master/Makefile
    mkdir -p /usr/sbin /etc/cron.d /var/www/boca/ \
    && install tools/boca-fixssh /usr/sbin/ \
    && install tools/cron-boca-fixssh /etc/cron.d/ \
    && chmod 700 /usr/sbin/boca-fixssh \
    && cp tools/boca.conf /etc/ \
    # boca-common.postinst
    # https://github.com/cassiopc/boca/blob/master/debian/boca-common.postinst
    && install tools/boca-config-dbhost.sh /usr/sbin/boca-config-dbhost \
    && chmod 700 /usr/sbin/boca-config-dbhost \
    && chmod 600 /var/www/boca/src/private/conf.php

# --- Stage: web ---
FROM base AS web

ENV APACHE_PID_FILE=/var/run/apache2/apache2.pid \
    APACHE_DIR=/etc/apache2 \
    APACHE_RUN_DIR=/var/run/apache2 \
    APACHE_LOCK_DIR=/var/lock/apache2 \
    APACHE_LOG_DIR=/var/log/apache2

# Install dependencies
# hadolint ignore=DL3008,DL3015
RUN apt-get -y update \
    && apt-get -y install --no-install-recommends \
        # Package: boca-web
        # https://github.com/cassiopc/boca/blob/master/debian/control
        # Depends:
        apache2 \
        libapache2-mod-php \
        php \
        php-fpm \
        python3-matplotlib \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p $APACHE_LOCK_DIR $APACHE_LOG_DIR $APACHE_RUN_DIR \
    && echo "ServerName localhost" >> $APACHE_DIR/apache2.conf \
    && ln -sf /proc/self/fd/1 $APACHE_LOG_DIR/access.log \
    && ln -sf /proc/self/fd/1 $APACHE_LOG_DIR/error.log \
    && chown -R "$APACHE_RUN_USER:$APACHE_RUN_GROUP" "$APACHE_LOCK_DIR" "$APACHE_LOG_DIR" "$APACHE_RUN_DIR"

WORKDIR /var/www/boca
RUN \
    # install-bocaweb
    # https://github.com/cassiopc/boca/blob/master/Makefile
    mkdir -p $APACHE_DIR/sites-available/ \
    && cp tools/000-boca.conf $APACHE_DIR/sites-available/000-boca.conf \
    && mkdir -p /usr/sbin/ \
    && install tools/dump.sh /usr/sbin/boca-dump \
    && chmod 700 /usr/sbin/boca-dump \
    && chown -R "$APACHE_RUN_USER:$APACHE_RUN_GROUP" /var/www/boca \
    && chmod -R go-rwx /var/www/boca/src/private \
    # boca-web.postinst
    # https://github.com/cassiopc/boca/blob/master/debian/boca-web.postinst
    && a2enmod ssl socache_shmcb proxy_fcgi setenvif \
    && mkdir -p $APACHE_DIR/sites-enabled \
    && cp tools/000-boca.conf $APACHE_DIR/sites-enabled/000-boca.conf \
    && apache2ctl configtest \
    && touch /etc/php/8.1/apache2/conf.d/99-boca-limits.ini \
    && touch /etc/php/8.1/cli/conf.d/99-boca-limits.ini \
    && chown "$APACHE_RUN_USER:$APACHE_RUN_GROUP" /etc/php/8.1/apache2/conf.d/99-boca-limits.ini /etc/php/8.1/cli/conf.d/99-boca-limits.ini

COPY --chmod=755 --chown="$APACHE_RUN_USER:$APACHE_RUN_GROUP" docker/dev/web/init.sh /

USER $APACHE_RUN_USER
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=5 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:80/boca/ || exit 1
EXPOSE 80
ENTRYPOINT ["/init.sh"]

# --- Stage: jail ---
FROM base AS jail

# Install dependencies
# hadolint ignore=DL3008,DL3015
RUN apt-get -y update \
    && apt-get -y install --no-install-recommends \
        # Package: boca-autojudge
        # https://github.com/cassiopc/boca/blob/master/debian/control
        # Depends:
        build-essential \
        debootstrap \
        makepasswd \
        quotatool \
        quota \
        file \
        lsb-release \
        schroot \
        default-jre-headless \
    && rm -rf /var/lib/apt/lists/*

# https://bugs.launchpad.net/ubuntu/+source/ca-certificates-java/+bug/2019908
COPY boca/tools/boca-createjail /var/www/boca/tools/
COPY boca/tools/safeexec.c /var/www/boca/tools/

WORKDIR /var/www/boca
RUN \
    # install-bocaautojudge
    # https://github.com/cassiopc/boca/blob/master/Makefile
    mkdir -p /usr/sbin/ /usr/bin/ /etc/ \
    && gcc tools/safeexec.c -o tools/safeexec \
    && install tools/safeexec /usr/bin/safeexec \
    && install tools/boca-createjail /usr/sbin/boca-createjail \
    && install tools/boca-autojudge.sh /usr/sbin/boca-autojudge \
    && chmod 4555 /usr/bin/safeexec \
    && chmod 700 /usr/sbin/boca-createjail \
    && chmod 700 /usr/sbin/boca-autojudge
    # boca-autojudge.postinst
    # https://github.com/cassiopc/boca/blob/master/debian/boca-autojudge.postinst
    # Done before
    # && chmod 4555 /usr/bin/safeexec \
    # && chmod 700 /usr/sbin/boca-createjail \
    # && chmod 700 /usr/sbin/boca-autojudge

# OpenJDK 17 installation with chroot does not work as expected
# Workaround: Install OpenJDK 17 manually
# https://www.linuxcapable.com/how-to-install-openjdk-17-on-ubuntu-linux/
# hadolint ignore=DL3003
RUN wget --progress=dot:giga https://download.java.net/java/GA/jdk17.0.2/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/openjdk-17.0.2_linux-x64_bin.tar.gz \
    && tar -xvf openjdk-17* \
    && cd jdk-17* \
    && mkdir -p /home/bocajail/usr/bin /home/bocajail/usr/lib \
    && cp -rf bin/* /home/bocajail/usr/bin/ \
    && cp -rf lib/* /home/bocajail/usr/lib/ \
    && cd .. \
    && rm -rf jdk-17* openjdk-17*

COPY --chmod=755 docker/dev/jail/init.sh /

# Add HEALTHCHECK instruction to the container image
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=5 \
    CMD ps ax | grep -v grep | grep php | grep autojudging.php > /dev/null || exit 1

# Use exec format to run program directly as pid 1
# https://www.padok.fr/en/blog/docker-processes-container
ENTRYPOINT ["/init.sh"]
