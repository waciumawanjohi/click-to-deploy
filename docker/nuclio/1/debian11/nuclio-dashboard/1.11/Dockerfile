FROM marketplace.gcr.io/google/c2d-debian11 as build

SHELL ["/bin/bash", "-c" ]
WORKDIR /app

#copy Nuclio sources
ARG NUCLIO_VERSION=1.11.12
ARG NUCLIO_DIST=https://github.com/nuclio/nuclio/archive/refs/tags/${NUCLIO_VERSION}.tar.gz
ADD $NUCLIO_DIST /tmp/nuclio.tar.gz
RUN (tar -xf /tmp/nuclio.tar.gz -C /app/ && rm /tmp/nuclio.tar.gz && mv /app/nuclio-${NUCLIO_VERSION}/* /app/) || true

#install docker cli and node
ARG DOCKER_CLI_ARCH=x86_64
ARG DOCKER_CLI_VERSION=20.10.19
ENV DOCKER_CLI_DOWNLOAD_URL="https://download.docker.com/linux/static/stable/$DOCKER_CLI_ARCH/docker-$DOCKER_CLI_VERSION.tgz"
RUN apt-get update && apt-get install ca-certificates curl -y \
    && mkdir -p /tmp/download \
    && curl -L $DOCKER_CLI_DOWNLOAD_URL | tar -xz -C /tmp/download \
    && mv /tmp/download/docker/docker /usr/local/bin/ \
    && rm -rf /tmp/download \
    && rm -rf /var/lib/apt/lists/*
RUN curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash \
    && source ~/.bashrc \
    && nvm install 10

#install golang
ARG GOLANG_VERSION=1.18.7
ADD https://golang.org/dl/go${GOLANG_VERSION}.linux-amd64.tar.gz /tmp/go.tar.gz
RUN (tar -xf /tmp/go.tar.gz -C /usr/local/ && rm /tmp/go.tar.gz) || true
ENV PATH="/usr/local/go/bin:${PATH}"

#build dashboard
RUN source ~/.bashrc \
    && npm install -g gulp \
    && cd /app/pkg/dashboard/ui \
    && rm -rf ./dist ./node_modules ./resources/*/node_modules \
    && npm install --production \
    && gulp build --production

#build dashboard binary
ARG NUCLIO_GO_LINK_FLAGS_INJECT_VERSION
ARG GOOS=linux
ARG GOARCH=amd64
RUN cd /app/cmd/dashboard/ && GOOS=$GOOS GOARCH=$GOARCH CGO_ENABLED=0 go build \
    -a \
    -installsuffix cgo \
    -ldflags="${NUCLIO_GO_LINK_FLAGS_INJECT_VERSION}" \
    -o dashboard main.go

FROM marketplace.gcr.io/google/nginx1

ENV C2D_RELEASE 1.11.12

# fetch nuclio functions templates during build time to ease installation on offline environments
# fix security issues
ARG NUCLIO_TEMPLATES_ZIP_GITHUB_URL="https://github.com/nuclio/nuclio-templates/archive/master.zip"
RUN apt-get update \
    && apt-get install -y --no-install-recommends git ca-certificates parallel \
       dpkg \
       gpgv \
       libc6 \
       libc-bin \
       libgnutls30 \
       libtirpc3 \
       libtirpc-common \
       libssl1.1 \
       openssl \
       zlib1g \
       expat \
       libexpat1 \
       libtasn1-6 \
       libksba8 \
    && apt-get autoremove -yqq --purge \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && wget -O /tmp/templates.zip $NUCLIO_TEMPLATES_ZIP_GITHUB_URL

# copy dashboard static from build assets stage. /etc/nuclio/dashboard/static will contain index.html and assets
COPY --from=build /app/pkg/dashboard/ui/dist /etc/nginx/static/

# copy dashboard binary from build binary stage
COPY --from=build /app/cmd/dashboard/dashboard /usr/local/bin

# copy a lightweight http client
COPY --from=gcr.io/iguazio/uhttpc:0.0.1-amd64 /home/nuclio/bin/uhttpc /usr/local/bin/uhttpc

# copy docker client
COPY --from=build /usr/local/bin/docker /usr/local/bin/docker

#copy LICENSE
COPY --from=build /app/LICENSE /usr/share/nuclio/LICENSE

# copy runners
COPY --from=build /app/cmd/dashboard/docker/runners /runners
COPY --from=build /app/cmd/dashboard/docker/runner.sh /runner.sh

# copy nginx config
COPY --from=build /app/cmd/dashboard/docker/default.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/cmd/dashboard/docker/nginx.conf /etc/nginx/nginx.conf

HEALTHCHECK --interval=1s --timeout=3s CMD /usr/local/bin/uhttpc --url http://127.0.0.1:8082/ready || exit 1

ENTRYPOINT []
CMD ["sh",  "-c", "./runner.sh"]

