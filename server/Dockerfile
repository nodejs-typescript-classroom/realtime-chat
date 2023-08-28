# stage 1 - Build base
FROM node:18-alpine AS base
WORKDIR /app
COPY src ./src
COPY package*.json ./
COPY tsconfig*.json ./
RUN npm install
# stage 2 - build the app
FROM base AS build
WORKDIR /app
RUN npm run build
# stage 3 run app
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --only-production
COPY --from=build /app/build ./
ENTRYPOINT [ "node", "main.js" ]