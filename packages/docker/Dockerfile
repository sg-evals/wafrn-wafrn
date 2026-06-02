# Shared build graph for frontend and backend

FROM --platform=$BUILDPLATFORM node:24-alpine AS frontend-builder

RUN apk add --no-cache perl curl tini bash git

RUN mkdir -p /app/packages/frontend
WORKDIR /app/packages/frontend

# install dependencies before setting NODE_ENV=production (needed for angular-devkit in devDependencies)
COPY package.json /app
COPY package-lock.json /app
COPY packages/frontend/package.json .
RUN npm install

# copy the rest of the files
COPY packages/frontend .

# copy git info too so the prebuild script can read it
COPY .git /app

# Override frontend files with files put here. Allows easy overriding of assets, and potentially actual code as well
RUN cp -af /app/packages/frontend/overrides/* /app/packages/frontend/src || true

# copy default ngsw config and use the frontend environment file for docker
RUN cp /app/packages/frontend/ngsw-config.json.example /app/packages/frontend/ngsw-config.json && \
  rm -rf src/environments/environment.prod.ts && \
  mv src/environments/environment.docker.ts src/environments/environment.prod.ts

ENV NODE_ENV=production

RUN npm run build && mv dist/wafrn/browser /app/frontend

FROM caddy:2 AS frontend

RUN apk add perl jq bash

COPY packages/frontend/Caddyfile /etc/caddy/Caddyfile
COPY --from=frontend-builder /app/packages/frontend/src/assets/pds-homepage.txt /pds-homepage/pds.txt
COPY --from=frontend-builder /app/frontend/ /usr/share/wafrn/frontend
COPY --from=frontend-builder /app/packages/frontend/overrides.sh /overrides.sh
COPY --from=frontend-builder /app/packages/frontend/docker-entrypoint.sh /docker-entrypoint.sh

RUN mkdir -p /var/www/html/frontend

VOLUME ["/overrides"]
VOLUME ["/var/www/html/frontend"]
VOLUME ["/var/www/html/uploads"]

ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]

FROM node:24-alpine AS backend

RUN apk add --no-cache perl curl tini bash ffmpeg exiv2

WORKDIR /app/packages/backend

COPY package.json /app
COPY package-lock.json /app
COPY packages/backend/package.json .

ENV NODE_ENV=production
ENV PORT=9000

RUN npm install

COPY packages/backend .
COPY packages/backend/environment.example.ts environment.ts

RUN mkdir -p logs /app/packages/frontend
COPY --from=frontend-builder /app/frontend /app/packages/frontend

VOLUME ["/app/packages/backend/uploads"]
VOLUME ["/app/packages/backend/cache"]

EXPOSE ${PORT}

ENTRYPOINT ["tini", "--"]

ENV NODE_OPTIONS=--dns-result-order=ipv4first

CMD ["npm", "start"]
