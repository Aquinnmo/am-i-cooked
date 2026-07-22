FROM node:20-bookworm-slim
WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

EXPOSE 5000
CMD ["node", "index.js"]
