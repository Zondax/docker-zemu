#*******************************************************************************
#    (c) 2020-2022 Zondax GmbH
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
#Download base ubuntu image
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND noninteractive
ENV LANG C.UTF-8

RUN apt-get update && \
    apt-get -y install build-essential git wget sudo udev zip \
                       curl cmake software-properties-common \
                       python3 python3-pip python-is-python3 \
                       libudev-dev libusb-1.0-0-dev

# Speculos dependencies
RUN apt-get update && \
    apt-get -y install qemu-user-static python3-construct \
                       python3-mnemonic python3-pyelftools gcc-arm-linux-gnueabihf \
                       libc6-dev-armhf-cross gdb-multiarch \
                       tesseract-ocr libtesseract-dev libvncserver-dev

RUN pip3 install construct flake8 flask flask_restful jsonschema mnemonic pycrypto pyelftools pbkdf2 pytest Pillow requests pytesseract

# Create zondax user
RUN adduser --disabled-password --gecos "" -u 1000 zondax
RUN echo "zondax ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
WORKDIR /home/zondax
USER zondax

# Speculos
RUN git clone https://github.com/LedgerHQ/speculos --depth 1
RUN cd /home/zondax/speculos

# Create SHA256SUMS, download dependencies and verify their integrity
RUN \
  echo 892a0875b9872acd04a9fde79b1f943075d5ea162415de3047c327df33fbaee5 openssl-1.1.1k.tar.gz >> SHA256SUMS && \
  echo f0ccd8242d55e2fd74b16ba518359151f6f8383ff8aef4976e48393f77bba8b6 cmocka-1.1.5.tar.xz >> SHA256SUMS && \
  wget --quiet https://www.openssl.org/source/openssl-1.1.1k.tar.gz && \
  wget --quiet https://cmocka.org/files/1.1/cmocka-1.1.5.tar.xz && \
  sha256sum --check SHA256SUMS && \
  rm SHA256SUMS

# Build dependencies and install them in ./install
RUN mkdir install && \
  tar xf openssl-1.1.1k.tar.gz && \
  cd openssl-1.1.1k && \
  ./Configure --cross-compile-prefix=arm-linux-gnueabihf- no-asm no-threads no-shared no-sock linux-armv4 --prefix=/home/zondax/speculos/install && \
  make -j CFLAGS=-mthumb && \
  make install_sw && \
  cd .. && \
  rm -r openssl-1.1.1k/ openssl-1.1.1k.tar.gz

RUN mkdir cmocka && \
  tar xf cmocka-1.1.5.tar.xz && \
  cd cmocka && \
  cmake ../cmocka-1.1.5 -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc -DCMAKE_C_FLAGS=-mthumb -DWITH_STATIC_LIB=true -DCMAKE_INSTALL_PREFIX=/home/zondax/speculos/install && \
  make install && \
  cd .. && \
  rm -r cmocka/ cmocka-1.1.5/ cmocka-1.1.5.tar.xz

RUN cd speculos && cmake -Bbuild -H. -DPRECOMPILED_DEPENDENCIES_DIR=./install -DWITH_VNC=1
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
