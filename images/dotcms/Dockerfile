# OpenJDK distributed under GPLv2+Oracle Classpath Exception license (http://openjdk.java.net/legal/gplv2+ce.html)
# Alpine Linux packages distributed under various licenses including GPL-3.0+ (https://pkgs.alpinelinux.org/packages)
FROM dotcms/dotcms-seed as build-cms

LABEL com.dotcms.contact "support@dotcms.com"
LABEL com.dotcms.vendor "dotCMS LLC"
LABEL com.dotcms.description "dotCMS Content Management System"

#  One of ["TARBALL_URL", "RELEASE", "NIGHTLY", "COMMIT", "TAG" ]
ARG BUILD_FROM=COMMIT

#  Value resolved in the context of $BUILD_FROM
ARG BUILD_ID=HEAD

# Tomcat version
ARG TOMCAT_VERSION=""

WORKDIR /srv

# dotCMS core distributed under GPLv3 license (https://github.com/dotCMS/core/blob/master/license.txt)
COPY build-src/ /build/
RUN chmod 500 /build/build_dotcms.sh && /build/build_dotcms.sh ${BUILD_FROM} ${BUILD_ID} ${TOMCAT_VERSION}

RUN mkdir -p /srv/utils /srv/templates /srv/config /srv/home
RUN chmod -R 666 /srv && find /srv/ -type d -exec chmod a+x {} \;

##
## Begin target image build
##

# OpenJDK distributed under GPLv2+Oracle Classpath Exception license (http://openjdk.java.net/legal/gplv2+ce.html)
# Alpine Linux packages distributed under various licenses including GPL-3.0+ (https://pkgs.alpinelinux.org/packages)
#ARG ARCH=arm64v8

FROM openjdk:8-jdk-alpine

WORKDIR /srv

RUN apk --no-cache upgrade \
    && apk add --no-cache bash openssl ca-certificates gnupg grep sed tini nss s6-dns go libpq git gcc musl-dev libc6-compat tomcat-native curl \
    && update-ca-certificates

RUN apk add libsass --no-cache --repository http://dl-cdn.alpinelinux.org/alpine/edge/community/

# This is a nasty hack for libjsass, Alpine 3.9 no longer has a deprecated symlink in /lib
RUN [ ! -e /lib/ld-linux-x86-64.so.2 ] &&  ln -s $(readlink /lib64/ld-linux-x86-64.so.2) /lib/ld-linux-x86-64.so.2 || :

# Download and build dockerize distributed under MIT license (https://github.com/jwilder/dockerize)
RUN wget --quiet https://github.com/jwilder/dockerize/archive/v0.6.1.tar.gz \
    && tar -C / -xzf v0.6.1.tar.gz  \
    && cd /dockerize-0.6.1 \
    && go get github.com/hpcloud/tail \
    && go get github.com/jwilder/gojq \
    && go get golang.org/x/net/context \
    && go build \
    && cp dockerize-0.6.1 /usr/local/bin/dockerize \
    && chmod a+x /usr/local/bin/dockerize

RUN cd / && rm -rf  /dockerize-0.6.1

RUN apk del -r --purge git gcc musl-dev go



COPY --from=build-cms /srv/ /srv/
COPY ROOT/ /

# Make scripts runable
RUN find /srv/ -type f -name "*.sh" -exec chmod a+x {} \;

# Make plugin merging directories writable
RUN find /srv/templates -type d -exec chmod 770 {} \;

RUN mkdir -p /data/shared/assets && mkdir -p /data/local/dotsecure/license && chmod -R 660 /data && find /data/ -type d -exec chmod 770 {} \;

ENV SERVICE_DELAY_DEFAULT_MIN 1
ENV SERVICE_DELAY_DEFAULT_STEP 3
ENV SERVICE_DELAY_DEFAULT_MAX 30

#VOLUME /data/local

## Ports
# Direct connect
EXPOSE 8080
# Connect from proxy, HTTP/80, non-secure
EXPOSE 8081
# Connect from proxy, HTTPS/443, secure
EXPOSE 8082
# Java Debugging - must be configured via CMS_JAVA_OPTS
EXPOSE 8000
# Glowroot profiler - must be configured via CMS_JAVA_OPTS
EXPOSE 4000

USER 1000000000

ENTRYPOINT ["/sbin/tini", "--", "/srv/entrypoint.sh"]
CMD ["dotcms"]