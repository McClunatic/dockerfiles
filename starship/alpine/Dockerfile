ARG BASE_REGISTRY=docker.io
ARG BASE_IMAGE=library/alpine
ARG BASE_TAG=3.20

FROM ${BASE_REGISTRY}/${BASE_IMAGE}:${BASE_TAG}
ARG VERSION=1.19.0

RUN <<EOF
set -eux
wget "https://starship.rs/install.sh"
chmod +x install.sh
./install.sh --yes --version v"${VERSION}" \
  --platform unknown-linux-musl --arch "$(apk --print-arch)"
EOF
