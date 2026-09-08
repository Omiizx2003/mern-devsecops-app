# Build stage
FROM node:20-alpine AS build

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

# Runtime stage
FROM node:20-alpine

WORKDIR /usr/src/app
RUN apk update && apk upgrade

COPY --from=build /usr/src/app/node_modules ./node_modules
COPY --from=build /usr/src/app/package.json ./package.json
COPY --from=build /usr/src/app/index.js ./index.js
COPY --from=build /usr/src/app/db.js ./db.js
COPY --from=build /usr/src/app/models ./models
COPY --from=build /usr/src/app/routes ./routes

# Remove npm and npx from runtime image
RUN rm -rf /usr/local/lib/node_modules/npm \
    /usr/local/bin/npm \
    /usr/local/bin/npx

EXPOSE 3500

CMD ["node", "index.js"]
