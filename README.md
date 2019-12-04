# Interintel Tech Challenge
## Security configuration setup on the base image (preferably CentOS 8)

We use centos:latest tag to always get the most recent version currently available.
We shall integrate our base image with systemd to start the Docker daemon.

```
    FROM centos:8
    ENV container docker
    RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == \
    systemd-tmpfiles-setup.service ] || rm -f $i; done); \
    rm -f /lib/systemd/system/multi-user.target.wants/*;\
    rm -f /etc/systemd/system/*.wants/*;\
    rm -f /lib/systemd/system/local-fs.target.wants/*; \
    rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
    rm -f /lib/systemd/system/basic.target.wants/*;\
    rm -f /lib/systemd/system/anaconda.target.wants/*;
    VOLUME [ "/sys/fs/cgroup" ]
    CMD ["/usr/sbin/init"]
```
This Dockerfile deletes a number of unit files which might cause issues. We now proceed to build our base image.
``` $ docker build --rm -t local/cent .```
