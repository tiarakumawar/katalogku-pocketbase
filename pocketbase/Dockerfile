# Image PocketBase untuk amd64 (VM/laptop) & arm64 (STB)
FROM alpine:3.22
ARG PB_VERSION=0.40.5
ARG TARGETARCH
ARG PB_URL=https://github.com/pocketbase/pocketbase/releases/download

# Jika zip PocketBase sudah ada di folder ini, dipakai langsung (hemat kuota).
# Jika tidak ada, diunduh dari GitHub seperti biasa.
COPY . /tmp/ctx/
RUN apk add --no-cache ca-certificates wget unzip \
 && F=/tmp/ctx/pocketbase_${PB_VERSION}_linux_${TARGETARCH}.zip \
 && if [ ! -f "$F" ]; then wget -q -O "$F" "${PB_URL}/v${PB_VERSION}/pocketbase_${PB_VERSION}_linux_${TARGETARCH}.zip"; fi \
 && unzip "$F" pocketbase -d /usr/local/bin \
 && rm -rf /tmp/ctx

EXPOSE 8090
CMD ["pocketbase", "serve", "--http=0.0.0.0:8090", "--dir=/pb_data"]
