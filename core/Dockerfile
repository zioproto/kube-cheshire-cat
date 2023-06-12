#Build me:
#docker buildx build --platform=linux/amd64,linux/arm64 -t zioproto/cheshire-cat-core:latest --push . --no-cache

FROM python:3.10.11-slim-bullseye
RUN mkdir -p /app && mkdir -p /admin
# install curl and xz-utils for nodejs
# libmagic-mgc libmagic1 are required for https://pypi.org/project/unstructured/ to work correctly (used in langchain)

WORKDIR /admin

RUN apt-get -y update && apt-get install -y curl xz-utils libmagic-mgc libmagic1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* &&\
    curl -sL https://github.com/zioproto/cheshire-cat/archive/HuggingFaceTextGenInference.tar.gz | tar xz -C /app --strip-components=1 &&\
    curl -sL https://github.com/cheshire-cat-ai/admin-vue/archive/main.tar.gz | tar xz -C /admin --strip-components=1 &&\
    NODEARCH= && dpkgArch="$(dpkg --print-architecture)" &&\
    case "${dpkgArch##*-}" in \
      amd64) NODEARCH='x64';; \
      ppc64el) NODEARCH='ppc64le';; \
      s390x) NODEARCH='s390x';; \
      arm64) NODEARCH='arm64';; \
      armhf) NODEARCH='armv7l';; \
      i386) NODEARCH='x86';; \
      *) echo "unsupported architecture"; exit 1 ;;\
    esac &&\
    pip install -U pip && \
    pip install --no-cache-dir /app/core &&\
    python3 /app/core/install_plugin_dependencies.py &&\
### ADMIN (static build) ###
    curl -sL https://github.com/pieroit/cheshire-cat-admin/archive/main.tar.gz | tar xz -C /admin --strip-components=1 &&\
    curl -fsSLO --compressed "https://nodejs.org/dist/v18.16.0/node-v18.16.0-linux-${NODEARCH}.tar.xz" &&\
    tar -xJf node-v18.16.0-linux-${NODEARCH}.tar.xz -C /usr/local --strip-components=1 --no-same-owner &&\
    rm node-v18.16.0-linux-${NODEARCH}.tar.xz &&\
    ln -s /usr/local/bin/node /usr/local/bin/nodejs &&\
    npm install &&\
    npm run build

WORKDIR /app/core
USER appuser
EXPOSE 1865
ENV PYTHONUNBUFFERED=1
CMD ["uvicorn", "cat.main:cheshire_cat_api"]
