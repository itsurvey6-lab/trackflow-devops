FROM node:22-alpine

RUN apk update && apk upgrade --no-cache

RUN npm install -g npm@latest

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY src ./src

RUN mkdir -p uploads

ENV NODE_ENV=production
ENV PORT=5000

EXPOSE 5000

CMD ["npm", "start"]
