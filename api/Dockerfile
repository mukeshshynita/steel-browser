ARG NODE_VERSION=22.13.0

FROM node:${NODE_VERSION}-slim AS base

WORKDIR /app

ENV NODE_ENV="production" \
    PUPPETEER_CACHE_DIR=/app/.cache \
    DISPLAY=:10 \
    PATH="/usr/bin:/app/selenium/driver:${PATH}" \
    CHROME_BIN=/usr/bin/google-chrome-stable \
    CHROME_PATH=/usr/bin/google-chrome-stable

LABEL org.opencontainers.image.source="https://github.com/steel-dev/steel-browser"

# Install dependencies without cache mounts
RUN rm -f /etc/apt/apt.conf.d/docker-clean && \
    apt-get update -qq && \
    DEBIAN_FRONTEND=noninteractive apt-get -yq dist-upgrade

FROM base AS build

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
      build-essential \
      pkg-config \
      python-is-python3 \
      xvfb

COPY --link package-lock.json package.json ./

RUN npm ci --include=dev

COPY --link . .

RUN npm run build

RUN npm prune --omit=dev

FROM build AS patcher
WORKDIR /app/patcher
RUN npm i --include=dev
RUN node ./scripts/patcher.js patch --packagePath /app/node_modules/puppeteer-core

FROM base AS production

# Install dependencies
RUN apt-get update && \ 
    DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
      wget \
      nginx \
      gnupg \
      fonts-ipafont-gothic \
      fonts-wqy-zenhei \
      fonts-thai-tlwg \
      fonts-kacst \
      fonts-freefont-ttf \
      libxss1 \
      xvfb \
      curl \
      unzip \
      default-jre \
      dbus \
      dbus-x11

# Install Chrome and ChromeDriver
RUN curl -o chrome.deb https://mirror.cs.uchicago.edu/google-chrome/pool/main/g/google-chrome-stable/google-chrome-stable_128.0.6613.119-1_amd64.deb \
    && apt-get install -y ./chrome.deb \
    && rm chrome.deb \
    && mkdir -p /selenium/driver \
    && curl -o chromedriver.zip https://storage.googleapis.com/chrome-for-testing-public/128.0.6613.119/linux64/chromedriver-linux64.zip \
    && unzip chromedriver.zip -d /tmp \
    && mv /tmp/chromedriver-linux64/chromedriver /selenium/driver/chromedriver \
    && rm -rf chromedriver.zip /tmp/chromedriver-linux64 \
    && chmod +x /selenium/driver/chromedriver

COPY --chmod=755 entrypoint.sh /app/entrypoint.sh

EXPOSE 3000 9223

ENV HOST_IP=localhost \
    DBUS_SESSION_BUS_ADDRESS=autolaunch:

ENTRYPOINT ["/app/entrypoint.sh"]

COPY --from=patcher /app /app
