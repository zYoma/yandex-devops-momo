FROM node:16 as builder
WORKDIR /usr/src/app
COPY . .
RUN npm install && npm run build && ls -la /usr/src/app

FROM nginxproxy/nginx-proxy
WORKDIR /app
COPY --from=builder /usr/src/app/nginx.tmpl /app/nginx.tmpl
COPY --from=builder /usr/src/app/dist/frontend/. /usr/share/nginx/html/
RUN ls -la /usr/share/nginx/html/ && echo "---" && ls -la /app/
EXPOSE 80
