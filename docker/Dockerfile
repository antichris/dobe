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
    build-essential=12.8ubuntu1.1 \
    gawk=1:5.0.1+dfsg-1 \
    gcc-multilib=4:9.3.0-1ubuntu2 \
    git=1:2.25.1-1ubuntu3.1 \
    libncurses5-dev=6.2-0ubuntu2 \
    libssl-dev=1.1.1f-1ubuntu2.3 \
    openjdk-8-jdk-headless=8u292-b10-0ubuntu1~20.04 \
    python3=3.8.2-0ubuntu2 \
    python-is-python2=2.7.17-4 \
    rsync=3.1.3-8 \
    subversion=1.13.0-3 \
    unzip=6.0-25ubuntu1 \
    wget=1.20.3-1ubuntu1 \
    zlib1g-dev=1:1.2.11.dfsg-2ubuntu1.2 \
#    gettext \
#    time \
#    flex \
##  Convenience utils specific to this Docker environment.
    bash-completion=1:2.10-1ubuntu1 \
    gosu=1.10-1ubuntu0.20.04.1 \
    openssh-client=1:8.2p1-4ubuntu0.2 \
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
