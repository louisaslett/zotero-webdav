FROM httpd:2.4-bookworm

# This file is based on https://github.com/BytemarkHosting/docker-webdav with
# adjustments to use Debian instead of Alpine and to enable LetsEncrypt certbot
# DNS-01 challenge authentication on AWS Route 53.
# Switch to Debian was necessary because recent Alpine versions are broken for
# WebDAV uploads (https://gitlab.alpinelinux.org/alpine/aports/-/issues/13112)

RUN apt-get update \
    && apt-get install -y certbot python3-certbot-dns-route53 cron \
    && rm -rf /var/cache/apt/archives /var/lib/apt/lists

COPY conf/ conf/

RUN \
    # Create empty default DocumentRoot
    mkdir -p "/var/www/html" \
    # Create directories for dav data and lock database
    && mkdir -p "/var/lib/dav/data" \
    && touch "/var/lib/dav/DavLock" \
    && chown -R www-data:www-data "/var/lib/dav" \
    \
    # Enable DAV modules
    && for i in dav dav_fs; do \
        sed -i -e "/^#LoadModule ${i}_module.*/s/^#//" "conf/httpd.conf"; \
    done \
    \
    # Make sure authentication modules are enabled
    && for i in authn_core authn_file authz_core authz_user auth_basic auth_digest; do \
        sed -i -e "/^#LoadModule ${i}_module.*/s/^#//" "conf/httpd.conf"; \
    done \
    \
    # Make sure other modules are enabled
    && for i in alias headers mime setenvif; do \
        sed -i -e "/^#LoadModule ${i}_module.*/s/^#//" "conf/httpd.conf"; \
    done \
    \
    # Run httpd as "www-data"
    && for i in User Group; do \
        sed -i -e "s|^$i .*|$i www-data|" "conf/httpd.conf"; \
    done \
    \
    # Include enabled configs and sites
    && printf '%s\n' "Include conf/conf-enabled/*.conf" \
        >> "conf/httpd.conf" \
    && printf '%s\n' "Include conf/sites-enabled/*.conf" \
        >> "conf/httpd.conf" \
    \
    # Enable dav and default site
    && mkdir -p "conf/conf-enabled" \
    && mkdir -p "conf/sites-enabled" \
    && ln -s ../conf-available/dav.conf "conf/conf-enabled" \
    && ln -s ../sites-available/default.conf "conf/sites-enabled"

COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
EXPOSE 80/tcp 443/tcp
ENTRYPOINT [ "docker-entrypoint.sh" ]
CMD [ "httpd-foreground" ]
