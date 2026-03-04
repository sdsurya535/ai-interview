FROM node:22-alpine

# Install Chromium from Alpine's package manager.
# This is much smaller than Puppeteer's bundled Chromium download (~120MB vs ~300MB).
RUN apk add --no-cache \
    chromium \
    nss \
    freetype \
    harfbuzz \
    ca-certificates \
    ttf-freefont \
    font-noto-cjk

# Tell Puppeteer to skip downloading its own Chromium
# and point it to the Alpine-installed one instead
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser

WORKDIR /app

# ---- Deps ----
COPY package*.json ./
RUN npm ci --omit=dev

# ---- Source ----
COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
