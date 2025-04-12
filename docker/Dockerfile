FROM httpd:2.4-alpine

RUN apk add --no-cache certbot certbot-dns-route53

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
    done
