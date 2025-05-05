FROM debian:bullseye-slim

# mkdir the man/man1 directory due to Debian bug #863199
RUN apt-get update && \
    mkdir -p /usr/share/man/man1 && \
    apt-get install --yes --no-install-recommends \
      autoconf \
      automake \
      bzip2 \
      cmake \
      curl \
      g++ \
      gcc \
      git \
      libc6-dev \
      libgmp-dev \
      libmpfr-dev \
      libsqlite3-dev \
      sqlite3 \
      make \
      opam \
      openjdk-11-jdk-headless \
      patch \
      patchelf \
      pkg-config \
      python3 \
      python3-distutils \
      unzip \
      xz-utils \
      zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

# Disable sandboxing
# Without this opam fails to compile OCaml for some reason. We don't need sandboxing inside a Docker container anyway.
RUN opam init --reinit --bare --disable-sandboxing --yes --auto-setup

# # Download the latest Infer from git
# RUN cd / && \
#     git clone --depth 1 https://github.com/facebook/infer/
COPY . /infer

# Build opam deps first, then infer. This way if any step fails we
# don't lose the significant amount of work done in the previous
# steps.
RUN cd /infer && ./build-infer.sh java --only-setup-opam
# RUN cd /infer && ./build-infer.sh java
RUN opam install ocaml-lsp-server

RUN rm -rf /infer
