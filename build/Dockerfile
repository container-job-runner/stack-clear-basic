ARG DEV_THEIA
ARG THEIA_INSTALL_DIR=/opt/theia

# == 1. build stage: Theia =====================================================
FROM clearlinux:latest as theia_builder
ARG DEV_THEIA
ARG THEIA_INSTALL_DIR
WORKDIR /root/theia
RUN if [ "$DEV_THEIA" = "TRUE" ] ; then \
    swupd bundle-add 'wget' 'curl' 'python-basic' 'c-basic' ; \
    # --> install nvm (note there is no .bashrc in clearlinux for root, so manually load nvm)
    export HOME=/root ; \
    export NVM_DIR="${HOME}/.nvm" ; \
    curl --silent -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.37.0/install.sh | bash ; \
    # ----> load nvm
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" ; \
    # --> install latest node 12 (erbium)
    nvm install lts/erbium ; \
    nvm use lts/erbium ; \
    # --> install yarn
    npm install -g yarn ; \
    wget https://github.com/container-job-runner/stack-config-files/releases/download/0.1.8-alpha/package.json ; \
    wget https://github.com/container-job-runner/stack-config-files/releases/download/0.1.8-alpha/yarn.lock ; \
    yarn --cache-folder ./ycache ; \
    rm -rf ./ycache ; \
    NODE_OPTIONS="--max_old_space_size=4096" ; \
    yarn theia build ; \
    yarn theia download:plugins ; \
    fi
RUN echo -e '#!/bin/bash\nyarn --cwd '$THEIA_INSTALL_DIR' start $@' > /root/theia_launcher ; \
    chmod a+x /root/theia_launcher ;

# == 2. build stage: Octave Dev Libraries ======================================
# This stage runs very slowly; unfortunately there are no small bundles that octave dev libraries
FROM clearlinux:latest as octave_devel_builder 
ARG LANG_OCTAVE
RUN mkdir /builder-includes 
RUN if [ "$LANG_OCTAVE" = "TRUE" ] ; then \
    swupd bundle-add R-extras-dev ; \
    cp -r /usr/include/octave-* /builder-includes ; \
    fi

# == 3. main Dockerfile: Clear Linux ===========================================
FROM clearlinux:latest

# ----> ARGS: languages
ARG LANG_C
ARG LANG_FORTRAN
ARG LANG_PYTHON3
ARG LANG_JULIA
ARG LANG_R
ARG LANG_OCTAVE
ARG LANG_LATEX
# ----> ARGS: libraries
ARG LIB_MATPLOTLIB
ARG LIB_LINALG
ARG LIB_OPENMPI
ARG LIB_X11
ARG LIB_RAY
# ----> ARGS: dev environemnts
ARG DEV_JUPYTER
ARG DEV_THEIA
ARG DEV_CLI
# ----> ARGS: additional software
ARG ASW_SPACK
ARG ASW_VNC
ARG ASW_SSHD
ARG ASW_SLURM
# ----> ARGS: additional options
ARG THEIA_INSTALL_DIR
ARG ASW_VNC_PASSWORD=password
ARG SHARED_STORAGE_DIR=/opt/shared

# -- copy root install files to /opt/build-scripts -----------------------------
RUN mkdir -p /opt/build-scripts/
WORKDIR /opt/build-scripts
COPY scripts/root/ ./
RUN chmod -R +x ./

# == User Creation ==============================================================
ARG USER_NAME=user
ARG USER_PASSWORD
ARG GRANT_SUDO=false
ARG USER_ID
ARG GROUP_ID
ARG ROOT_PASSWORD
ARG DYNAMIC_USERMOD

# -- Generate restricted user account ------------------------------------------
RUN /opt/build-scripts/user-create.sh

# == Software Installation (Root) ===============================================
COPY --from=octave_devel_builder /builder-includes /usr/include/
RUN ./install.sh
COPY --from=theia_builder /root/theia $THEIA_INSTALL_DIR
COPY --from=theia_builder /root/theia_launcher /usr/local/bin/theia

# == Software Installation (User) ===============================================

# -- copy user config files to ~/.build-config
RUN mkdir -p "/home/$USER_NAME/.build/config"
WORKDIR "/home/$USER_NAME/.build/config"
COPY config/user/ ./
# -- copy user install scripts to ~/.build-scripts -----------------------------
RUN mkdir -p "/home/$USER_NAME/.build/scripts"
WORKDIR "/home/$USER_NAME/.build/scripts"
COPY scripts/user/ ./
RUN chown -R $USER_ID:$GROUP_ID "/home/$USER_NAME/.build"

USER $USER_NAME
RUN chmod -R +x ./
# -- install user dependencies -------------------------------------------------
RUN ./install.sh

# -- install extra root dependencies (placed here to prevent full rebuild) -----
USER 0
RUN /opt/build-scripts/install-extra.sh

# -- install extra user dependencies (placed here to prevent full rebuild) -----
USER $USER_NAME
RUN /home/$USER_NAME/.build/scripts/install-extra.sh

# -- remove user build directory -----------------------------------------------
RUN rm -r /home/$USER_NAME/.build

WORKDIR "/home/$USER_NAME"
ENTRYPOINT [ "/opt/build-scripts/entrypoint.sh"]