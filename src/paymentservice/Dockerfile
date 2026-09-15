FROM --platform=$BUILDPLATFORM node:22.23.2-alpine3.24 AS builder

# Native dependencies required by packages such as @google-cloud/profiler
RUN apk add --no-cache \
    python3 \
    make \
    g++

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm ci --omit=dev --ignore-scripts

COPY . .

FROM node:22.23.2-alpine3.24

WORKDIR /usr/src/app

# Update Alpine packages to the latest security-fixed versions
RUN apk update \
    && apk upgrade \
    && apk add --no-cache libstdc++ \
    && rm -rf /var/cache/apk/*

COPY --from=builder /usr/src/app/node_modules ./node_modules
COPY . .

# npm/npx are not required at runtime
RUN rm -rf /usr/local/lib/node_modules/npm \
    && rm -f /usr/local/bin/npm \
    && rm -f /usr/local/bin/npx

EXPOSE 50051

ENTRYPOINT ["node", "index.js"]
