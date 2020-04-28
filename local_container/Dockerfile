FROM fedora
RUN dnf update -y \
    && dnf install -y httpd-tools \
    && dnf clean all \
    && rm -rf /var/cache/yum
CMD sleep 7200
