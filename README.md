# syntax=docker/dockerfile:1

# ---------- Estágio 1: instala só as dependências de produção ----------
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# ---------- Estágio 2: imagem final, enxuta e sem root ----------
FROM node:20-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app

# Instala o curl para o healthcheck (Alpine usa apk)
RUN apk add --no-cache curl

COPY --from=deps /app/node_modules ./node_modules
COPY package.json ./
COPY src ./src

# Altera a propriedade dos arquivos para o usuário node se sua aplicação precisar escrever no app
# COPY --chown=node:node package.json ./
# COPY --chown=node:node src ./src

# A imagem node já traz o usuário "node" (sem privilégios de root)
USER node
EXPOSE 3000

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://127.0.0.1:${PORT:-3000}/health || exit 1

CMD ["node", "src/server.js"]
