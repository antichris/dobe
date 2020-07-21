FROM ubuntu:20.04 AS prompt-setup

COPY prompt-setup /
RUN /prompt-setup /etc/skel/.bashrc

FROM ubuntu:20.04

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
    gosu \
&&  rm -rf /var/lib/apt/lists/* \
&&  git config -f /etc/skel/.gitconfig user.name 'Dockerized OpenWrt Build Environment' \
&&  git config -f /etc/skel/.gitconfig user.email 'dobe@example.com' \
&&  :

WORKDIR /src

COPY make pull /
COPY --from=prompt-setup /etc/skel/.bashrc /etc/skel/.bashrc
COPY entrypoint /usr/bin/entrypoint

ENTRYPOINT ["entrypoint"]
CMD ["/bin/bash"]

VOLUME /src
