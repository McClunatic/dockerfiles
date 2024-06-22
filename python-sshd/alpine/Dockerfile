ARG BASE_REGISTRY=docker.io
ARG BASE_IMAGE=library/alpine
ARG BASE_TAG=3.20
ARG PYTHON_IMAGE=library/python
ARG PYTHON_TAG=3.12-alpine3.20

FROM ${BASE_REGISTRY}/${BASE_IMAGE}:${BASE_TAG}
ARG VERSION=1.19.0

RUN <<EOF
set -eux
wget "https://starship.rs/install.sh"
chmod +x install.sh
./install.sh --yes --version v"${VERSION}" \
  --platform unknown-linux-musl --arch "$(apk --print-arch)"
EOF

FROM ${BASE_REGISTRY}/${PYTHON_IMAGE}:${PYTHON_TAG}
ARG USER_NAME=python
ARG SSH_PORT=10648
COPY --from=0 /usr/local/bin/starship /usr/local/bin/starship

RUN <<EOF
set -eux
apk update
apk upgrade
apk add bash openssh-server
adduser -D -s /bin/bash ${USER_NAME}
touch /entrypoint.sh
chmod a+x /entrypoint.sh
EOF

RUN cat <<EOF >> /entrypoint.sh
#!/bin/sh

trap 'pkill sshd' SIGTERM EXIT; until /usr/sbin/sshd -Ddef ~/.sshd/config & wait \$!
do
  echo "sshd exited with exit code \$?. Restarting..." >&2
  sleep 1
done
EOF

USER ${USER_NAME}
RUN <<EOF
mkdir ~/.sshd
grep -v -e "^ *#" -e "^$" -e "Include" /etc/ssh/sshd_config > ~/.sshd/config
echo "PidFile none" >> ~/.sshd/config
echo "Port ${SSH_PORT}" >> ~/.sshd/config
for keytype in rsa ecdsa ed25519
do
  hostkeyfile=~/.sshd/ssh_host_${keytype}_key
  ssh-keygen -q -t $keytype -f $hostkeyfile
  echo "HostKey $hostkeyfile" >> ~/.sshd/config
done
EOF

EXPOSE ${SSH_PORT}
ENTRYPOINT ["/entrypoint.sh"]
