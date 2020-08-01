## This Source Code Form is subject to the terms of the Mozilla Public
## License, v. 2.0. If a copy of the MPL was not distributed with this
## file, You can obtain one at https://mozilla.org/MPL/2.0/.

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
&&  ssh-keyscan \
    github.com \
    gitlab.com \
    bitbucket.org \
    >> /etc/ssh/ssh_known_hosts \
&&  :

WORKDIR /src

COPY make pull /
COPY --from=prompt-setup /etc/skel/.bashrc /etc/skel/.bashrc
COPY entrypoint /usr/bin/entrypoint

ENTRYPOINT ["entrypoint"]
CMD ["/bin/bash"]

VOLUME /src
