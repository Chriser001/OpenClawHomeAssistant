ARG BUILD_FROM
FROM ${BUILD_FROM}

# Switch to USTC mirror for faster apt in China
RUN sed -i 's|http://deb.debian.org/debian|http://mirrors.ustc.edu.cn/debian|g' /etc/apt/sources.list.d/debian.sources 2>/dev/null || \
    sed -i 's|http://deb.debian.org/debian|http://mirrors.ustc.edu.cn/debian|g' /etc/apt/sources.list

# Base image is Debian Bookworm (glibc). This avoids musl-related native module issues
# that occur on Alpine (e.g. clipboard, node-llama-cpp).

# Install base packages (without nodejs/npm - we'll get Node 22 binary from npmmirror)
# build-essential provides gcc/cc needed by Homebrew for OpenClaw's brew-managed dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    bash \
    git \
    openssh-client \
    ca-certificates \
    tzdata \
    jq \
    curl \
    tar \
    xz-utils \
    file \
    python3 \
    cmake \
    nginx \
    gnupg \
    build-essential \
    sudo \
    vim \
    nano \
    fd-find \
    ripgrep \
    rsync \
    bat \
    less \
    openssl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js 22 from binary (via npmmirror.com mirror) instead of nodesource apt repo,
# which is no longer mirrored by Chinese mainland mirrors (USTC/Tsinghua all stopped syncing).
# Docker TARGETARCH: amd64, arm64, arm/v7 → node filenames: x64, arm64, armv7l
ARG TARGETARCH
RUN ARCH=$(echo "${TARGETARCH:-amd64}" | sed 's|amd64|x64|;s|arm64|arm64|;s|arm/v7|armv7l|') \
    && NODE_VERSION="22.20.0" \
    && echo "Downloading Node.js v${NODE_VERSION} for linux-${ARCH} from npmmirror" \
    && curl -fsSL "https://npmmirror.com/mirrors/node/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-${ARCH}.tar.xz" -o /tmp/node.tar.xz \
    && tar -xf /tmp/node.tar.xz -C /usr/local --strip-components=1 \
    && rm /tmp/node.tar.xz \
    && node -v | grep -E '^v22\.' \
    && npm -v

# Install ttyd (web terminal) - not in Debian repos, download binary
# Docker TARGETARCH: amd64, arm64, arm/v7 → ttyd filenames: x86_64, aarch64, armhf
ARG TARGETARCH
RUN ARCH=$(echo ${TARGETARCH:-$(dpkg --print-architecture)} | sed 's|arm64|aarch64|;s|arm/v7|armhf|;s|armv7|armhf|;s|amd64|x86_64|') \
    && echo "Downloading ttyd for arch: ${ARCH}" \
    && curl -fsSL "https://ghfast.top/https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.${ARCH}" -o /usr/local/bin/ttyd \
    && chmod +x /usr/local/bin/ttyd

RUN node -v && npm -v

# Install pnpm globally (required by some OpenClaw skills like clawhub)
RUN npm install -g pnpm && pnpm -v

# Install Chromium for website automation tasks
# Includes necessary dependencies for headless browser operation
RUN apt-get update && apt-get install -y --no-install-recommends \
    chromium \
    chromium-driver \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Homebrew (Linuxbrew) for OpenClaw skill dependencies
# Homebrew is optional - some skills need CLI tools like gemini, aider, etc.
# NOTE: Homebrew requires CPU with SSSE3 support (Intel Core 2 or newer, ~2006+)
# If installation fails (e.g., older CPUs), the add-on will still work but some skills may not install
ENV HOMEBREW_NO_AUTO_UPDATE=1 \
    HOMEBREW_NO_INSTALL_CLEANUP=1 \
    HOMEBREW_NO_ANALYTICS=1

RUN useradd -m -s /bin/bash linuxbrew \
    && mkdir -p /home/linuxbrew/.linuxbrew \
    && chown -R linuxbrew:linuxbrew /home/linuxbrew

# Make git clone of GitHub repos use ghfast.top proxy
USER linuxbrew
RUN git config --global url."https://ghfast.top/https://github.com/".insteadOf "https://github.com/"
USER root

USER linuxbrew
# Shallow clone Homebrew (optional add-on feature - some skills need brew CLI tools)
# Note: --version and brew update are skipped because they download portable ruby from
# ghcr.io (not mirrored), which blocks on Chinese networks. The shallow clone alone
# provides a working brew for installing CLI tools.
RUN git clone --depth=1 https://ghfast.top/https://github.com/Homebrew/brew /home/linuxbrew/.linuxbrew/Homebrew \
    && mkdir -p /home/linuxbrew/.linuxbrew/bin \
    && ln -sf ../Homebrew/bin/brew /home/linuxbrew/.linuxbrew/bin/brew \
    || echo "WARNING: Homebrew installation failed. Some skills may not work."
USER root

# Add Homebrew to PATH for all users (wrapper comes first to intercept root calls)
# PATH is set even if brew failed - wrapper will handle missing brew gracefully
ENV PATH="/usr/local/bin:/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:${PATH}"

# Copy brew wrapper that allows root to run brew by delegating to linuxbrew user
COPY brew-wrapper.sh /usr/local/bin/brew
RUN chmod +x /usr/local/bin/brew

# Verify brew binary exists (skip --version to avoid triggering portable ruby download)
USER linuxbrew
RUN if [ -x /home/linuxbrew/.linuxbrew/bin/brew ]; then \
    echo "Homebrew installed successfully"; \
    else \
    echo "Homebrew not available"; \
    fi
USER root

# Install OpenClaw globally
# Bundle node-llama-cpp in the image so the default local memory/embeddings
# provider works in HAOS without requiring manual npm installs in /usr/lib.
RUN npm config set fund false && npm config set audit false \
    && npm config set registry https://registry.npmmirror.com \
    && npm install -g openclaw@2026.5.28 node-llama-cpp@3.18.1

# Shell aliases and color options for interactive use
RUN tee -a /etc/bash.bashrc <<'EOF'

# Aliases
alias bat="batcat"
alias fd="fdfind"

# Enable colors
alias ls="ls --color=auto"
alias grep="grep --color=auto"
alias egrep="grep -E --color=auto"
alias fgrep="grep -F --color=auto"
alias diff="diff --color=auto"
alias ip="ip --color=auto"
if [ -z "${LS_COLORS+x}" ]; then
  export LS_COLORS="di=1;34:ln=1;36:so=1;pi=33:ex=1;32:bd=1;33:cd=1;33:su=1;31:sg=1;31:tw=1;34:ow=1;34"
fi
EOF
COPY run.sh /run.sh
COPY oc_config_helper.py /oc_config_helper.py
COPY render_nginx.py /render_nginx.py
COPY oc-cleanup.sh /usr/local/bin/oc-cleanup
COPY oc-gateway /usr/local/bin/oc-gateway
COPY openclaw-proxy-shim.cjs /usr/local/lib/openclaw-proxy-shim.cjs
COPY nginx.conf.tpl /etc/nginx/nginx.conf.tpl
COPY landing.html.tpl /etc/nginx/landing.html.tpl
RUN chmod +x /run.sh /oc_config_helper.py /render_nginx.py /usr/local/bin/oc-cleanup /usr/local/bin/oc-gateway \
    && mkdir -p /run/nginx

CMD [ "/run.sh" ]
