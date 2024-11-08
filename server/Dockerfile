# dev build
FROM ghcr.io/immich-app/base-server-dev:20241029@sha256:31c2d6cba42253680ff7738dffa66e97dfe5b1c7faf23b29d07ab456e9c1e2b9 AS dev

RUN apt-get install --no-install-recommends -yqq tini
WORKDIR /usr/src/app
COPY server/package.json server/package-lock.json ./
RUN npm ci && \
  # exiftool-vendored.pl, sharp-linux-x64 and sharp-linux-arm64 are the only ones we need
  # they're marked as optional dependencies, so we need to copy them manually after pruning
  rm -rf node_modules/@img/sharp-libvips* && \
  rm -rf node_modules/@img/sharp-linuxmusl-x64
COPY server .
ENV PATH="${PATH}:/usr/src/app/bin" \
  IMMICH_ENV=development \
  NVIDIA_DRIVER_CAPABILITIES=all \
  NVIDIA_VISIBLE_DEVICES=all
ENTRYPOINT ["tini", "--", "/bin/sh"]


FROM dev AS prod

RUN npm run build
RUN npm prune --omit=dev --omit=optional
COPY --from=dev /usr/src/app/node_modules/@img ./node_modules/@img
COPY --from=dev /usr/src/app/node_modules/exiftool-vendored.pl ./node_modules/exiftool-vendored.pl

# web build
FROM node:22.11.0-alpine3.20@sha256:f265794478aa0b1a23d85a492c8311ed795bc527c3fe7e43453b3c872dcd71a3 AS web

WORKDIR /usr/src/open-api/typescript-sdk
COPY open-api/typescript-sdk/package*.json open-api/typescript-sdk/tsconfig*.json ./
RUN npm ci
COPY open-api/typescript-sdk/ ./
RUN npm run build

WORKDIR /usr/src/app
COPY web/package*.json web/svelte.config.js ./
RUN npm ci
COPY web ./
COPY i18n ../i18n
RUN npm run build


# prod build
FROM ghcr.io/immich-app/base-server-prod:20241029@sha256:669cc57091e3d2924b9e3001acb4998ed9c9585370017adcdf7beed21d249aae

WORKDIR /usr/src/app
ENV NODE_ENV=production \
  NVIDIA_DRIVER_CAPABILITIES=all \
  NVIDIA_VISIBLE_DEVICES=all
COPY --from=prod /usr/src/app/node_modules ./node_modules
COPY --from=prod /usr/src/app/dist ./dist
COPY --from=prod /usr/src/app/bin ./bin
COPY --from=web /usr/src/app/build /build/www
COPY server/resources resources
COPY server/package.json server/package-lock.json ./
COPY server/start*.sh ./
COPY "docker/scripts/get-cpus.sh" ./
RUN npm link && npm install -g @immich/cli && npm cache clean --force
COPY LICENSE /licenses/LICENSE.txt
COPY LICENSE /LICENSE
ENV PATH="${PATH}:/usr/src/app/bin"

ARG BUILD_ID
ARG BUILD_IMAGE
ARG BUILD_SOURCE_REF
ARG BUILD_SOURCE_COMMIT

ENV IMMICH_BUILD=${BUILD_ID}
ENV IMMICH_BUILD_URL=https://github.com/immich-app/immich/actions/runs/${BUILD_ID}
ENV IMMICH_BUILD_IMAGE=${BUILD_IMAGE}
ENV IMMICH_BUILD_IMAGE_URL=https://github.com/immich-app/immich/pkgs/container/immich-server
ENV IMMICH_REPOSITORY=immich-app/immich
ENV IMMICH_REPOSITORY_URL=https://github.com/immich-app/immich
ENV IMMICH_SOURCE_REF=${BUILD_SOURCE_REF}
ENV IMMICH_SOURCE_COMMIT=${BUILD_SOURCE_COMMIT}
ENV IMMICH_SOURCE_URL=https://github.com/immich-app/immich/commit/${BUILD_SOURCE_COMMIT}

VOLUME /usr/src/app/upload
EXPOSE 2283
ENTRYPOINT ["tini", "--", "/bin/bash"]
CMD ["start.sh"]

HEALTHCHECK CMD immich-healthcheck
