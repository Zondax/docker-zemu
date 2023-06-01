#*******************************************************************************
#    (c) 2020-2023 Zondax GmbH
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
#*******************************************************************************

### Ledger speculos-builder Dockerfile
FROM docker.io/library/python:3.9-slim
ENV LANG C.UTF-8

RUN export DEBIAN_FRONTEND=noninteractive && \
  apt-get update && \
  apt-get install -qy \
    cmake \
    curl \
    gcc-arm-linux-gnueabihf \
    git \
    libc6-dev-armhf-cross \
    libvncserver-dev \
    python3-pip \
    qemu-user-static \
    tesseract-ocr \
    libtesseract-dev \
    wget && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/

# There are issues with PYTHONHOME if using distro packages, use pip instead.
RUN pip3 install construct flake8 flask flask_restful jsonschema mnemonic pycrypto pyelftools pbkdf2 pytest Pillow requests pytesseract

# Create SHA256SUMS, download dependencies and verify their integrity
RUN \
  echo 892a0875b9872acd04a9fde79b1f943075d5ea162415de3047c327df33fbaee5 openssl-1.1.1k.tar.gz >> SHA256SUMS && \
  echo f0ccd8242d55e2fd74b16ba518359151f6f8383ff8aef4976e48393f77bba8b6 cmocka-1.1.5.tar.xz >> SHA256SUMS && \
  echo 70127766f8031cde3df4224d88f7b33dec6c33fc7ac6b8e4308d4f7d0bdffd7b d0bc304a132df43856d8302e15dabee97d3d8a95.tar.gz && \
  wget --quiet https://www.openssl.org/source/openssl-1.1.1k.tar.gz && \
  wget --quiet https://cmocka.org/files/1.1/cmocka-1.1.5.tar.xz && \
  wget --quiet https://github.com/supranational/blst/archive/d0bc304a132df43856d8302e15dabee97d3d8a95.tar.gz && \
  sha256sum --check SHA256SUMS && \
  rm SHA256SUMS

# Build dependencies and install them in /install
RUN mkdir install && \
  tar xf openssl-1.1.1k.tar.gz && \
  cd openssl-1.1.1k && \
  ./Configure --cross-compile-prefix=arm-linux-gnueabihf- no-asm no-threads no-shared no-sock linux-armv4 --prefix=/install && \
  make -j CFLAGS=-mthumb && \
  make install_sw && \
  cd .. && \
  rm -r openssl-1.1.1k/ openssl-1.1.1k.tar.gz

RUN mkdir cmocka && \
  tar xf cmocka-1.1.5.tar.xz && \
  cd cmocka && \
  cmake ../cmocka-1.1.5 -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc -DCMAKE_C_FLAGS=-mthumb -DWITH_STATIC_LIB=true -DCMAKE_INSTALL_PREFIX=/install && \
  make install && \
  cd .. && \
  rm -r cmocka/ cmocka-1.1.5/ cmocka-1.1.5.tar.xz

RUN tar xf d0bc304a132df43856d8302e15dabee97d3d8a95.tar.gz && \
  cd blst-d0bc304a132df43856d8302e15dabee97d3d8a95 && \
  sh build.sh CC=arm-linux-gnueabihf-gcc && \
  cp libblst.a ../install/lib/ && \
  cp bindings/blst.h ../install/include/ && \
  cp bindings/blst_aux.h ../install/include/ && \
  cd .. && \
  rm -r blst-d0bc304a132df43856d8302e15dabee97d3d8a95/ d0bc304a132df43856d8302e15dabee97d3d8a95.tar.gz

# Create zondax user
RUN adduser --disabled-password --gecos "" -u 1000 zondax
RUN echo "zondax ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
WORKDIR /home/zondax
USER zondax

# Speculos
RUN git clone https://github.com/LedgerHQ/speculos --depth 1

RUN cd speculos && cmake -Bbuild -H. -DPRECOMPILED_DEPENDENCIES_DIR=/install -DWITH_VNC=1
RUN make -C speculos/build

# Patch proxy to connect to all interfaces
RUN sed -i "s/HOST = '127.0.0.1'/HOST = '0.0.0.0'/g" speculos/tools/ledger-live-http-proxy.py

# Open TCP ports
# gdb
EXPOSE 1234/tcp
EXPOSE 1234/udp
# device keyboard
EXPOSE 1235/tcp
EXPOSE 1235/udp
# APDU RAW
EXPOSE 9999/tcp
EXPOSE 9999/udp
# HTTP APDU PROXY
EXPOSE 9998/tcp
EXPOSE 9998/udp
# RFB
EXPOSE 8001/tcp
EXPOSE 8001/udp
# API Speculos
EXPOSE 5000/tcp
EXPOSE 5000/udp

# ENV
ADD entrypoint.sh /home/zondax/entrypoint.sh

# START SCRIPT
ENTRYPOINT ["/home/zondax/entrypoint.sh"]
