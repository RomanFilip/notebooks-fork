ARG BASE_IMAGE
FROM ${BASE_IMAGE}

ARG SOURCE_CODE=rstudio/c9s-python-3.11

LABEL name="odh-notebook-rstudio-server-c9s-python-3.11" \
      summary="RStudio Server image with python 3.11 based on CentOS Stream 9" \
      description="RStudio server image with python 3.11 based on CentOS Stream 9" \
      io.k8s.display-name="RStudio server image with python 3.11 based on CentOS Stream 9" \
      io.k8s.description="RStudio server image with python 3.11 based on CentOS Stream 9" \
      authoritative-source-url="https://github.com/opendatahub-io/notebooks" \
      io.openshift.build.commit.ref="main" \
      io.openshift.build.source-location="https://github.com/opendatahub-io/notebooks/tree/main/rstudio/c9s-python-3.11" \
      io.openshift.build.image="quay.io/opendatahub/workbench-images:rstudio-c9s-python-3.11"

USER 0

ENV R_VERSION=4.4.1

# Install R
RUN yum install -y yum-utils && \
    yum-config-manager --enable crb && \
    yum install -y https://download.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm && \
    INSTALL_PKGS="R-core R-core-devel R-java R-Rcpp R-highlight \
    R-littler R-littler-examples openssl-libs compat-openssl11" && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    echo 'options(repos = c(CRAN = "https://cran.rstudio.com/"), download.file.method = "libcurl")' >> /usr/lib64/R/etc/Rprofile.site && \
    (umask 002;touch /usr/lib64/R/etc/Renviron.site) && \
    yum -y clean all --enablerepo='*'

# set R library to default (used in install.r from littler)
RUN chmod -R a+w /usr/lib64/R/library
ENV LIBLOC /usr/lib64/R/library

# set User R Library path
RUN mkdir -p /opt/app-root/bin/Rpackages/4.4 && chmod -R a+w /opt/app-root/bin/Rpackages/4.4
ENV R_LIBS_USER /opt/app-root/bin/Rpackages/4.4

WORKDIR /tmp/

# Install RStudio
RUN wget --progress=dot:giga https://download2.rstudio.org/server/rhel9/x86_64/rstudio-server-rhel-2024.04.2-764-x86_64.rpm && \
    yum install -y rstudio-server-rhel-2024.04.2-764-x86_64.rpm && \
    rm rstudio-server-rhel-2024.04.2-764-x86_64.rpm && \
    yum -y clean all  --enablerepo='*'

# Specific RStudio config and fixes
RUN chmod 1777 /var/run/rstudio-server && \
    mkdir -p /usr/share/doc/R
COPY ${SOURCE_CODE}/rsession.conf /etc/rstudio/rsession.conf

# package installation
# install necessary texlive-framed package to make Knit R markup to PDF rendering possible
RUN dnf install -y libsodium-devel.x86_64 libgit2-devel.x86_64 libcurl-devel harfbuzz-devel.x86_64 fribidi-devel.x86_64 cmake "flexiblas-*" texlive-framed \
    && dnf clean all && rm -rf /var/cache/yum
# Install R packages
RUN R -e "install.packages('remotes')"
RUN R -e "require('remotes'); \
    remotes::install_version('Rcpp','1.0.13'); \
    remotes::install_version('tidyverse','2.0.0'); \
    remotes::install_version('tidymodels','1.2.0'); \
    remotes::install_version('plumber','1.2.2'); \
    remotes::install_version('vetiver','0.2.5'); \
    remotes::install_version('devtools','2.4.5');"

# Install NGINX to proxy RStudio and pass probes check
ENV NGINX_VERSION=1.24 \
    NGINX_SHORT_VER=124 \
    NGINX_CONFIGURATION_PATH=${APP_ROOT}/etc/nginx.d \
    NGINX_CONF_PATH=/etc/nginx/nginx.conf \
    NGINX_DEFAULT_CONF_PATH=${APP_ROOT}/etc/nginx.default.d \
    NGINX_CONTAINER_SCRIPTS_PATH=/usr/share/container-scripts/nginx \
    NGINX_APP_ROOT=${APP_ROOT} \
    NGINX_LOG_PATH=/var/log/nginx \
    NGINX_PERL_MODULE_PATH=${APP_ROOT}/etc/perl

# Modules does not exist
RUN yum -y module enable nginx:$NGINX_VERSION && \
    INSTALL_PKGS="nss_wrapper bind-utils gettext hostname nginx nginx-mod-stream nginx-mod-http-perl fcgiwrap initscripts chkconfig supervisor" && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    rpm -V $INSTALL_PKGS && \
    nginx -v 2>&1 | grep -qe "nginx/$NGINX_VERSION\." && echo "Found VERSION $NGINX_VERSION" && \
    yum -y clean all --enablerepo='*'

COPY --chown=1001:0 ${SOURCE_CODE}/supervisord/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Copy extra files to the image.
COPY ${SOURCE_CODE}/nginx/root/ /

# Changing ownership and user rights to support following use-cases:
# 1) running container on OpenShift, whose default security model
#    is to run the container under random UID, but GID=0
# 2) for working root-less container with UID=1001, which does not have
#    to have GID=0
# 3) for default use-case, that is running container directly on operating system,
#    with default UID and GID (1001:0)
# Supported combinations of UID:GID are thus following:
# UID=1001 && GID=0
# UID=<any>&& GID=0
# UID=1001 && GID=<any>
RUN sed -i -f ${NGINX_APP_ROOT}/nginxconf.sed ${NGINX_CONF_PATH} && \
    mkdir -p ${NGINX_APP_ROOT}/etc/nginx.d/ && \
    mkdir -p ${NGINX_APP_ROOT}/etc/nginx.default.d/ && \
    mkdir -p ${NGINX_APP_ROOT}/api/ && \
    mkdir -p ${NGINX_CONTAINER_SCRIPTS_PATH}/nginx-start && \
    mkdir -p ${NGINX_LOG_PATH} && \
    mkdir -p ${NGINX_PERL_MODULE_PATH} && \
    chown -R 1001:0 ${NGINX_CONF_PATH} && \
    chown -R 1001:0 ${NGINX_APP_ROOT}/etc && \
    chown -R 1001:0 ${NGINX_CONTAINER_SCRIPTS_PATH}/nginx-start && \
    chown -R 1001:0 /var/lib/nginx /var/log/nginx /run && \
    chmod    ug+rw  ${NGINX_CONF_PATH} && \
    chmod -R ug+rwX ${NGINX_APP_ROOT}/etc && \
    chmod -R ug+rwX ${NGINX_CONTAINER_SCRIPTS_PATH}/nginx-start && \
    chmod -R ug+rwX /var/lib/nginx /var/log/nginx /run && \
    rpm-file-permissions

# Configure nginx
COPY ${SOURCE_CODE}/nginx/serverconf/ /opt/app-root/etc/nginx.default.d/
COPY ${SOURCE_CODE}/nginx/httpconf/ /opt/app-root/etc/nginx.d/
COPY ${SOURCE_CODE}/nginx/api/ /opt/app-root/api/

# Launcher
WORKDIR /opt/app-root/bin

COPY ${SOURCE_CODE}/utils utils/
COPY ${SOURCE_CODE}/run-rstudio.sh ${SOURCE_CODE}/setup_rstudio.py ${SOURCE_CODE}/rsession.sh ${SOURCE_CODE}/run-nginx.sh ./

WORKDIR /opt/app-root/src

USER 1001

CMD ["/opt/app-root/bin/run-rstudio.sh"]
