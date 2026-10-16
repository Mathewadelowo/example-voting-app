FROM node:24-bookworm-slim@sha256:a5f4366e834695d5f363b920c104953a535d3a9864d1bdf74566a0ebe01daee5 AS base

RUN apt-get update -qq && apt-get install -qy \ 
    ca-certificates \
    bzip2 \
    curl \
    libfontconfig \
    --no-install-recommends
RUN yarn global add phantomjs-prebuilt
ADD . /app
WORKDIR /app
CMD ["/app/tests.sh"]
