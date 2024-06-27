ARG BASE_REGISTRY=docker.io
ARG BASE_IMAGE=library/fedora
ARG BASE_TAG=40

FROM ${BASE_REGISTRY}/${BASE_IMAGE}:${BASE_TAG} AS pythonbuild
ARG PYTHON_VERSION=3.12.4

# Add python; borrow from github.com/docker-library/python
# https://devguide.python.org/getting-started/setup-building/index.html#install-dependencies
ENV GPG_KEY 7169605F62C751356D054A26A821E680E5FA6305
RUN <<EOF
set -eux
dnf update -y
dnf install -y dnf-plugins-core
dnf builddep -y python3

curl -O "https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tar.xz"
curl -O "https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tar.xz.asc"

GNUPGHOME="$(mktemp -d)"; export GNUPGHOME
gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$GPG_KEY"
gpg --batch --verify Python-${PYTHON_VERSION}.tar.xz.asc Python-${PYTHON_VERSION}.tar.xz
gpgconf --kill all
rm -rf "$GNUPGHOME" Python-${PYTHON_VERSION}.tar.xz.asc
mkdir -p /usr/src/python
tar --extract --directory /usr/src/python --strip-components=1 --file Python-${PYTHON_VERSION}.tar.xz
rm Python-${PYTHON_VERSION}.tar.xz

cd /usr/src/python
./configure \
  --build="$(uname -m)-linux-gnu" \
  --enable-loadable-sqlite-extensions \
  --enable-optimizations \
  --enable-option-checking=fatal \
  --enable-shared \
  --with-lto \
  --with-system-expat \
  --without-ensurepip

EXTRA_CFLAGS="$(rpm --eval '%{build_cflags}' | xargs)"
LDFLAGS="$(rpm --eval '%{build_ldflags}' | xargs)"
make \
  -j "$(nproc)" \
  "EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
  "LDFLAGS=${LDFLAGS:-}" \
  "PROFILE_TASK=${PROFILE_TASK:-}"

# https://github.com/docker-library/python/issues/784
# prevent accidental usage of a system installed libpython of the same version
rm python
make -j "$(nproc)" \
  "EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
  "LDFLAGS=${LDFLAGS:--Wl},-rpath='\$\$ORIGIN/../lib'" \
  "PROFILE_TASK=${PROFILE_TASK:-}" \
  python

make install

# enable GDB to load debugging data: https://github.com/docker-library/python/pull/701
bin="$(readlink -ve /usr/local/bin/python3)"
dir="$(dirname "$bin")"
mkdir -p "/usr/share/gdb/auto-load/$dir"
cp -vL Tools/gdb/libpython.py "/usr/share/gdb/auto-load/$bin-gdb.py"
cd
rm -rf /usr/src/python

find /usr/local -depth \
  \( \
    \( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
    -o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
  \) -exec rm -rf '{}' +

ldconfig
python3 --version
EOF

# Add python; borrow from github.com/docker-library/python
RUN <<EOF
set -eux
for src in idle3 pydoc3 python3 python3-config
do
  dst="$(echo "$src" | tr -d 3)"
  [ -s "/usr/local/bin/$src" ]
  [ ! -e "/usr/local/bin/$dst" ]
  ln -svT "$src" "/usr/local/bin/$dst"
done
EOF

# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION 24.0
# https://github.com/pypa/get-pip
ENV PYTHON_GET_PIP_URL https://github.com/pypa/get-pip/raw/dbf0c85f76fb6e1ab42aa672ffca6f0a675d9ee4/public/get-pip.py
ENV PYTHON_GET_PIP_SHA256 dfe9fd5c28dc98b5ac17979a953ea550cec37ae1b47a5116007395bfacff2ab9

RUN <<EOF
set -eux

curl -LO "$PYTHON_GET_PIP_URL"
echo "$PYTHON_GET_PIP_SHA256 *get-pip.py" | sha256sum -c -
export PYTHONDONTWRITEBYTECODE=1
python get-pip.py \
  --disable-pip-version-check \
  --no-cache-dir \
  --no-compile \
  "pip==$PYTHON_PIP_VERSION"
rm -f get-pip.py
pip --version
EOF

FROM ${BASE_REGISTRY}/${BASE_IMAGE}:${BASE_TAG} AS pythonsshd
ARG STARSHIP_VERSION=1.19.0
ARG SSH_PORT=10648
ARG USER_NAME=python
COPY --from=pythonbuild /usr/local /usr/local

# Add python dependencies and developer tools and create entrypoint script
RUN <<EOF
set -eux
dnf update -y
dnf install -y \
  bzip2-libs \
  expat \
  gdbm-libs \
  libb2 \
  libffi \
  libnsl2 \
  libtirpc \
  libuuid \
  libxcrypt \
  mpdecimal \
  ncurses-libs \
  openssl-libs \
  readline \
  sqlite-libs \
  tzdata \
  xz-libs \
  zlib-ng-compat
dnf install -y bash git helix openssh openssh-server xorg-x11-xauth xeyes
useradd --create-home --shell /bin/bash ${USER_NAME}
touch /entrypoint.sh
chmod a+x /entrypoint.sh
chmod a+r /etc/ssh/sshd_config
EOF

# Add starship for bash shell prompt
RUN <<EOF
set -eux
curl -O "https://starship.rs/install.sh"
chmod +x install.sh
./install.sh --yes --version v"${STARSHIP_VERSION}"
rm install.sh
EOF

# Write entrypoint script
RUN cat <<EOF >> /entrypoint.sh
#!/bin/sh

trap 'pkill sshd' SIGTERM EXIT; until /usr/sbin/sshd -Ddef ~/.sshd/config & wait \$!
do
  echo "sshd exited with exit code \$?. Restarting..." >&2
  sleep 1
done
EOF

# Create developer user for non-root sshd
USER ${USER_NAME}
RUN <<EOF
mkdir ~/.sshd
grep -v -e "^ *#" -e "^$" -e "Include" /etc/ssh/sshd_config > ~/.sshd/config
sed -i 's/UsePAM yes/UsePAM no/g' > ~/.sshd/config
echo "X11Forwarding yes" >> ~/.sshd/config
echo "PidFile none" >> ~/.sshd/config
echo "Port ${SSH_PORT}" >> ~/.sshd/config
for keytype in rsa ecdsa ed25519
do
  hostkeyfile=~/.sshd/ssh_host_${keytype}_key
  ssh-keygen -q -t $keytype -f $hostkeyfile
  echo "HostKey $hostkeyfile" >> ~/.sshd/config
done
EOF

# Expose port and set entrypoint
EXPOSE ${SSH_PORT}
ENTRYPOINT ["/entrypoint.sh"]
