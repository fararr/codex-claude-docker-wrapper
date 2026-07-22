# Dockerfile  (claudecode/Dockerfile)
FROM php:8.3-cli-bookworm

# Pin the CLI version. Bump deliberately, not automatically.
ARG CLAUDE_CODE_VERSION=2.1.202
# npm-installed Claude Code requires Node >= 22 (as of v2.1.198). 24 is fine.
ARG NODE_MAJOR=24

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    bash \
    ca-certificates \
    curl \
    fd-find \
    git \
    gpg \
    jq \
    less \
    make \
    patch \
    procps \
    python3 \
    python3-pip \
    python3-venv \
    ripgrep \
    shellcheck \
    sqlite3 \
    tini \
    tree \
    unzip \
    zip \
    libicu-dev \
    libzip-dev \
    libxml2-dev \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    libwebp-dev \
    libgmp-dev \
    libpq-dev \
    libldap2-dev \
    && rm -rf /var/lib/apt/lists/*

# Node.js — required to *install* the npm package (the claude binary itself is
# native and does not invoke Node at runtime).
RUN mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
        | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" \
        > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

# PHP extensions for Laravel / Symfony / general PHP CLI work.
# gd, gmp, pgsql and ldap link against the -dev libs installed in the apt-get
# block above, so that block MUST run before this one.
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-configure ldap --with-libdir=lib/"$(dpkg-architecture -qDEB_HOST_MULTIARCH)" \
    && docker-php-ext-install -j"$(nproc)" \
        bcmath \
        calendar \
        exif \
        gd \
        gmp \
        intl \
        ldap \
        mysqli \
        opcache \
        pcntl \
        pdo_mysql \
        pdo_pgsql \
        pgsql \
        soap \
        sockets \
        zip

# PECL: redis is near-ubiquitous for Laravel cache / queue / session.
RUN pecl install redis \
    && docker-php-ext-enable redis \
    && rm -rf /tmp/pear

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# Claude Code CLI.
# npm pulls the native binary via a per-platform optional dependency
# (e.g. @anthropic-ai/claude-code-linux-x64) and links `claude` into place.
RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION} \
    && npm cache clean --force

# Same "node" user convention as the rest of your image family.
RUN useradd -m -u 1000 -s /bin/bash node \
    && mkdir -p /workspace /home/node/.claude /home/node/.local/bin /home/node/.composer /home/node/.cache/pip \
    && chown -R node:node /workspace /home/node

USER node

ENV HOME=/home/node
ENV PATH=/home/node/.local/bin:/usr/local/bin:/usr/bin:/bin

# Consolidate ALL Claude Code state (settings, .credentials.json, .claude.json,
# projects/, todos/, shell-snapshots/) under one directory so a single mount
# persists everything — including your auth token. Without this, config splits
# between ~/.claude/ and the ~/.claude.json file at HOME root.
ENV CLAUDE_CONFIG_DIR=/home/node/.claude

# Keep the pinned version deterministic; don't let the binary self-update.
ENV DISABLE_AUTOUPDATER=1
# Optional: drop telemetry / statsig / update pings (tighter egress).
ENV CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1

ENV COMPOSER_HOME=/home/node/.composer
ENV COMPOSER_CACHE_DIR=/home/node/.composer/cache
ENV PIP_CACHE_DIR=/home/node/.cache/pip
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /workspace

RUN php -v \
    && node -v \
    && npm -v \
    && composer --version \
    && python3 --version \
    && claude --version

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["claude"]