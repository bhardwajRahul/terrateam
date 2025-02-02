# Define an argument for the base image
ARG BASE_IMAGE

# Optional base setup stage if BASE_IMAGE is not provided
FROM alpine:3.18 as build-base

# Install base packages
RUN apk add --no-cache \
    bash \
    bsd-compat-headers \
    ca-certificates \
    clang \
    cmake \
    coreutils \
    curl \
    curl-dev \
    git \
    gmp-dev \
    libffi-dev \
    libretls \
    libretls-dev \
    m4 \
    make \
    mercurial \
    musl-dev \
    npm \
    opam \
    openssh \
    perl \
    py3-yaml \
    python3 \
    sqlite-dev \
    yj \
    zlib-dev

# Initialize opam
RUN opam init -ay --disable-sandboxing

# Set up OCaml compiler
RUN eval $(opam env) && \
    opam switch create -y 5.1.1

# Build libkqueue
COPY ./vendor/libkqueue /mono/vendor/libkqueue
WORKDIR /mono/vendor/libkqueue
RUN cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=lib $PWD && \
    make && \
    make install

# Copy source files
COPY ./ /mono
WORKDIR /

# Set up opam repositories and dependencies
RUN eval $(opam env) && \
    opam repository set-url default "https://github.com/jjm-enterprises/opam-repository.git#terrateam" && \
    opam repository add opam-acsl mono/opam && \
    opam pin add -y containers 3.12 && \
    opam install -y hll pds && \
    mkdir -p opam-mono/compilers opam-mono/packages && \
    echo 'opam-version: "2.0"' > opam-mono/repo && \
    opam repository add opam-mono opam-mono && \
    cd /mono/code && \
    hll generate \
      -n monorepo \
      --opam-dir ../../opam-mono \
      --tag 1.0 \
      --test-deps-as-regular-deps \
      --url-override file://$PWD && \
    opam update opam-mono && \
    opam info monorepo && \
    opam install -j4 -y --deps-only --no-depexts monorepo

# Use provided BASE_IMAGE if specified, otherwise use the build-base stage
FROM ${BASE_IMAGE:-build-base} as build-setup

# Copy source files
COPY ./ /mono

# Build target for code-indexer
FROM build-setup as code-indexer-build
WORKDIR /mono/code
RUN ulimit -s 524288 && ulimit -a && eval $(opam env) && \
    (time make -j4 release_terrat_code_indexer || time make -j1 release_terrat_code_indexer)
RUN eval $(opam env) && \
    time env OCAMLRUNPARAM=b make -j4 -k test && \
    find . -name '*.tap' -exec cat '{}' \;

# Build target for terrat-ee
FROM build-setup as terrat-ee-build
ARG TERRAT_UI_ANALYTICS=false
WORKDIR /mono/code
RUN ulimit -s 524288 && ulimit -a && eval $(opam env) && \
    (time env TERRAT_UI_ANALYTICS=$TERRAT_UI_ANALYTICS make -j4 release_terrat_ee release_terrat_ui \
     || time env TERRAT_UI_ANALYTICS=$TERRAT_UI_ANALYTICS make -j1 release_terrat_ee release_terrat_ui)
RUN eval $(opam env) && \
    time env OCAMLRUNPARAM=b make -j4 -k test && \
    find . -name '*.tap' -exec cat '{}' \;

# Build target for terrat-oss
FROM build-setup as terrat-oss-build
WORKDIR /mono/code
RUN ulimit -s 524288 && ulimit -a && eval $(opam env) && \
    (time make -j4 release_terrat_oss || time make -j1 release_terrat_oss)
RUN eval $(opam env) && \
    time env OCAMLRUNPARAM=b make -j4 -k test && \
    find . -name '*.tap' -exec cat '{}' \;

# Final code-indexer image
FROM alpine:3.18 as code-indexer
COPY --from=code-indexer-build /mono/code/build/release/terrat_code_indexer/terrat_code_indexer.native /usr/local/bin/terrat_code_indexer
WORKDIR /mnt
ENTRYPOINT ["/usr/local/bin/terrat_code_indexer"]

# Final terrat-ee image
FROM alpine:3.18 as terrat-ee
RUN apk add --no-cache \
    bash \
    curl \
    gettext \
    gmp \
    jq \
    libffi \
    libretls \
    nginx \
    py3-cryptography \
    py3-jwt \
    py3-requests \
    py3-yaml \
    python3 \
    runit \
    yj
RUN mkdir -p /usr/local/bin /etc/service/terrat /etc/service/nginx /usr/local/share/terrat/ui/assets
COPY docker/terrat/service/terrat /etc/service/terrat/run
COPY docker/terrat/service/nginx /etc/service/nginx/run
COPY docker/terrat/nginx.conf.template /etc/nginx/nginx.conf.template
COPY --from=terrat-ee-build /usr/lib/libkqueue.so /usr/lib/libkqueue.so
COPY --from=terrat-ee-build /mono/code/build/release/terrat_ee/terrat_ee.native /usr/local/bin/terrat
COPY --from=terrat-ee-build /mono/code/build/release/terrat_ui_files/assets /usr/local/share/terrat/ui/assets/
EXPOSE 8080
CMD ["runsvdir", "/etc/service"]

# Final terrat-oss image
FROM alpine:3.18 as terrat-oss
RUN apk add --no-cache \
    bash \
    curl \
    gettext \
    gmp \
    jq \
    libffi \
    libretls \
    nginx \
    py3-cryptography \
    py3-jwt \
    py3-requests \
    py3-yaml \
    python3 \
    runit \
    yj
RUN mkdir -p /usr/local/bin /etc/service/terrat /etc/service/nginx /usr/local/share/terrat/ui/assets
COPY docker/terrat/service/terrat /etc/service/terrat/run
COPY docker/terrat/service/nginx /etc/service/nginx/run
COPY docker/terrat/nginx.conf.template /etc/nginx/nginx.conf.template
COPY --from=terrat-oss-build /usr/lib/libkqueue.so /usr/lib/libkqueue.so
COPY --from=terrat-oss-build /mono/code/build/release/terrat_oss/terrat_oss.native /usr/local/bin/terrat
EXPOSE 8080
CMD ["runsvdir", "/etc/service"]
