ARG BASE_REGISTRY=docker.io
ARG BASE_IMAGE=library/python
ARG BASE_TAG=3.12-slim-bookworm

FROM ${BASE_REGISTRY}/${BASE_IMAGE}:${BASE_TAG}
ARG STARSHIP_VERSION=1.19.0
ARG HELIX_VERSION=24.03
ARG SSH_PORT=10648
ARG USER_NAME=python

# Add developer tools and create entrypoint script
RUN <<EOF
set -eux
apt update -y
apt upgrade -y
apt install -y bash curl gcc git xz-utils openssh-client openssh-server xauth x11-apps
useradd --create-home --shell $(which bash) ${USER_NAME}
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

# Add helix editor
RUN <<EOF
set -eux
curl -LO "https://github.com/helix-editor/helix/releases/download/24.03/helix-${HELIX_VERSION}-$(uname -m)-linux.tar.xz"
mkdir /usr/local/helix
tar -xvf helix-${HELIX_VERSION}-$(uname -m)-linux.tar.xz -C /usr/local/helix --strip-components 1 
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
echo "AllowAgentForwarding yes" >> ~/.sshd/config
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
