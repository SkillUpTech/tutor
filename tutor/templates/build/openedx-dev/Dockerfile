ARG USERID=1000
ARG PROD_IMAGE={{ DOCKER_IMAGE_OPENEDX }}

######################################### base ########################################
# - Extends docker.io/ubuntu:20.04
# - Install common system utilities.
# - Installs some additional software for the development user.
#   - Namely sudo, git, and some text editing and network utilities.
# - Installs platform requirements.
#######################################################################################
FROM docker.io/ubuntu:20.04 AS base
MAINTAINER Overhang.io <contact@overhang.io>

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y \
        build-essential \
        curl \
        git \
        language-pack-en \
    && rm -rf /var/lib/apt/lists/*
ENV LC_ALL en_US.UTF-8

# Install useful system requirements. We could put these in the previous apt-get RUN, but
# we separate them to allow for layer reuse from the prod image when using --cache-from.
RUN apt-get update \
    && apt-get install -y \
        vim \
        iputils-ping \
        dnsutils \
        telnet \
        sudo \
    && rm -rf /var/lib/apt/lists/*

# Install platform requirements. Separated for same reason as above.
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

###################################### prod-alias #####################################
# Just allows us to refer to the production openedx image as 'prod-alias' later on.
# See https://stackoverflow.com/questions/43473236/docker-build-arg-and-copy.
#######################################################################################
FROM ${PROD_IMAGE} AS prod-alias

####################################### dev-user ######################################
# Extends base. The instructions below could simply be added to 'base', but separating
# the stage allows for easier debugging.
# - Creates the dev user.
# - Changes to the dev user.
#######################################################################################
FROM base AS dev-user

# Configure new user.
ENV HOME=/openedx
ARG USERID 
RUN useradd -l \
    --uid ${USERID} \
    --groups sudo \
    --home-dir ${HOME} \
    --create-home \
    --shell /bin/bash \
    --password openedx \
    openedx

# Enable passwordless sudo.
RUN sed -i.bak \
    -e 's/%sudo\s\+ALL=(ALL\(:ALL\)\?)\s\+ALL/%sudo ALL=NOPASSWD:ALL/g' \
    /etc/sudoers

# Change to the new user and run 'sudo' (to suppress output for future runs).
USER openedx
WORKDIR ${HOME}
RUN sudo echo "Executing first 'sudo' call as ${USERNAME}."
RUN echo "source /openedx/venv/bin/activate; source /openedx/nodeenv/bin/activate" >> ${HOME}/.bashrc

######################################### dev ########################################
# Extends dev-user.
# - Copies dockerize, python, and /openedx from the production image with 
#   permissions and ownership changed to the dev user.
# - Installs development requirements.
# - Sets necessary environment variables.
#######################################################################################
FROM dev-user AS dev

ENV LMS_CFG /openedx/config/lms.env.json
ENV STUDIO_CFG /openedx/config/cms.env.json
ENV REVISION_CFG /openedx/config/revisions.yml
ENV PATH /openedx/venv/bin:./node_modules/.bin:/openedx/nodeenv/bin:${PATH}
ENV PYENV_ROOT /opt/pyenv
ENV VIRTUAL_ENV /openedx/venv/
COPY --from=prod-alias --chown=openedx:openedx /usr/local/bin/dockerize /usr/local/bin/dockerize
COPY --from=prod-alias --chown=openedx:openedx /opt/pyenv /opt/pyenv
COPY --from=prod-alias --chown=openedx:openedx /openedx /openedx

WORKDIR /openedx/edx-platform
# Install dev python requirements
RUN pip install -r requirements/edx/development.txt

# We install ipython from source to avoid too many deprecation warnings
# https://github.com/ipython/ipython/issues/12206
# We might be able to avoid this once they make a release later than 7.19.0.
RUN pip install \
        ipdb==0.13.4 \
        git+https://github.com/ipython/ipython.git@d0649a54a8936a8019d54549779dc92bcbde4e68#egg=ipython

# Reinstall private requirements so they take precedence over openedx development requirements.
# This means private requirements can override any module, including those included in the platform's requirements.
RUN cd /openedx/requirements/ \
    && touch ./private.txt \
    && pip install -r ./private.txt

{{ patch("openedx-dev-dockerfile-post-python-requirements") }}

COPY --chown=openedx:openedx --chmod=a+x ./bin /openedx/dev_bin
ENV PATH "/openedx/dev_bin:/openedx/bin:${PATH}"

# Recompile static assets: in development mode all static assets are stored in edx-platform,
# and the location of these files is stored in webpack-stats.json. If we don't recompile
# static assets, then production assets will be served instead.
RUN rm -r /openedx/staticfiles && \
    mkdir /openedx/staticfiles && \
    openedx-assets webpack --env=dev

ENV SERVICE_VARIANT lms

######################################## test #########################################
# Extends dev. Sets the tests entrypoint. Must be built with `--target test`.
#######################################################################################
FROM dev AS test
ENV SETTINGS test
ENTRYPOINT ["docker-entrypoint-test.sh"]
CMD pytest

####################################### final #########################################
# Extends dev to create the default final image for this Dockerfile.
#######################################################################################
FROM dev AS final
ENV SETTINGS tutor.development
ENTRYPOINT ["docker-entrypoint.sh"]
