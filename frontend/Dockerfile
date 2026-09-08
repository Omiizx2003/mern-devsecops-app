FROM node:20-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm ci --fetch-retries=5 --fetch-retry-mintimeout=20000 --fetch-retry-maxtimeout=120000
COPY . .

RUN npm run build


FROM nginx:alpine

RUN apk update && apk upgrade

COPY --from=build /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]