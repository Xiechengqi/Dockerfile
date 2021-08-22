
****
<details><summary>展开</summary><pre><code>

``` yaml

```
</code></pre></details>
----

****
<details><summary>展开</summary><pre><code>

``` yaml

```
</code></pre></details>
----

**https://github.com/tuna/freedns-go/blob/master/Dockerfile**

<details><summary>展开</summary><pre><code>

``` yaml
FROM python:alpine as update_db
WORKDIR /usr/src/app
COPY chinaip .
RUN pip3 install -r requirements.txt
RUN python3 update_db.py

FROM golang:alpine as builder
WORKDIR /go/src/github.com/tuna/freedns-go
COPY go.* ./
RUN go mod download
COPY . .
COPY --from=update_db /usr/src/app/db.go chinaip/
RUN go build -o ./build/freedns-go

FROM alpine
COPY --from=builder /go/src/github.com/tuna/freedns-go/build/freedns-go ./
ENTRYPOINT ["./freedns-go"]
CMD ["-f", "114.114.114.114:53", "-c", "8.8.8.8:53", "-l", "0.0.0.0:53"]
```
</code></pre>
</details>

----
**https://github.com/cptactionhank/docker-atlassian-confluence/blob/master/Dockerfile**
<details><summary>展开</summary><pre><code>

``` yaml
FROM openjdk:8-alpine

# Setup useful environment variables
ENV CONF_HOME     /var/atlassian/confluence
ENV CONF_INSTALL  /opt/atlassian/confluence
ENV CONF_VERSION  7.9.3

ENV JAVA_CACERTS  $JAVA_HOME/jre/lib/security/cacerts
ENV CERTIFICATE   $CONF_HOME/certificate

# Install Atlassian Confluence and helper tools and setup initial home
# directory structure.
RUN set -x \
    && apk --no-cache add curl xmlstarlet bash ttf-dejavu libc6-compat gcompat \
    && mkdir -p                "${CONF_HOME}" \
    && chmod -R 700            "${CONF_HOME}" \
    && chown daemon:daemon     "${CONF_HOME}" \
    && mkdir -p                "${CONF_INSTALL}/conf" \
    && curl -Ls                "https://www.atlassian.com/software/confluence/downloads/binary/atlassian-confluence-${CONF_VERSION}.tar.gz" | tar -xz --directory "${CONF_INSTALL}" --strip-components=1 --no-same-owner \
    && curl -Ls                "https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.44.tar.gz" | tar -xz --directory "${CONF_INSTALL}/confluence/WEB-INF/lib" --strip-components=1 --no-same-owner "mysql-connector-java-5.1.44/mysql-connector-java-5.1.44-bin.jar" \
    && chmod -R 700            "${CONF_INSTALL}/conf" \
    && chmod -R 700            "${CONF_INSTALL}/temp" \
    && chmod -R 700            "${CONF_INSTALL}/logs" \
    && chmod -R 700            "${CONF_INSTALL}/work" \
    && chown -R daemon:daemon  "${CONF_INSTALL}/conf" \
    && chown -R daemon:daemon  "${CONF_INSTALL}/temp" \
    && chown -R daemon:daemon  "${CONF_INSTALL}/logs" \
    && chown -R daemon:daemon  "${CONF_INSTALL}/work" \
    && echo -e                 "\nconfluence.home=$CONF_HOME" >> "${CONF_INSTALL}/confluence/WEB-INF/classes/confluence-init.properties" \
    && xmlstarlet              ed --inplace \
        --delete               "Server/@debug" \
        --delete               "Server/Service/Connector/@debug" \
        --delete               "Server/Service/Connector/@useURIValidationHack" \
        --delete               "Server/Service/Connector/@minProcessors" \
        --delete               "Server/Service/Connector/@maxProcessors" \
        --delete               "Server/Service/Engine/@debug" \
        --delete               "Server/Service/Engine/Host/@debug" \
        --delete               "Server/Service/Engine/Host/Context/@debug" \
                               "${CONF_INSTALL}/conf/server.xml" \
    && touch -d "@0"           "${CONF_INSTALL}/conf/server.xml" \
    && chown daemon:daemon     "${JAVA_CACERTS}"

# Use the default unprivileged account. This could be considered bad practice
# on systems where multiple processes end up being executed by 'daemon' but
# here we only ever run one process anyway.
USER daemon:daemon

# Expose default HTTP connector port.
EXPOSE 8090 8091

# Set volume mount points for installation and home directory. Changes to the
# home directory needs to be persisted as well as parts of the installation
# directory due to eg. logs.
VOLUME ["/var/atlassian/confluence", "/opt/atlassian/confluence/logs"]

# Set the default working directory as the Confluence home directory.
WORKDIR /var/atlassian/confluence

COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]

# Run Atlassian Confluence as a foreground process by default.
CMD ["/opt/atlassian/confluence/bin/start-confluence.sh", "-fg"]
```
</code></pre></details>

----
**https://github.com/jc21/nginx-proxy-manager/blob/develop/docker/Dockerfile**
<details><summary>展开</summary><pre><code>

``` yaml
# This is a Dockerfile intended to be built using `docker buildx`
# for multi-arch support. Building with `docker build` may have unexpected results.

# This file assumes that the frontend has been built using ./scripts/frontend-build

FROM nginxproxymanager/nginx-full:node

ARG TARGETPLATFORM
ARG BUILD_VERSION
ARG BUILD_COMMIT
ARG BUILD_DATE

ENV SUPPRESS_NO_CONFIG_WARNING=1 \
	S6_FIX_ATTRS_HIDDEN=1 \
	S6_BEHAVIOUR_IF_STAGE2_FAILS=1 \
	NODE_ENV=production \
	NPM_BUILD_VERSION="${BUILD_VERSION}" \
	NPM_BUILD_COMMIT="${BUILD_COMMIT}" \
	NPM_BUILD_DATE="${BUILD_DATE}"

RUN echo "fs.file-max = 65535" > /etc/sysctl.conf \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends jq logrotate \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/lists/*

# s6 overlay
COPY scripts/install-s6 /tmp/install-s6
RUN /tmp/install-s6 "${TARGETPLATFORM}" && rm -f /tmp/install-s6

EXPOSE 80 81 443

COPY backend       /app
COPY frontend/dist /app/frontend
COPY global        /app/global

WORKDIR /app
RUN yarn install

# add late to limit cache-busting by modifications
COPY docker/rootfs /

# Remove frontend service not required for prod, dev nginx config as well
RUN rm -rf /etc/services.d/frontend /etc/nginx/conf.d/dev.conf

# Change permission of logrotate config file
RUN chmod 644 /etc/logrotate.d/nginx-proxy-manager

VOLUME [ "/data", "/etc/letsencrypt" ]
ENTRYPOINT [ "/init" ]
HEALTHCHECK --interval=5s --timeout=3s CMD /bin/check-health

LABEL org.label-schema.schema-version="1.0" \
	org.label-schema.license="MIT" \
	org.label-schema.name="nginx-proxy-manager" \
	org.label-schema.description="Docker container for managing Nginx proxy hosts with a simple, powerful interface " \
	org.label-schema.url="https://github.com/jc21/nginx-proxy-manager" \
	org.label-schema.vcs-url="https://github.com/jc21/nginx-proxy-manager.git" \
	org.label-schema.cmd="docker run --rm -ti jc21/nginx-proxy-manager:latest"
```
</code></pre></details>

----
**https://github.com/rtCamp/action-slack-notify/blob/master/Dockerfile**
<details><summary>展开</summary><pre><code>

``` yaml
FROM golang:1.14-alpine3.11@sha256:6578dc0c1bde86ccef90e23da3cdaa77fe9208d23c1bb31d942c8b663a519fa5 AS builder

LABEL "com.github.actions.icon"="bell"
LABEL "com.github.actions.color"="yellow"
LABEL "com.github.actions.name"="Slack Notify"
LABEL "com.github.actions.description"="This action will send notification to Slack"
LABEL "org.opencontainers.image.source"="https://github.com/rtCamp/action-slack-notify"

WORKDIR ${GOPATH}/src/github.com/rtcamp/action-slack-notify
COPY main.go ${GOPATH}/src/github.com/rtcamp/action-slack-notify

ENV CGO_ENABLED 0
ENV GOOS linux

RUN go get -v ./...
RUN go build -a -installsuffix cgo -ldflags '-w  -extldflags "-static"' -o /go/bin/slack-notify .

# alpine:latest at 2020-01-18T01:19:37.187497623Z
FROM alpine@sha256:ab00606a42621fb68f2ed6ad3c88be54397f981a7b70a79db3d1172b11c4367d

COPY --from=builder /go/bin/slack-notify /usr/bin/slack-notify

ENV VAULT_VERSION 1.0.2

RUN apk update \
	&& apk upgrade \
	&& apk add \
	bash \
	jq \
	ca-certificates \
	python \
	py2-pip \
	rsync && \
	pip install shyaml && \
	rm -rf /var/cache/apk/*

# Setup Vault
RUN wget https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip && \
	unzip vault_${VAULT_VERSION}_linux_amd64.zip && \
	rm vault_${VAULT_VERSION}_linux_amd64.zip && \
	mv vault /usr/local/bin/vault

# fix the missing dependency - https://stackoverflow.com/a/35613430
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2

COPY *.sh /

RUN chmod +x /*.sh

ENTRYPOINT ["/entrypoint.sh"]
```
</code></pre></details>

----
**https://github.com/tanmng/docker-chevereto/blob/master/Dockerfile**
<details><summary>展开</summary><pre><code>

``` yaml
# Specify the version of PHP we use for our Chevereto
ARG PHP_VERSION=7.4-apache
FROM alpine as downloader

ARG CHEVERETO_VERSION=1.3.0
RUN apk add --no-cache curl && \
    curl -sS -o /tmp/chevereto.zip -L "https://github.com/Chevereto/Chevereto-Free/archive/${CHEVERETO_VERSION}.zip" && \
    mkdir -p /extracted && \
    cd /extracted && \
    unzip /tmp/chevereto.zip  && \
    mv "Chevereto-Free-${CHEVERETO_VERSION}/" Chevereto/
COPY settings.php /extracted/Chevereto/app/settings.php

FROM php:$PHP_VERSION

# Install required packages and configure plugins + mods for Chevereto
RUN apt-get update \
    && apt-get install -y \
        libgd-dev \
        libwebp-dev \
        libzip-dev \
    && bash -c 'if [[ $PHP_VERSION == 7.4.* ]]; then \
      docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/ --with-webp; \
    else \
      docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/  --with-webp-dir=/usr/include/; \
    fi' && \
    docker-php-ext-install \
        exif \
        gd \
        mysqli \
        pdo \
        pdo_mysql \
        zip && \
    a2enmod rewrite

# Download installer script
COPY --from=downloader --chown=33:33 /extracted/Chevereto /var/www/html

# Expose the image directory as a volume
VOLUME /var/www/html/images

# DB connection environment variables
ENV CHEVERETO_DB_HOST=db CHEVERETO_DB_USERNAME=chevereto CHEVERETO_DB_PASSWORD=chevereto CHEVERETO_DB_NAME=chevereto CHEVERETO_DB_PREFIX=chv_ CHEVERETO_DB_PORT=3306
ARG BUILD_DATE
ARG CHEVERETO_VERSION=1.2.2

# Set all required labels, we set it here to make sure the file is as reusable as possible
LABEL org.label-schema.url="https://github.com/tanmng/docker-chevereto" \
      org.label-schema.name="Chevereto Free" \
      org.label-schema.license="Apache-2.0" \
      org.label-schema.version="${CHEVERETO_VERSION}" \
      org.label-schema.vcs-url="https://github.com/tanmng/docker-chevereto" \
      maintainer="Tan Nguyen <tan.mng90@gmail.com>" \
      build_signature="Chevereto free version ${CHEVERETO_VERSION}; built on ${BUILD_DATE}; Using PHP version ${PHP_VERSION}"
```
</code></pre></details>

----
**https://github.com/tuna/freedns-go/blob/master/Dockerfile**
<details><summary>展开</summary><pre><code>

``` yaml
FROM python:alpine as update_db
WORKDIR /usr/src/app
COPY chinaip .
RUN pip3 install -r requirements.txt
RUN python3 update_db.py

FROM golang:alpine as builder
WORKDIR /go/src/github.com/tuna/freedns-go
COPY go.* ./
RUN go mod download
COPY . .
COPY --from=update_db /usr/src/app/db.go chinaip/
RUN go build -o ./build/freedns-go


FROM alpine
COPY --from=builder /go/src/github.com/tuna/freedns-go/build/freedns-go ./
ENTRYPOINT ["./freedns-go"]
CMD ["-f", "114.114.114.114:53", "-c", "8.8.8.8:53", "-l", "0.0.0.0:53"]
```
</code></pre></details>

----	
**https://github.com/sorenisanerd/gotty/blob/master/Dockerfile**
<details><summary>展开</summary><pre><code>

``` yaml
FROM golang:1.13.1

WORKDIR /gotty
COPY . /gotty
RUN CGO_ENABLED=0 make

FROM alpine:latest

RUN apk update && \
    apk upgrade && \
    apk --no-cache add ca-certificates && \
    apk add bash
WORKDIR /root
COPY --from=0 /gotty/gotty /usr/bin/
CMD ["gotty",  "-w", "bash"]
```
</code></pre></details>
