ARG BASE_REGISTRY=docker.io
ARG BASE_NAMESPACE=library
ARG BASE_REPO=python
ARG BASE_TAG=3.12.4-alpine3.20
ARG PYTHON_VERSION=3.12.4

FROM ${BASE_REGISTRY}/library/python:${BASE_TAG} AS python_base
RUN echo "prebuilt base"

FROM ${BASE_REGISTRY}/redhat/ubi9:${BASE_TAG} AS ubi9_base
ARG PYTHON_VERSION
RUN echo "ubi9 base"
ENV PATH /usr/local/bin:$PATH

# https://devguide.python.org/getting-started/setup-building/index.html#install-dependencies
RUN <<EOF
# Install dependencies referring to debian for comparison
# Missing: bluetooth, gdbm, readline, tk
keep=( gcc gdb )
buildonly=(
  make
  redhat-rpm-config  # Needed for rpm --eval macro expansion
  wget
  xz

  bzip2-devel
  expat-devel
  glibc-devel
  libdb-devel
  libffi-devel
  libuuid-devel
  ncurses-devel
  openssl-devel
  sqlite-devel
  xz-devel
  zlib-devel
)

dnf update -y
dnf install -y ${keep[@]} ${buildonly[@]}

# Fetch, verify, and extract source
declare -A key_id=(
  ["3.12.4"]="A821E680E5FA6305"
)
curl -O "https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tar.xz"
curl -O "https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tar.xz.asc"

export GNUPGHOME="$(mktemp -d)"
gpg --batch --recv-keys "${key_id[$PYTHON_VERSION]}"
gpg --batch --verify Python-${PYTHON_VERSION}.tar.xz.asc Python-${PYTHON_VERSION}.tar.xz
gpgconf --kill all
rm -rf "$GNUPGHOME" Python-${PYTHON_VERSION}.tar.xz.asc
unset GNUPGHOME

mkdir -p /usr/src/python
tar --extract --directory /usr/src/python --strip-components=1 --file Python-${PYTHON_VERSION}.tar.xz
rm Python-${PYTHON_VERSION}.tar.xz

# Configure
cd /usr/src/python
./configure \
  --build="$(uname -m)-linux-gnu" \
  --enable-loadable-sqlite-extensions \
  --enable-optimizations \
  --enable-option-checking=fatal \
  --enable-shared \
  --with-lto \
  --with-system-expat

# Make
EXTRA_CFLAGS="$(rpm --eval '%{build_cflags}' | xargs)"
LDFLAGS="$(rpm --eval '%{build_ldflags}' | xargs)"
make \
  -j "$(nproc)" \
  "EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
  "LDFLAGS=${LDFLAGS:-} -Wl,-rpath='\$\$ORIGIN/../lib'"

# Make install
make install

# enable GDB to load debugging data: https://github.com/docker-library/python/pull/701
bin="$(readlink -ve /usr/local/bin/python3)"
dir="$(dirname "$bin")"
mkdir -p "/usr/share/gdb/auto-load/$dir"
cp -vL Tools/gdb/libpython.py "/usr/share/gdb/auto-load/$bin-gdb.py"
cd
rm -rf /usr/src/python

# Shrink the install by removing optimized files and build packages
find /usr/local -depth \
  \( \
    \( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
    -o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
  \) -exec rm -rf '{}' +

ldconfig

find /usr/local -type f -executable -exec ldd {} ";" \
    | awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); printf "*%s\n", so }' \
    | grep -v '\*not' \
    | sort -u \
    | while read -r line; do dnf whatprovides -q $line | head -1 | awk '{ split($0, a, " :"); print a[1] }'; done \
    | sort -u \
    | xargs dnf mark install

dnf remove -y ${buildonly[@]}
dnf autoremove -y
dnf clean all

# Make some useful symlinks
for src in idle3 pip3 pydoc3 python3 python3-config
do
  dst="$(echo $src | tr -d 3)"
  [ -s "/usr/local/bin/$src" ]
  [ ! -e "/usr/local/bin/$dst" ]
  ln -svT "$src" "/usr/local/bin/$dst"
done

# Install pip with ensurepip
python3 -m ensurepip
EOF

FROM ${BASE_REPO}_base AS final
RUN python3 --version
CMD ["python3"]
