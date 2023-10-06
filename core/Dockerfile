#Build me:
#docker buildx build --platform=linux/amd64,linux/arm64 -t zioproto/cheshire-cat-core:latest --push . --no-cache

FROM python:3.10.11-slim-bullseye
RUN mkdir -p /app && mkdir -p /admin
# libmagic-mgc libmagic1 are required for https://pypi.org/project/unstructured/ to work correctly (used in langchain)

WORKDIR /admin

RUN apt-get -y update && apt-get install -y curl fastjar libmagic-mgc libmagic1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* &&\
    curl -sL https://github.com/cheshire-cat-ai/core/archive/refs/tags/1.3.1.tar.gz | tar xz -C /app --strip-components=1 &&\
    curl -sL https://github.com/cheshire-cat-ai/admin-vue/releases/download/Admin/release.zip | jar -xv &&\
    pip install -U pip && \
    pip install --no-cache-dir /app/core &&\
    python3 -c "import nltk; nltk.download('punkt')" &&\
    python3 /app/core/install_plugin_dependencies.py

WORKDIR /app/core
USER appuser
EXPOSE 1865
ENV PYTHONUNBUFFERED=1
CMD ["uvicorn", "cat.main:cheshire_cat_api"]
