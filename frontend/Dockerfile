FROM node:16 as builder
WORKDIR /usr/src/app
ARG VUE_APP_API_URL=http://backend:8081/api
ARG NODE_ENV=dev
COPY . .
RUN npm install && npm run build && ls -la /usr/src/app

FROM nginx:stable
WORKDIR /app
COPY --from=builder /usr/src/app/dist/. /usr/share/nginx/html/
RUN ls -la /usr/share/nginx/html/ && echo "---" && ls -la /app/
EXPOSE 80