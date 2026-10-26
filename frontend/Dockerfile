FROM node:22-bookworm-slim AS build

WORKDIR /opt/front

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile --non-interactive

COPY . .
# Vue CLI 4 uses webpack 4, which needs the legacy provider on modern Node.
RUN NODE_OPTIONS=--openssl-legacy-provider yarn build

FROM nginx:stable-alpine

COPY --from=build /opt/front/dist /usr/share/nginx/html/
COPY --from=build /opt/front/default.conf /etc/nginx/conf.d/default.conf
