ARG NPM_REGISTRY=https://registry.npmjs.org/
ARG PYTHON_VERSION=3.8.6
ARG DOCKERIZE_VERSION=v0.6.1
ARG EDX_PLATFORM_REPOSITORY=https://github.com/edx/edx-platform.git
ARG EDX_PLATFORM_VERSION={{ OPENEDX_COMMON_VERSION }}
ARG OPENEDX_I18N_VERSION={{ OPENEDX_COMMON_VERSION }}

###################################### minimal ########################################
# Extends docker.io/ubuntu:20.04.
# - Installs base system requirements for future stages.
#######################################################################################
FROM docker.io/ubuntu:20.04 as minimal
LABEL io.overhang.maintainer="Overhang.io"
LABEL io.overhang.email="contact@overhang.io"

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y \
        build-essential \
        curl \
        git \
        language-pack-en
ENV LC_ALL en_US.UTF-8

###################################### minimal ########################################
# Extends minimal.
# - Installs python with pyenv in /opt/pyenv.
# - Creates virtualenv in /openedx/venv.
#######################################################################################
FROM minimal as python
RUN apt-get update \
    && apt-get install -y \
        libssl-dev \
        zlib1g-dev \
        libbz2-dev \
        libreadline-dev \
        libsqlite3-dev \
        wget \
        curl \
        llvm \
        libncurses5-dev \
        libncursesw5-dev \
        xz-utils \
        tk-dev \
        libffi-dev \
        liblzma-dev \
        python-openssl \
        git \
    && rm -rf /var/lib/apt/lists/*
ENV PYENV_ROOT /opt/pyenv
RUN git clone https://github.com/pyenv/pyenv $PYENV_ROOT --branch v1.2.21 --depth 1
ARG PYTHON_VERSION
RUN $PYENV_ROOT/bin/pyenv install $PYTHON_VERSION
RUN $PYENV_ROOT/versions/$PYTHON_VERSION/bin/python -m venv /openedx/venv

##################################### dockerize #######################################
# Extends minimal.
# - Installs dockerize into /usr/local/bin
#######################################################################################
FROM minimal as dockerize
ARG DOCKERIZE_VERSION
RUN curl -L \
        -o /tmp/dockerize.tar.gz \
        https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && tar -C /usr/local/bin -xzvf /tmp/dockerize.tar.gz \
    && rm /tmp/dockerize.tar.gz

####################################### code ##########################################
# Extends minimal.
# - Checks out the OpenEdx Platform source code.
#######################################################################################
FROM minimal as code
ARG EDX_PLATFORM_REPOSITORY
ARG EDX_PLATFORM_VERSION
RUN mkdir -p /openedx/edx-platform \
    && git clone $EDX_PLATFORM_REPOSITORY \
        --branch $EDX_PLATFORM_VERSION \
        --depth 1 \
        /openedx/edx-platform

WORKDIR /openedx/edx-platform
{% if patch("openedx-dockerfile-git-patches-default") %}
# Custom edx-platform patches
{{ patch("openedx-dockerfile-git-patches-default") }}
{% else %}
# Patch edx-platform
RUN curl --silent https://github.com/overhangio/edx-platform/commit/8ecc1903ca9170a719c0e63e99fb231822eb26d8.patch | git apply -
{% endif %}

###################################### locales ########################################
# Extends minimal.
# - Downloads extra locales to /openedx/locale/contrib/locale.
#######################################################################################
FROM minimal as locales
ARG OPENEDX_I18N_VERSION
RUN cd /tmp \
    && curl -L -o openedx-i18n.tar.gz https://github.com/openedx/openedx-i18n/archive/$OPENEDX_I18N_VERSION.tar.gz \
    && tar xzf /tmp/openedx-i18n.tar.gz \
    && mkdir -p /openedx/locale/contrib \
    && mv openedx-i18n-*/edx-platform/locale /openedx/locale/contrib \
    && rm -rf openedx-i18n*

############################### python-requirements ###################################
# Extends python.
# - Installs python requirements into the virtualenv.
#######################################################################################
FROM python as python-requirements
ENV PATH /openedx/venv/bin:${PATH}
ENV VIRTUAL_ENV /openedx/venv/

RUN apt-get update \
    && apt-get install -y \
        software-properties-common \
        libmysqlclient-dev \
        libxmlsec1-dev \
    && rm -rf /var/lib/apt/lists/*

# Install the right version of pip/setuptools
RUN pip install setuptools==44.1.0 pip==20.0.2 wheel==0.34.2

# Install scorm xblock
RUN pip install "openedx-scorm-xblock<13.0.0,>=12.0.0"

# Install django-redis for using redis as a django cache
RUN pip install django-redis==4.12.1

# Install uwsgi
RUN pip install uwsgi==2.0.19.1

# Note that this means that we need to reinstall all requirements whenever there is a
# change in edx-platform, which sucks. But there is no obvious alternative, as we need
# to install some packages from edx-platform.
COPY --from=code /openedx/edx-platform /openedx/edx-platform
WORKDIR /openedx/edx-platform

# Install base requirements
RUN pip install -r ./requirements/edx/base.txt

# Install private requirements: this is useful for installing custom xblocks.
COPY ./requirements/ /openedx/requirements
RUN cd /openedx/requirements/ \
    && touch ./private.txt \
    && pip install -r ./private.txt

{{ patch("openedx-dockerfile-post-python-requirements") }}

################################ nodejs-requirements ##################################
# Extends python.
# - Install nodejs with nodeenv in /openedx/nodeenv.
#######################################################################################
FROM python as nodejs-requirements
ENV PATH /openedx/nodeenv/bin:/openedx/venv/bin:${PATH}

# Install nodeenv with the version provided by edx-platform
RUN pip install nodeenv==1.6.0
RUN nodeenv /openedx/nodeenv --node=12.13.0 --prebuilt

# Install nodejs requirements
COPY --from=code /openedx/edx-platform/package.json /openedx/edx-platform/package.json
WORKDIR /openedx/edx-platform
ARG NPM_REGISTRY
RUN npm install --verbose --registry=${NPM_REGISTRY}

##################################### production ######################################
# Extends minimal. This is the production image with system and python requirements.
#######################################################################################
FROM minimal as production

# Install system requirements
RUN apt-get update \
    && apt-get install -y \
        gettext \
        gfortran \
        graphviz \
        graphviz-dev \
        libffi-dev \
        libfreetype6-dev \
        libgeos-dev \
        libjpeg8-dev \
        liblapack-dev \
        libmysqlclient-dev \
        libpng-dev \
        libsqlite3-dev \
        libxmlsec1-dev \
        lynx \
        ntp \
        pkg-config \
        rdfind \
    && rm -rf /var/lib/apt/lists/*

COPY --from=dockerize /usr/local/bin/dockerize /usr/local/bin/dockerize
COPY --from=locales /openedx/locale/contrib/locale /openedx/locale/contrib/locale
COPY --from=python /opt/pyenv /opt/pyenv
COPY --from=python-requirements /openedx/venv /openedx/venv
COPY --from=python-requirements /openedx/requirements /openedx/requirements
COPY --from=python-requirements /openedx/edx-platform /openedx/edx-platform
COPY --from=nodejs-requirements /openedx/nodeenv /openedx/nodeenv
COPY --from=nodejs-requirements /openedx/edx-platform/node_modules /openedx/edx-platform/node_modules

ENV PATH /openedx/venv/bin:./node_modules/.bin:/openedx/nodeenv/bin:${PATH}
ENV VIRTUAL_ENV /openedx/venv/

WORKDIR /openedx/edx-platform

# Create folder that will store lms/cms.env.json files, as well as
# the tutor-specific settings files.
RUN mkdir -p /openedx/config ./lms/envs/tutor ./cms/envs/tutor
COPY revisions.yml /openedx/config/
ENV LMS_CFG /openedx/config/lms.env.json
ENV STUDIO_CFG /openedx/config/cms.env.json
ENV REVISION_CFG /openedx/config/revisions.yml
COPY settings/lms/*.py ./lms/envs/tutor/
COPY settings/cms/*.py ./cms/envs/tutor/

# Copy user-specific locales to /openedx/locale/user/locale and compile them
COPY ./locale/ /openedx/locale/user/locale/
RUN cd /openedx/locale/user && \
    django-admin.py compilemessages -v1

# Compile i18n strings: in some cases, js locales are not properly compiled out of the box
# and we need to do a pass ourselves. Also, we need to compile the djangojs.js files for
# the downloaded locales.
RUN ./manage.py lms --settings=tutor.i18n compilejsi18n
RUN ./manage.py cms --settings=tutor.i18n compilejsi18n

# Copy scripts
COPY ./bin /openedx/bin
RUN chmod a+x /openedx/bin/*
ENV PATH /openedx/bin:${PATH}

{{ patch("openedx-dockerfile-pre-assets") }}

# Collect production assets. By default, only assets from the default theme
# will be processed. This makes the docker image lighter and faster to build.
# Only the custom themes added to /openedx/themes will be compiled.
# Here, we don't run "paver update_assets" which is slow, compiles all themes
# and requires a complex settings file. Instead, we decompose the commands
# and run each one individually to collect the production static assets to
# /openedx/staticfiles.
ENV NO_PYTHON_UNINSTALL 1
ENV NO_PREREQ_INSTALL 1

# We need to rely on a separate openedx-assets command to accelerate asset processing.
# For instance, we don't want to run all steps of asset collection every time the theme
# is modified.
RUN openedx-assets xmodule \
    && openedx-assets npm \
    && openedx-assets webpack --env=prod \
    && openedx-assets common
COPY ./themes/ /openedx/themes/
RUN openedx-assets themes \
    && openedx-assets collect --settings=tutor.assets \
    # De-duplicate static assets with symlinks
    && rdfind -makesymlinks true -followsymlinks true /openedx/staticfiles/

# Create a data directory, which might be used (or not)
RUN mkdir /openedx/data

# service variant is "lms" or "cms"
ENV SERVICE_VARIANT lms
ENV SETTINGS tutor.production

{{ patch("openedx-dockerfile") }}

# Entrypoint will set right environment variables
ENTRYPOINT ["docker-entrypoint.sh"]

# Run server
EXPOSE 8000
CMD uwsgi \
    --static-map /static=/openedx/staticfiles/ \
    --static-map /media=/openedx/media/ \
    --http 0.0.0.0:8000 \
    --thunder-lock \
    --single-interpreter \
    --enable-threads \
    --processes=${UWSGI_WORKERS:-2} \
    --buffer-size=8192 \
    --wsgi-file ${SERVICE_VARIANT}/wsgi.py
