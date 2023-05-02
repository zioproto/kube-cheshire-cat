#Build me:
#docker buildx build --platform=linux/amd64,linux/arm64 -t zioproto/cheshire-cat-core:latest --push . --no-cache

FROM curlimages/curl as build
RUN mkdir -p /tmp/cheshire-cat
RUN curl -sL https://github.com/pieroit/cheshire-cat/archive/pull/137/head.tar.gz | tar xz -C /tmp/cheshire-cat --strip-components=1

FROM python:3.10.11-slim-bullseye as runner
COPY --from=build /tmp/cheshire-cat/core /core
WORKDIR /core
RUN echo uvicorn[standard]==0.20.0 >> requirements.txt && echo gunicorn==20.1.0 >> requirements.txt && pip install -U pip && pip install --no-cache-dir -r requirements.txt && adduser --disabled-password --gecos '' appuser
USER appuser
EXPOSE 1865
ENV PYTHONUNBUFFERED=1
CMD ["uvicorn", "cat.main:cheshire_cat_api", "--log-config", "./logger.yml"]
