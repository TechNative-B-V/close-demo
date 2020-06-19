FROM ubuntu:focal

MAINTAINER Mathijs van Veluw - TechNative B.V. <mathijs@technative.nl>

ENV container docker
ENV DEBIAN_FRONTEND=noninteractive LC_ALL=C.UTF-8 TZ=UTC TERM=xterm-256color

RUN apt-get update \
    && apt-get install --no-install-recommends -y \
    dbus systemd systemd-sysv python python-apt gpg sudo bash ca-certificates \
    && apt-get clean \
    && rm -rf /usr/share/doc/* /usr/share/man/* /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Don't start any optional services except for the few we need.
RUN find /etc/systemd/system \
    /lib/systemd/system \
    -path '*.wants/*' \
    -not -name '*dbus*' \
    -not -name '*journald*' \
    -not -name '*systemd-tmpfiles*' \
    -not -name '*systemd-user-sessions*' \
    -exec rm \{} \;

# SHELL ["/bin/bash", "-c"]
# RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [[ $i == systemd-tmpfiles-setup.service ]] || rm -f $i; done); \
#     rm -f /lib/systemd/system/multi-user.target.wants/*; \
#     rm -f /etc/systemd/system/*.wants/*; \
#     rm -f /lib/systemd/system/local-fs.target.wants/*; \
#     rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
#     rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
#     rm -f /lib/systemd/system/basic.target.wants/*; \
#     rm -f /lib/systemd/system/anaconda.target.wants/*; \
#     rm -f /lib/systemd/system/plymouth*; \
#     rm -f /lib/systemd/system/systemd-update-utmp*;

RUN systemctl set-default multi-user.target

STOPSIGNAL SIGRTMIN+3

CMD ["/sbin/init"]
