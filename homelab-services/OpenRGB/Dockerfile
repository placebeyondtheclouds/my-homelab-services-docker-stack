FROM ubuntu:24.04

# Build-time environment variables
ENV DEBIAN_FRONTEND=noninteractive

RUN <<EOF
apt-get update &&
    apt-get install -y --no-install-recommends dirmngr ca-certificates apt-transport-https
EOF

# set up mirrors
RUN <<EOF
tee /etc/apt/sources.list.d/ubuntu.sources <<-'EOF2'
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: http://security.ubuntu.com/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF2
EOF

# set timezone
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# Install dependencies
RUN <<EOF
apt-get update &&
    apt-get install -y --no-install-recommends git build-essential cmake curl software-properties-common &&
    apt-get install -y git build-essential qtcreator qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools libusb-1.0-0-dev libhidapi-dev pkgconf libmbedtls-dev qttools5-dev-tools &&
    apt-get clean &&
    rm -rf /var/lib/apt/lists/*
EOF

# Clone hashcat
WORKDIR /root

RUN git clone https://github.com/CalcProgrammer1/OpenRGB.git

WORKDIR /root/OpenRGB

RUN mkdir build

WORKDIR /root/OpenRGB/build

RUN qmake ../OpenRGB.pro

RUN make -j $(nproc)

RUN make install

# RUN udevadm control --reload-rules && sudo udevadm trigger

# CMD ["openrgb", "--server"]

CMD ["tail", "-f", "/dev/null"]
