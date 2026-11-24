# ---- Etapa 1: compilar Angular ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN if [ -f package-lock.json ]; then npm ci; else npm install; fi
COPY . .
RUN npm run build -- --configuration production

# ---- Etapa 2: Nginx sin privilegios (compatible con OpenShift: UID aleatorio, puerto 8080) ----
FROM nginxinc/nginx-unprivileged:stable-alpine
COPY --from=build /app/dist/logistica-mundial/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
