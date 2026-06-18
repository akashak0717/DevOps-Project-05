# ---------- Stage 1: Install dependencies ----------
FROM node:20-alpine AS deps

WORKDIR /app

# Copy only package files first (better layer caching)
COPY package.json package-lock.json ./

RUN npm ci --omit=dev

# ---------- Stage 2: Final runtime image ----------
FROM node:20-alpine AS runner

WORKDIR /app

# Run as non-root user for security
RUN addgroup -g 1001 -S nodejs && adduser -S nodeuser -u 1001

# Copy installed node_modules from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy application source code
COPY . .

# Drop privileges
USER nodeuser

EXPOSE 4000

ENV NODE_ENV=production

CMD ["node", "server.js"]