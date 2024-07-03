ARG BASE_REGISTRY=docker.io
ARG BASE_NAMESPACE=library
ARG BASE_REPO=alpine
ARG BASE_TAG=3.20.1
ARG PYTHON_VERSION=3.12.4

FROM ${BASE_REGISTRY}/library/alpine:${BASE_TAG} AS alpine_base
RUN echo "alpine base"

# https://github.com/docker-library/python/blob/master/Dockerfile-linux.template
RUN <<EOF
apk add --no-cache git
apk add --no-cache --virtual .build-deps \
    bluez-dev \
    bzip2-dev \
    dpkg-dev dpkg \
    expat-dev \
    findutils \
    gcc \
    gdbm-dev \
    libc-dev \
    libffi-dev \
    libnsl-dev \
    libtirpc-dev \
    linux-headers \
    make \
    ncurses-dev \
    openssl-dev \
    pax-utils \
    readline-dev \
    sqlite-dev \
    tcl-dev \
    tk \
    tk-dev \
    util-linux-dev \
    xz-dev \
    zlib-dev
echo "export EXTRA_CFLAGS='-DTHREAD_STACK_SIZE=0x100000'" >> /env
EOF

FROM ${BASE_REGISTRY}/library/debian:${BASE_TAG} AS debian_base
RUN echo "debian base"

# https://devguide.python.org/getting-started/setup-building/index.html#install-dependencies
RUN <<EOF
sed -i -e 's/Types: deb$/Types: deb deb-src/g' \
       -e 's/# http/http/g' /etc/apt/sources.list.d/debian.sources
apt-get -y update
apt-get -y install git
apt-get -y build-dep python3
apt-get -y install pkg-config
apt-get -y install build-essential gdb lcov pkg-config \
      libbz2-dev libffi-dev libgdbm-dev libgdbm-compat-dev liblzma-dev \
      libncurses5-dev libreadline6-dev libsqlite3-dev libssl-dev \
      lzma lzma-dev tk-dev uuid-dev zlib1g-dev libmpdec-dev
apt-get -y install libexpat1-dev
echo "export EXTRA_CFLAGS=\"$(dpkg-buildflags --get CFLAGS)\"" >> /env
echo "export LDFLAGS=\"$(dpkg-buildflags --get LDFLAGS)\"" >> /env
EOF

FROM ${BASE_REGISTRY}/redhat/ubi9:${BASE_TAG} AS ubi9_base
RUN echo "ubi9 base"
# https://devguide.python.org/getting-started/setup-building/index.html#install-dependencies
RUN <<EOF
dnf update -y
dnf install -y git
dnf install -y dnf-plugins-core  # install this to use 'dnf builddep'
dnf builddep -y python3
echo "export EXTRA_CFLAGS=\"$(rpm --eval '%{build_cflags}' | xargs)\"" >> /env
echo "export LDFLAGS=\"$(rpm --eval '%{build_ldflags}' | xargs)\"" >> /env
EOF

FROM ${BASE_REPO}_base AS python_build
ARG PYTHON_VERSION
RUN echo "python build"

RUN <<EOF
. /env && rm /env
git clone --depth 1 --branch "v${PYTHON_VERSION}" https://github.com/python/cpython
cd cpython
./configure --enable-optimizations --with-lto --with-system-expat \
    --enable-shared --enable-loadable-sqlite-extensions
make -j $(nproc) "EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" "LDFLAGS=${LDFLAGS:-}"

# https://github.com/docker-library/python/issues/784
# prevent accidental usage of a system installed libpython of the same version
rm python
make -j $(nproc) \
    "EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
    "LDFLAGS=${LDFLAGS:--Wl},-rpath='\$\$ORIGIN/../lib'" \
    "PROFILE_TASK=${PROFILE_TASK:-}" \
    python
make install
python3 --version
EOF
