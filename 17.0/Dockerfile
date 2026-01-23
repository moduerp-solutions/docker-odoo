FROM ubuntu:jammy
LABEL org.opencontainers.image.authors="ModuERP Solutions <moduerp.solutions@gmail.com>"

SHELL ["/bin/bash", "-xo", "pipefail", "-c"]

# Generate locale C.UTF-8 for postgres and general locale data
ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    LC_ALL=en_US.UTF-8 \
    TZ=Etc/UTC

# Install some deps, lessc and less-plugin-clean-css
RUN --mount=type=cache,target=/var/lib/apt,sharing=locked \
    DEBIAN_FRONTEND=noninteractive apt-get update \
    && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo $TZ > /etc/timezone \
    && DEBIAN_FRONTEND=noninteractive apt-get upgrade -y > /dev/null \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        dirmngr \
        fonts-noto-cjk \
        gnupg \
        libssl-dev \
        node-less \
        npm \
        xz-utils \
        build-essential \
        geoip-database \
        git \
        libcairo2-dev \
        locales \
        patch \
        pkg-config \
        sudo \
        vim \
        less \
        zip \
        unzip \
        wget \
        fontconfig \
        xfonts-75dpi \
        xfonts-base \
        libjpeg-turbo8 \
        libxml2-dev \
        libxslt1-dev \
        libffi-dev \
        libsasl2-dev \
        libldap2-dev \
        libjpeg-dev \
        zlib1g-dev \
        libpq-dev \
        software-properties-common > /dev/null

# Setting locale
RUN sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen \
    && locale-gen

# Install wkhtmltopdf
ARG TARGETARCH
RUN if [ -z "${TARGETARCH}" ]; then \
        TARGETARCH="$(dpkg --print-architecture)"; \
    fi; \
    WKHTMLTOPDF_ARCH=${TARGETARCH} && \
    case ${TARGETARCH} in \
    "amd64") WKHTMLTOPDF_ARCH=amd64 && WKHTMLTOPDF_SHA=967390a759707337b46d1c02452e2bb6b2dc6d59  ;; \
    "arm64")  WKHTMLTOPDF_SHA=90f6e69896d51ef77339d3f3a20f8582bdf496cc  ;; \
    "ppc64le" | "ppc64el") WKHTMLTOPDF_ARCH=ppc64el && WKHTMLTOPDF_SHA=5312d7d34a25b321282929df82e3574319aed25c  ;; \
    esac \
    && curl -o wkhtmltox.deb -sSL https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_${WKHTMLTOPDF_ARCH}.deb \
    && echo ${WKHTMLTOPDF_SHA} wkhtmltox.deb | sha1sum -c - \
    && apt-get install -y --no-install-recommends ./wkhtmltox.deb \
    && rm -rf /var/lib/apt/lists/* wkhtmltox.deb

# Install latest postgresql-client
RUN echo 'deb http://apt.postgresql.org/pub/repos/apt/ jammy-pgdg main' > /etc/apt/sources.list.d/pgdg.list \
    && GNUPGHOME="$(mktemp -d)" \
    && export GNUPGHOME \
    && repokey='B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8' \
    && gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "${repokey}" \
    && gpg --batch --armor --export "${repokey}" > /etc/apt/trusted.gpg.d/pgdg.gpg.asc \
    && gpgconf --kill all \
    && rm -rf "$GNUPGHOME" \
    && apt-get update  \
    && apt-get install --no-install-recommends -y postgresql-client \
    && rm -f /etc/apt/sources.list.d/pgdg.list \
    && rm -rf /var/lib/apt/lists/*

# Install rtlcss (on Debian buster)
RUN npm install -g rtlcss \
    && npm cache clean --force

# Config git
RUN git config --system advice.detachedHead false \
    && git config --system --add safe.directory /opt/odoo/customizations

# Create directory for Odoo
# /var/lib/odoo : filestore, sessions, etc.
# /opt/odoo : odoo code, custom modules, etc.
RUN if id -u ubuntu >/dev/null 2>&1; then userdel -r ubuntu; fi \
    && mkdir -p /var/lib/odoo /opt/odoo \
    && useradd -MUu 1000 -s /bin/bash -d /opt/odoo odoo \
    && chown -R odoo:odoo /var/lib/odoo /opt/odoo

# Copy entrypoint script and Odoo configuration file
COPY ./entrypoint.sh /
COPY ./odoo.conf /opt/odoo/

# Link odoo-bin to /usr/local/bin
RUN ln -s /opt/odoo/odoo-requirements/odoo/odoo-bin /usr/local/bin/

# Install Python3.13
RUN add-apt-repository ppa:deadsnakes/ppa
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    python3.13 \
    python3.13-venv \
    python3.13-dev \
    python3-pip \
 && rm -rf /var/lib/apt/lists/*

# Make python3.13 the default python
RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.13 1 \
 && update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.13 1

# Upgrade pip tooling
RUN python3.13 -m ensurepip --upgrade \
 && python3.13 -m pip install --upgrade setuptools wheel pip

# Install Odoo python dependencies
ENV ODOO_VERSION=17.0
RUN git clone --depth=1 --branch ${ODOO_VERSION} https://github.com/odoo/odoo.git /tmp/odoo \
    && pip3 install --no-cache-dir -r /tmp/odoo/requirements.txt
RUN rm -rf /tmp/odoo

# Remove requirements file to avoid false positives in vulnerability scans
RUN find /opt/odoo -name requirements.txt -delete

# Set the default config file
ENV ODOO_RC=/opt/odoo/odoo.conf

# Copy wait-for-psql script
COPY wait-for-psql.py /usr/local/bin/wait-for-psql.py

RUN apt-get update \
    && git config --system core.checkStat minimal \
    && echo "odoo ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/odoo

USER odoo 
EXPOSE 8069 8071 8072
