# docker build .
# docker images
# docker build -t lastlink/nextcloud-ml .
# FROM nextcloud:27.0-fpm as builder
# # while alpine image would be smaller in size install commands are different and it's harder to get a custom node version installed

# # install pdlib
# # https://github.com/goodspb/pdlib
# RUN apt-get update && \

#     apt-get install -y build-essential wget cmake libx11-dev libopenblas-dev unzip && \
#     rm -rf /var/lib/apt/lists/*

# ARG DLIB_BRANCH=v19.22
# RUN wget -c -q https://github.com/davisking/dlib/archive/$DLIB_BRANCH.tar.gz \

#     && tar xf $DLIB_BRANCH.tar.gz \
#     && mv dlib-* dlib \
#     && cd dlib/dlib \
#     && mkdir build \
#     && cd build \
#     && cmake -DBUILD_SHARED_LIBS=ON -DDLIB_USE_CUDA=1 -DUSE_AVX_INSTRUCTIONS=1 --config Release .. \
#     && make \
#     && make install

# Build and install PDLib on builder

# ARG PDLIB_BRANCH=master
# https://github.com/matiasdelellis/pdlib
# RUN wget -c -q https://github.com/matiasdelellis/pdlib/archive/$PDLIB_BRANCH.zip \

#     && unzip $PDLIB_BRANCH \
#     && mv pdlib-* pdlib \
#     && cd pdlib \
#     && phpize \
#     && ./configure \
#     && make \
#     && make install

# Enable PDlib on builder

# If necesary take the php settings folder uncommenting the next line
# RUN php -i | grep "Scan this dir for additional .ini files"
# RUN echo "extension=pdlib.so" > /usr/local/etc/php/conf.d/pdlib.ini

# Test PDlib instalation on builer

# RUN apt-get update && \

#     apt-get install -y git && \
#     rm -rf /var/lib/apt/lists/*
# RUN git clone https://github.com/matiasdelellis/pdlib-min-test-suite.git \

#     && cd pdlib-min-test-suite \
#     && make

#
# If pass the tests, we are able to create the final image.
#

# end pdlib custom build https://hub.docker.com/r/jlohn/nextcloud-pdlib/dockerfile

# new base image importing .so file
FROM nextcloud:27.0-fpm

# Install dlib and PDlib to image

# COPY --from=builder /usr/local/lib/libdlib.so* /usr/local/lib/

# If is necesary take the php extention folder uncommenting the next line
# RUN php -i | grep extension_dir
# COPY --from=builder /usr/local/lib/php/extensions/no-debug-non-zts-20210902/pdlib.so /usr/local/lib/php/extensions/no-debug-non-zts-20210902/

# Enable PDlib on final image

# RUN echo "extension=pdlib.so" > /usr/local/etc/php/conf.d/pdlib.ini

# Pdlib is already installed, now without all build dependencies.
# You could test again if everything is correct, uncommenting the next lines
#
# RUN apt-get update && \

#     apt-get install -y  --no-install-recommends \
#         git \
#         wget \ 
#         libopenblas-base && \
#     rm -rf /var/lib/apt/lists/*
# RUN git clone https://github.com/matiasdelellis/pdlib-min-test-suite.git \

#     && cd pdlib-min-test-suite \
#     && make

# https://docs.nextcloud.com/server/21/admin_manual/installation/server_tuning.html
# echo $'pm = dynamic\npm.max_children = 120\npm.start_servers = 12\npm.min_spare_servers = 6\npm.max_spare_servers = 18' > www.conf
# /var/php/php/fpm/
# increase max children

# Install required packages
# RUN echo $'pm = dynamic\npm.max_children = 120\npm.start_servers = 12\npm.min_spare_servers = 6\npm.max_spare_servers = 18' > /usr/local/php/php/fpm/www.conf && \
RUN set -ex; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ffmpeg \
        ghostscript \
        # libopenblas-base \
        libmagickcore-6.q16-6-extra \
        procps \
        smbclient \
        supervisor \
#       libreoffice \
    ; \
    rm -rf /var/lib/apt/lists/* 
# echo $'pm = dynamic\npm.max_children = 120\npm.start_servers = 12\npm.min_spare_servers = 6\npm.max_spare_servers = 18' > www.conf

RUN set -ex; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libbz2-dev \
        libc-client-dev \
        libkrb5-dev \
        libsmbclient-dev \
    ; \
    \
    docker-php-ext-configure imap --with-kerberos --with-imap-ssl; \
    docker-php-ext-install \
        bz2 \
        imap \
        # pdlib \
    ; \
    pecl install smbclient; \
    docker-php-ext-enable smbclient; \
    \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark; \
    ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
        | awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); print so }' \
        | sort -u \
        | xargs -r dpkg-query --search \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p \
    /var/log/supervisord \
    /var/run/supervisord \
;

COPY supervisord.conf /


ENV NEXTCLOUD_UPDATE=1

CMD ["/usr/bin/supervisord", "-c", "/supervisord.conf"]
SHELL ["/bin/bash", "-login", "-c"]

ENV NODE_VERSION=20.11.0

ENV NVM_DIR /usr/local/nvm 

RUN mkdir -p $NVM_DIR \
    && curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash \
    && . $NVM_DIR/nvm.sh \
    && nvm install $NODE_VERSION \
    && nvm alias default $NODE_VERSION \
    && nvm use default \
    && node --version \
    && npm --version

# make nodejs work on all users
RUN n=$(which node);n=${n%/bin/node}; chmod -R 755 $n/bin/*; cp -r $n/{bin,lib,share} /usr/local

# Install optional packages
RUN set -ex; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        python3-cffi \
        python3-scipy \
        # python3-pynacl \
        python3-cryptography \
        python3-pip \
    ; \
    rm -rf /var/lib/apt/lists/* 
    # && \
    # python3 -m pip install -U pip  && \
    # python3 -m pip install pillow_heif --break-system-package 