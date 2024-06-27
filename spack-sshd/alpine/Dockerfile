ARG BASE_REGISTRY=docker.io
ARG BASE_IMAGE=library/python
ARG BASE_TAG=3.12-alpine3.20

FROM ${BASE_REGISTRY}/${BASE_IMAGE}:${BASE_TAG}
ARG STARSHIP_VERSION=1.19.0
ARG SSH_PORT=10648
ARG USER_NAME=spack

# Add starship for bash shell prompt
RUN <<EOF
set -eux
wget "https://starship.rs/install.sh"
chmod +x install.sh
./install.sh --yes --version v"${STARSHIP_VERSION}" \
  --platform unknown-linux-musl --arch "$(apk --print-arch)"
rm install.sh
EOF

# Add developer tools and spack prereqs and create entrypoint script
RUN <<EOF
set -eux
apk update
apk upgrade
apk add bash git helix openssh xauth xeyes
for grammar in toml python json lua cmake ini regex c bash comment cpp
do
  apk add tree-sitter-$grammar
done
apk add file gcc g++ gnupg lsb-release-minimal make patch patchelf xz zstd linux-headers
adduser -D -s /bin/bash ${USER_NAME}
touch /entrypoint.sh
chmod a+x /entrypoint.sh
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
sed -i 's/X11Forwarding no/X11Forwarding yes/g' ~/.sshd/config
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
