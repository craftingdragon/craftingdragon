# syntax=docker/dockerfile:1.7

ARG NODE_VERSION=24-alpine3.23

FROM node:${NODE_VERSION} AS build

RUN apk upgrade --no-cache
RUN apk add --no-cache git

WORKDIR /src

ARG GITHUB_README_STATS_REPO=https://github.com/anuraghazra/github-readme-stats.git
ARG GITHUB_README_STATS_REF=master

RUN git clone --filter=blob:none "${GITHUB_README_STATS_REPO}" . \
  && git checkout "${GITHUB_README_STATS_REF}"

# The upstream Express server is documented for non-Vercel hosting, but express
# is kept as a devDependency upstream. Promote it before installing production deps.
RUN node -e "const fs=require('fs');const pkg=JSON.parse(fs.readFileSync('package.json','utf8'));pkg.dependencies={...(pkg.dependencies||{})};if(pkg.devDependencies?.express){pkg.dependencies.express=pkg.devDependencies.express;delete pkg.devDependencies.express;}pkg.scripts={...(pkg.scripts||{}),start:'node express.js'};fs.writeFileSync('package.json',JSON.stringify(pkg,null,2)+'\n');"

RUN npm update --package-lock-only --ignore-scripts \
  && npm ci --omit=dev --ignore-scripts \
  && npm cache clean --force

# Keep development metadata and the upstream Git database out of the runtime.
RUN rm -rf .git package-lock.json tests node_modules/.package-lock.json
RUN node -e "const fs=require('fs');const p=JSON.parse(fs.readFileSync('package.json'));delete p.devDependencies;fs.writeFileSync('package.json',JSON.stringify(p));"

FROM node:${NODE_VERSION} AS runtime

ENV NODE_ENV=production \
  PORT=9000 \
  CACHE_SECONDS=86400

WORKDIR /app

RUN apk upgrade --no-cache
# npm and Corepack are build tools, not required by the running server.
RUN rm -rf /usr/local/lib/node_modules /opt/yarn* /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack /usr/local/bin/yarn /usr/local/bin/yarnpkg

RUN addgroup -g 10001 -S readmestats \
  && adduser -u 10001 -S -G readmestats readmestats

COPY --from=build --chown=readmestats:readmestats /src /app

USER 10001:10001

EXPOSE 9000

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD node -e "const http=require('http');const user=(process.env.WHITELIST||'octocat').split(',')[0];http.get('http://127.0.0.1:'+(process.env.PORT||9000)+'/api?username='+encodeURIComponent(user),res=>process.exit(res.statusCode<500?0:1)).on('error',()=>process.exit(1));"

CMD ["node", "express.js"]
