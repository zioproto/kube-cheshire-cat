#Build me:
#docker buildx build --platform=linux/amd64,linux/arm64 -t zioproto/cheshire-cat-admin:latest --push . --no-cache

FROM curlimages/curl as build
RUN mkdir -p /tmp/cheshire-cat
RUN curl -sL https://github.com/zioproto/cheshire-cat/archive/kubernetes.tar.gz | tar xz -C /tmp/cheshire-cat --strip-components=1

FROM node:18.16.0-bullseye-slim as runner
COPY --from=build /tmp/cheshire-cat/admin /app
COPY . /app
RUN adduser --disabled-password --gecos '' appuser
RUN chown -R appuser:appuser /app
RUN chmod 755 /app
USER appuser
WORKDIR /app

#TODO:
#ENV NODE_ENV=production
#RUN npm ci --only=production ; npm run build # Review package.json for promotion of devDependencies

RUN npm install

# Expose port 3000 and build + start application
EXPOSE 3000


#CMD ["/bin/bash", "-c", "npm run start"] # Review package.json for promotion of devDependencies
CMD ["/bin/bash", "-c", "npm run dev"]

