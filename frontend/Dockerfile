# Etapa builder
FROM node:20-alpine AS builder
WORKDIR /build
# Copiar solo manifiestos
COPY package.json package-lock.json ./
RUN npm ci
# Disponible en tiempo de compilacion por si vite lo inyecta en el bundle
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION} \
    VITE_APP_VERSION=${APP_VERSION}
# copiar solo el codigo fuente que consume vite
COPY index.html vite.config.js ./
COPY src/ ./src/
RUN npm run build

# Etapa servidor web nginx sin privilegios
FROM nginxinc/nginx-unprivileged:1.27-alpine
# configuracion resuelta al arrancar + version observable en el contenedor
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION} \
    API_HOST=catalogo-api \
    API_PORT=8000 \
    NGINX_ENVSUBST_FILTER=^API_
# solo el template y /dist
COPY --chown=nginx:nginx nginx/templates/default.conf.template /etc/nginx/templates/default.conf.template
COPY --from=builder --chown=nginx:nginx /build/dist /usr/share/nginx/html
# USER nginx (uid 101) y CMD heredados de la imagen base
EXPOSE 8080
