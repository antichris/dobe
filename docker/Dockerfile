FROM ubuntu:18.04

RUN set -x \
&&  apt-get update \
&&  apt-get install -y --no-install-recommends \
##  Essential OpenWrt build dependencies.
    build-essential \
    gawk \
    gcc-multilib \
    git \
    libncurses5-dev \
    libssl-dev \
    openjdk-8-jdk-headless \
    python3 \
    python \
    rsync \
    subversion \
    unzip \
    wget \
    zlib1g-dev \
#    gettext \
#    time \
#    flex \
##  Convenience utils specific to this Docker environment.
    bash-completion \
&&  rm -rf /var/lib/apt/lists/* \
&&  git config -f /etc/skel/.gitconfig user.name 'Dockerized OpenWrt Build Environment' \
&&  git config -f /etc/skel/.gitconfig user.email 'dobe@example.com' \
&&  :

##  These should be addjusted to match UID and GID of the mounted dirs.
ARG USER_ID=1000
ARG GROUP_ID=1000
ARG USER_NAME=openwrt
ARG GROUP_NAME=openwrt

RUN set -x \
##  Add non-root user and group, give full sudo access not requiring password.
&&  groupadd -g $GROUP_ID $GROUP_NAME \
&&  useradd -lmg $GROUP_NAME -u $USER_ID $USER_NAME \
&&  echo "$USER_NAME ALL=NOPASSWD: ALL" > /etc/sudoers.d/$USER_NAME

WORKDIR /src

COPY make /

USER $USER_NAME

VOLUME /src
