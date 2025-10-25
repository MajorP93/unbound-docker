FROM alpine:3.22 AS openssl
LABEL maintainer="MajorP93"

ENV VERSION_OPENSSL=openssl-3.5.4 \
    SHA256_OPENSSL=967311f84955316969bdb1d8d4b983718ef42338639c621ec4c34fddef355e99 \
    SOURCE_OPENSSL=https://www.openssl.org/source/ \
    # OpenSSL OMC
    OPGP_OPENSSL_1=EFC0A467D613CB83C7ED6D30D894E2CE8B3D79F5 \
    # Richard Levitte
    OPGP_OPENSSL_2=7953AC1FBC3DC8B3B292393ED5E9E43F7DF9EE8C \
    # Matt Caswell
    OPGP_OPENSSL_3=8657ABB260F056B1E5190839D9C4D26D0E604491 \
    # Paul Dale
    OPGP_OPENSSL_4=B7C1C14360F353A36862E4D5231C84CDDCC69C45 \
    # Tomas Mraz
    OPGP_OPENSSL_5=A21FAB74B0088AA361152586B8EF1A6BA9DA2D5C \
    # Tim Hudson
    OPGP_OPENSSL_6=C1F33DD8CE1D4CC613AF14DA9195C48241FBF7DD \
    # Kurt Roeckx
    OPGP_OPENSSL_7=E5E52560DD91C556DDBDA5D02064C53641C25E5D \
    # OpenSSL
    OPGP_OPENSSL_8=BA5473A2B0587B07FB27CF2D216094DFD0CB81EF

WORKDIR /tmp/src

RUN set -eux && \
    apk add --no-cache --virtual .build-deps \
      build-base \
      ca-certificates \
      curl \
      gnupg \
      perl \
      pkgconf \
      linux-headers \
      libidn2-dev && \
    curl -L "${SOURCE_OPENSSL}${VERSION_OPENSSL}.tar.gz" -o openssl.tar.gz && \
    echo "${SHA256_OPENSSL}  openssl.tar.gz" | sha256sum -c - && \
    curl -L "${SOURCE_OPENSSL}${VERSION_OPENSSL}.tar.gz.asc" -o openssl.tar.gz.asc && \
    export GNUPGHOME="$(mktemp -d)" && \
    gpg --no-tty --keyserver hkps://keyserver.ubuntu.com --recv-keys \
      "$OPGP_OPENSSL_1" "$OPGP_OPENSSL_2" "$OPGP_OPENSSL_3" "$OPGP_OPENSSL_4" \
      "$OPGP_OPENSSL_5" "$OPGP_OPENSSL_6" "$OPGP_OPENSSL_7" "$OPGP_OPENSSL_8" && \
    gpg --batch --verify openssl.tar.gz.asc openssl.tar.gz && \
    tar xzf openssl.tar.gz && \
    cd "${VERSION_OPENSSL}" && \
    ./config \
      --prefix=/opt/openssl \
      --openssldir=/opt/openssl \
      no-weak-ssl-ciphers \
      no-ssl3 \
      no-shared \
      enable-ec_nistp_64_gcc_128 \
      -DOPENSSL_NO_HEARTBEATS \
      -fstack-protector-strong && \
    make depend && \
    make -j"$(getconf _NPROCESSORS_ONLN 2>/dev/null || echo 1)" && \
    make install_sw && \
    apk del .build-deps && \
    rm -rf /var/cache/apk/* /tmp/* /var/tmp/* /root/.gnupg

FROM alpine:3.22 AS unbound
LABEL maintainer="MajorP93"

ENV UNBOUND_VERSION=1.24.1 \
    UNBOUND_SHA256=7f2b1633e239409619ae0527f67878b0f33ae0ec0ee5a3a51c042c359ba1eeab \
    UNBOUND_DOWNLOAD_URL=https://nlnetlabs.nl/downloads/unbound/

WORKDIR /tmp/src
COPY --from=openssl /opt/openssl /opt/openssl

RUN set -eux && \
    apk add --no-cache --virtual .build-deps \
      build-base \
      curl \
      libevent-dev \
      expat-dev \
      ldns-dev \
      nghttp2-dev \
      protobuf-c-dev \
      pkgconf \
      linux-headers && \
    apk add --no-cache ca-certificates && \
    curl -sSL "${UNBOUND_DOWNLOAD_URL}unbound-${UNBOUND_VERSION}.tar.gz" -o unbound.tar.gz && \
    echo "${UNBOUND_SHA256}  unbound.tar.gz" | sha256sum -c - && \
    tar xzf unbound.tar.gz && \
    cd "unbound-${UNBOUND_VERSION}" && \
    addgroup -S _unbound && \
    adduser -S -G _unbound -h /etc -s /sbin/nologin _unbound && \
    ./configure \
      --disable-dependency-tracking \
      --prefix=/opt/unbound \
      --with-pthreads \
      --with-username=_unbound \
      --with-ssl=/opt/openssl \
      --with-libevent \
      --with-libnghttp2 \
      --enable-dnstap \
      --enable-tfo-server \
      --enable-tfo-client \
      --enable-event-api \
      --enable-subnet && \
    make -j"$(getconf _NPROCESSORS_ONLN 2>/dev/null || echo 1)" install && \
    mv /opt/unbound/etc/unbound/unbound.conf /opt/unbound/etc/unbound/unbound.conf.example && \
    apk del .build-deps && \
    rm -rf /tmp/* /var/tmp/* /var/cache/apk/* /opt/unbound/share/man

FROM alpine:3.22
LABEL maintainer="MajorP93"

ENV NAME=unbound \
    SUMMARY="unbound is a validating, recursive, and caching DNS resolver." \
    DESCRIPTION="unbound is a validating, recursive, and caching DNS resolver." \
    PATH=/opt/unbound/sbin:"$PATH"

WORKDIR /tmp/src

COPY --from=unbound /opt /opt

RUN set -eux && \
    apk add --no-cache \
      bash \
      perl \
      coreutils \
      ca-certificates \
      drill \
      ldns-tools \
      libevent \
      nghttp2 \
      expat \
      protobuf-c && \
    addgroup -S _unbound && \
    adduser -S -G _unbound -h /etc -s /sbin/nologin _unbound && \
    rm -rf /var/cache/apk/* /tmp/*

COPY data/ /

RUN chmod +x /unbound.sh

WORKDIR /opt/unbound/

LABEL org.opencontainers.image.version=${UNBOUND_VERSION} \
      org.opencontainers.image.title="majorp93/unbound" \
      org.opencontainers.image.description="a validating, recursive, and caching DNS resolver" \
      org.opencontainers.image.url="https://github.com/MajorP93/unbound-docker" \
      org.opencontainers.image.vendor="MajorP93" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.source="https://github.com/MajorP93/unbound-docker"

EXPOSE 53/tcp
EXPOSE 53/udp

HEALTHCHECK --interval=30s --timeout=30s --start-period=10s --retries=3 CMD drill @127.0.0.1 cloudflare.com || exit 1

CMD ["/unbound.sh"]
