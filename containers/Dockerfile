FROM docker.io/library/alpine:edge AS git
RUN apk update && apk add --no-cache \
    git \
    alpine-base \
    build-base \
    python3-dev \
    py3-pip
WORKDIR /
RUN git -c advice.detachedHead=false clone --branch main --depth=1 --recurse-submodules --shallow-submodules https://github.com/pfm-powerforme/searxng.git source
WORKDIR /source/
RUN git update-index -q --refresh
RUN --mount=type=cache,id=pip,target=/root/.cache/pip \
 python -m venv venv \
 && source venv/bin/activate \
 && pip install -r requirements.txt \
 && pip install -r requirements-server.txt \
 && pip install -r requirements-dev.txt
RUN source venv/bin/activate \
    && python -m searx.version freeze \
	&& eval "$(python -m searx.version)"
RUN rm -rf venv
	

FROM docker.io/library/alpine:edge AS builder
### Base Start ###
RUN apk update && apk add --no-cache \
    git \
    alpine-base \
    build-base \
    python3-dev \
    py3-pip \
    brotli \
    libffi-dev
ENV PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
    SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt" \
    HISTFILE="/dev/null"
COPY --from=git /source/ /usr/local/searxng/
WORKDIR /usr/local/searxng/
### Base End ###
RUN --mount=type=cache,id=pip,target=/root/.cache/pip \
 python -m venv .venv \
 && source .venv/bin/activate \
 && pip install -r requirements.txt \
 && pip install -r requirements-server.txt
RUN set -eux; \
    python -m compileall -q ./searx/; \
    touch -c --date=@$(git log -1 --format="%cd" --date=unix -- ./searx/settings.yml) ./searx/settings.yml; \
    find ./searx/static/ -type f \
        \( -name "*.html" -o -name "*.css" -o -name "*.js" -o -name "*.svg" \) \
        -exec gzip -9 -k {} + \
        -exec brotli -9 -k {} + \
        -exec gzip --test {}.gz + \
        -exec brotli --test {}.br +;


FROM docker.io/library/alpine:edge AS runtime
### Base Start ###
RUN apk update && apk add --no-cache \
    alpine-baselayout \
    ca-certificates-bundle \
    busybox \
    python3 \
    wget \
    libxml2 \
    mailcap
RUN addgroup -g 977 searxng && \
    adduser -D -u 977 -G searxng -s /bin/ash searxng
RUN mkdir -p /usr/local/searxng/ /etc/searxng/ /var/cache/searxng/ && \
    chown searxng:searxng /usr/local/searxng/ /etc/searxng/ /var/cache/searxng/ && \
    chmod 0555 /usr/local/searxng/ && \
    chmod 0755 /etc/searxng/ /var/cache/searxng/
ENV PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
    SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt" \
    HISTFILE="/dev/null" \
    CONFIG_PATH="/etc/searxng" \
    DATA_PATH="/var/cache/searxng"
WORKDIR /usr/local/searxng/
### Base End ###
COPY --chown=searxng:searxng --from=builder /usr/local/searxng/.venv/ .venv/
COPY --chown=searxng:searxng --from=builder /usr/local/searxng/searx/ searx/
COPY --chown=searxng:searxng --from=git /source/container/entrypoint.sh entrypoint.sh
COPY searxng/settings.yml /etc/searxng/settings.yml
COPY searxng/limiter.toml /etc/searxng/limiter.toml
COPY searxng/favicons.toml /etc/searxng/favicons.toml
ENV SEARXNG_VERSION="latest" \
	 SEARXNG_SETTINGS_PATH="$CONFIG_PATH/settings.yml" \
     GRANIAN_PROCESS_NAME="searxng" \
     GRANIAN_INTERFACE="wsgi" \
     GRANIAN_HOST="::" \
     GRANIAN_PORT="8080" \
     GRANIAN_WEBSOCKETS="false" \
     GRANIAN_LOOP="uvloop" \
     GRANIAN_BLOCKING_THREADS="4" \
     GRANIAN_WORKERS_KILL_TIMEOUT="30s" \
     GRANIAN_BLOCKING_THREADS_IDLE_TIMEOUT="5m"
VOLUME $CONFIG_PATH
VOLUME $DATA_PATH
EXPOSE 8080
ENTRYPOINT ["/usr/local/searxng/entrypoint.sh"]