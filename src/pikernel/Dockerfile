# FROM ubuntu:latest AS builder

# ARG KERNEL_REF=rpi-6.12.y
# ARG GENEVE_MODE=y
# ARG KERNEL=kernel8

# RUN apt-get update && apt-get install -y bc bison flex libssl-dev make git crossbuild-essential-arm64 debhelper-compat cpio kmod libelf-dev:native libncurses-dev dwarves dpkg-dev debhelper rsync && mkdir -p /build

# RUN cd /build && git clone --depth=1 https://github.com/raspberrypi/linux.git . \
# && git fetch --depth=1 origin ${KERNEL_REF}

# RUN mkdir -p /b/firmware/overlays


# RUN cd /build && echo 'CONFIG_LOCALVERSION="-alphabet5-6.12.y"' >> .config
# RUN  cd /build && \
#     mkdir output && \
#     make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CONFIG_ARM64_VA_BITS_48=y CONFIG_GENEVE=y bcm2711_defconfig  && \
# # pi5    make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CONFIG_ARM64_VA_BITS_48=y CONFIG_GENEVE=y bcm2712_defconfig
#     make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CONFIG_ARM64_VA_BITS_48=y CONFIG_GENEVE=y KDEB_DEST_DIR=/build/output -j$(nproc) deb-pkg


# # RUN cd /build && ls -la arch/arm64/boot && cp arch/arm64/boot/Image.gz /b/firmware/$KERNEL.img && \
# #   cp arch/arm64/boot/dts/broadcom/*.dtb /b/firmware/ && \
# #   cp arch/arm64/boot/dts/overlays/*.dtb* /b/firmware/overlays/ && \
# #   cp arch/arm64/boot/dts/overlays/README /b/firmware/overlays/

# # FROM scratch

# # COPY --from=builder /b/ /boot/
FROM python:3-bookworm as BUILDER2

WORKDIR /opt/kernel
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    build-essential gcc \
    ca-certificates kmod \
    git bc bison flex libssl-dev \
    make libc6-dev libncurses5-dev \
    crossbuild-essential-arm64 dpkg-dev \
    rsync cpio debhelper quilt dh-exec python3-dacite \
    && rm -rf /var/lib/apt/lists/*

#TODO: Copy gpg key and remove trusted from the  file
COPY raspi.list /etc/apt/sources.list.d/raspi.list

# Just to make sure it works
RUN apt-get update \
    && rm -rf /var/lib/apt/lists/*

COPY scripts ./scripts
# Make sure scripts are executable
RUN chmod ugo+x ./scripts/*.sh

FROM httpd:2 as APT

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    dpkg-dev \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /usr/local/apache2/htdocs/debs