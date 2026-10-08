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

FROM python:3.12-slim

ENV DEBIAN_FRONTEND noninteractive
ENV LANG C.UTF-8

# Speculos build dependencies
RUN apt-get update && \
  apt-get install -qy cmake curl gcc-arm-linux-gnueabihf \
    git libc6-dev-armhf-cross libvncserver-dev python3-pip \
    qemu-user-static wget \
    gdb-multiarch binutils-arm-none-eabi && \
    apt-get clean && rm -rf /var/lib/apt/lists/

# Speculos runtime deps are installed from its pyproject.toml below
RUN pip3 install ledgerblue flake8 pbkdf2 pytest

# Create zondax user
RUN adduser --disabled-password --gecos "" -u 1000 zondax
RUN echo "zondax ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
WORKDIR /home/zondax
USER zondax

# Speculos
RUN git clone https://github.com/LedgerHQ/speculos --depth 1

RUN cd speculos && cmake -Bbuild -H. -DWITH_VNC=1
RUN make -C speculos/build

# Zemu runs speculos.py from the source tree, so only install the deps declared in pyproject.toml
# (`pip install .` would rebuild the launcher a second time)
RUN python -c "import tomllib; print('\\n'.join(tomllib.load(open('speculos/pyproject.toml', 'rb'))['project']['dependencies']))" > /tmp/speculos-requirements.txt && \
  pip install --user -r /tmp/speculos-requirements.txt && \
  rm /tmp/speculos-requirements.txt

# Patch proxy to connect to all interfaces (fail the build if upstream changes the line again)
RUN sed -i -E "s/^HOST = ['\"]127\.0\.0\.1['\"]/HOST = \"0.0.0.0\"/" speculos/tools/ledger-live-http-proxy.py && \
  grep -q '^HOST = "0.0.0.0"' speculos/tools/ledger-live-http-proxy.py

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
