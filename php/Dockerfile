FROM php:8.3-cli-bookworm

ARG CODEX_VERSION=0.142.0
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
    && rm -rf /var/lib/apt/lists/*

# Node.js for Codex CLI
RUN mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
        | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" \
        > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

# PHP extensions useful for Laravel / Symfony CLI work
RUN docker-php-ext-install -j"$(nproc)" \
    intl \
    pdo_mysql \
    mysqli \
    zip \
    bcmath \
    opcache

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# Codex CLI
RUN npm install -g @openai/codex@${CODEX_VERSION} \
    && npm cache clean --force

# Reuse the common node user pattern for consistency with the other image families
RUN useradd -m -u 1000 -s /bin/bash node \
    && mkdir -p /workspace /home/node/.codex /home/node/.local/bin /home/node/.composer /home/node/.cache/pip \
    && chown -R node:node /workspace /home/node

USER node

ENV HOME=/home/node
ENV PATH=/home/node/.local/bin:/usr/local/bin:/usr/bin:/bin
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
    && codex --version

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["codex"]