FROM node:20-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY package*.json ./
RUN if [ -f package-lock.json ]; then npm ci --omit=dev; else npm install --omit=dev; fi
COPY src ./src
# OpenShift ejecuta con un UID aleatorio del grupo 0: dar permisos de lectura al grupo
RUN chgrp -R 0 /app && chmod -R g=u /app
USER 1001
EXPOSE 3000
CMD ["node","src/server.js"]
