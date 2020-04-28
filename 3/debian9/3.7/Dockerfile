FROM gcr.io/google-appengine/debian10 AS plugins

COPY license-checksums /

RUN set -eux; \
  apt-get update && apt-get install -y ca-certificates wget; \
  \
  mkdir /plugins && cd /plugins; \
  \
  readonly base_url='https://github.com/deadtrickster/prometheus_rabbitmq_exporter/releases/download/v3.7.9.1'; \
  wget "${base_url}/accept-0.3.5.ez"; \
  wget "${base_url}/prometheus-4.3.0.ez"; \
  wget "${base_url}/prometheus_cowboy-0.1.7.ez"; \
  wget "${base_url}/prometheus_httpd-2.1.10.ez"; \
  wget "${base_url}/prometheus_process_collector-1.4.3.ez"; \
  wget "${base_url}/prometheus_rabbitmq_exporter-3.7.9.1.ez"; \
  \
  mkdir -p "/licenses" && cd "/licenses"; \
  readonly git_base='https://raw.githubusercontent.com/deadtrickster'; \
  \
  mkdir accept && wget -O accept/LICENSE "${git_base}/accept/master/LICENSE"; \
  mkdir prometheus && wget -O prometheus/LICENSE "${git_base}/prometheus.erl/master/LICENSE"; \
  mkdir prometheus_cowboy && wget -O prometheus_cowboy/LICENSE "${git_base}/prometheus-cowboy/master/LICENSE"; \
  mkdir prometheus_httpd && wget -O prometheus_httpd/LICENSE "${git_base}/prometheus-httpd/master/LICENSE"; \
  mkdir prometheus_rabbitmq_exporter && wget -O prometheus_rabbitmq_exporter/LICENSE "${git_base}/prometheus_rabbitmq_exporter/master/LICENSE"; \
  \
  sha256sum -c /license-checksums


FROM gcr.io/google-appengine/debian10

RUN set -eux; \
  apt-get update; \
  apt-get install -y --no-install-recommends \
# grab gosu for easy step-down from root
    gosu \
# install openssl and libssl dev libs
    openssl \
    libssl-dev \
  ; \
  rm -rf /var/lib/apt/lists/*; \
# verify that the "gosu" binary works
  gosu nobody true; \
# verify openssl
  openssl version

ENV OTP_VERSION 22.3.2
ENV OTP_SOURCE_SHA256="4a3719c71a7998e4f57e73920439b4b1606f7c045e437a0f0f9f1613594d3eaa"

# Install dependencies required to build Erlang/OTP from source
# http://erlang.org/doc/installation_guide/INSTALL.html
# autoconf: Required to configure Erlang/OTP before compiling
# dpkg-dev: Required to set up host & build type when compiling Erlang/OTP
# libncurses5-dev: Required for Erlang/OTP new shell & observer_cli - https://github.com/zhongwencool/observer_cli
RUN set -eux; \
  \
  savedAptMark="$(apt-mark showmanual)"; \
  apt-get update; \
  apt-get install --yes --no-install-recommends \
    autoconf \
    ca-certificates \
    dirmngr \
    dpkg-dev \
    gcc \
    libncurses5-dev \
    m4 \
    make \
    wget \
  ; \
  rm -rf /var/lib/apt/lists/*; \
  \
  openssl version; \
  \
  OTP_SOURCE_URL="https://github.com/erlang/otp/archive/OTP-$OTP_VERSION.tar.gz"; \
  OTP_PATH="/usr/local/src/otp-$OTP_VERSION"; \
  \
# Download, verify & extract OTP_SOURCE
  mkdir -p "$OTP_PATH"; \
  wget --progress dot:giga --output-document "$OTP_PATH.tar.gz" "$OTP_SOURCE_URL"; \
  echo "$OTP_SOURCE_SHA256 *$OTP_PATH.tar.gz" | sha256sum --check --strict -; \
  tar --extract --file "$OTP_PATH.tar.gz" --directory "$OTP_PATH" --strip-components 1; \
  \
# Configure Erlang/OTP for compilation, disable unused features & applications
# http://erlang.org/doc/applications.html
# ERL_TOP is required for Erlang/OTP makefiles to find the absolute path for the installation
  cd "$OTP_PATH"; \
  export ERL_TOP="$OTP_PATH"; \
  ./otp_build autoconf; \
  CFLAGS="$(dpkg-buildflags --get CFLAGS)"; export CFLAGS; \
# add -rpath to avoid conflicts between our OpenSSL's "libssl.so" and the libssl package by making sure /usr/local/lib is searched first (but only for Erlang/OpenSSL to avoid issues with other tools using libssl; https://github.com/docker-library/rabbitmq/issues/364)
  export CFLAGS="$CFLAGS -Wl,-rpath=/usr/lib/ssl"; \
  hostArch="$(dpkg-architecture --query DEB_HOST_GNU_TYPE)"; \
  buildArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
  dpkgArch="$(dpkg --print-architecture)"; dpkgArch="${dpkgArch##*-}"; \
  ./configure \
    --host="$hostArch" \
    --build="$buildArch" \
    --disable-dynamic-ssl-lib \
    --enable-hipe \
    --disable-sctp \
    --disable-silent-rules \
    --enable-clock-gettime \
    --enable-hybrid-heap \
    --enable-kernel-poll \
    --enable-shared-zlib \
    --enable-smp-support \
    --enable-threads \
    --with-microstate-accounting=extra \
    --with-ssl="/usr/include/openssl" \
    --without-common_test \
    --without-debugger \
    --without-dialyzer \
    --without-diameter \
    --without-edoc \
    --without-erl_docgen \
    --without-erl_interface \
    --without-et \
    --without-eunit \
    --without-ftp \
    --without-hipe \
    --without-jinterface \
    --without-megaco \
    --without-observer \
    --without-odbc \
    --without-reltool \
    --without-ssh \
    --without-tftp \
    --without-wx \
  ; \
# Compile & install Erlang/OTP
  make -j "$(getconf _NPROCESSORS_ONLN)" GEN_OPT_FLGS="-O2 -fno-strict-aliasing"; \
  make install; \
  cd ..; \
  rm -rf \
    "$OTP_PATH"* \
    /usr/local/lib/erlang/lib/*/examples \
    /usr/local/lib/erlang/lib/*/src \
  ; \
  \
# Check that Erlang/OTP crypto & ssl were compiled against OpenSSL correctly
  erl -noshell -eval 'io:format("~p~n~n~p~n~n", [crypto:supports(), ssl:versions()]), init:stop().'

ENV RABBITMQ_DATA_DIR=/var/lib/rabbitmq
# Create rabbitmq system user & group, fix permissions & allow root user to connect to the RabbitMQ Erlang VM
RUN set -eux; \
  groupadd --gid 999 --system rabbitmq; \
  useradd --uid 999 --system --home-dir "$RABBITMQ_DATA_DIR" --gid rabbitmq rabbitmq; \
  mkdir -p "$RABBITMQ_DATA_DIR" /etc/rabbitmq /tmp/rabbitmq-ssl /var/log/rabbitmq; \
  chown -fR rabbitmq:rabbitmq "$RABBITMQ_DATA_DIR" /etc/rabbitmq /tmp/rabbitmq-ssl /var/log/rabbitmq; \
  chmod 777 "$RABBITMQ_DATA_DIR" /etc/rabbitmq /tmp/rabbitmq-ssl /var/log/rabbitmq; \
  ln -sf "$RABBITMQ_DATA_DIR/.erlang.cookie" /root/.erlang.cookie


ENV RABBITMQ_VERSION 3.7.25
ENV RABBITMQ_SHA512="365ba90b11209c75a27ce39303330ef45333496e756ec510c4dda90f8e1e3ce444d8c1948ea3181966b2871430675c650bd1a8474b7a1f5fa49e3c68cb8832cd"
ENV RABBITMQ_HOME=/opt/rabbitmq

# Add RabbitMQ to PATH, send all logs to TTY
ENV PATH=$RABBITMQ_HOME/sbin:$PATH \
  RABBITMQ_LOGS=-

# Install RabbitMQ
RUN set -eux; \
  \
  savedAptMark="$(apt-mark showmanual)"; \
  apt-get update; \
  apt-get install --yes --no-install-recommends \
    ca-certificates \
    dirmngr \
    gnupg2 \
    wget \
    xz-utils \
  ; \
  rm -rf /var/lib/apt/lists/*; \
  \
  RABBITMQ_SOURCE_URL="https://github.com/rabbitmq/rabbitmq-server/releases/download/v$RABBITMQ_VERSION/rabbitmq-server-generic-unix-latest-toolchain-$RABBITMQ_VERSION.tar.xz"; \
  RABBITMQ_PATH="/usr/local/src/rabbitmq-$RABBITMQ_VERSION"; \
  \
  wget --progress dot:giga --output-document "$RABBITMQ_PATH.tar.xz" "$RABBITMQ_SOURCE_URL"; \
  \
  mkdir -p "$RABBITMQ_HOME"; \
  echo "$RABBITMQ_SHA512 *$RABBITMQ_PATH.tar.xz" | sha512sum --check --strict -; \
  tar --extract --file "$RABBITMQ_PATH.tar.xz" --directory "$RABBITMQ_HOME" --strip-components 1; \
  rm -rf "$RABBITMQ_PATH"*; \
# Do not default SYS_PREFIX to RABBITMQ_HOME, leave it empty
  grep -qE '^SYS_PREFIX=\$\{RABBITMQ_HOME\}$' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
  sed -i 's/^SYS_PREFIX=.*$/SYS_PREFIX=/' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
  grep -qE '^SYS_PREFIX=$' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
  chown -R rabbitmq:rabbitmq "$RABBITMQ_HOME"; \
  \
  # reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
  apt-mark auto '.*' > /dev/null; \
  [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
  find /usr/local -type f -executable -exec ldd '{}' ';' \
    | awk '/=>/ { print $(NF-1) }' \
    | sort -u \
    | xargs -r dpkg-query --search \
    | cut -d: -f1 \
    | sort -u \
    | xargs -r apt-mark manual \
  ; \
  apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
# verify assumption of no stale cookies
  [ ! -e "$RABBITMQ_DATA_DIR/.erlang.cookie" ]

RUN apt update && apt install -y openssl

RUN mkdir /usr/share/doc/rabbitmq-plugins

COPY --from=plugins --chown=rabbitmq /plugins/* /opt/rabbitmq/plugins/
COPY --from=plugins --chown=rabbitmq /licenses/* /usr/share/doc/rabbitmq-plugins/

# Ensure RabbitMQ was installed correctly by running a few commands that do not depend on a running server, as the rabbitmq user
# If they all succeed, it's safe to assume that things have been set up correctly
RUN	gosu rabbitmq rabbitmqctl help; \
  gosu rabbitmq rabbitmqctl list_ciphers; \
  gosu rabbitmq rabbitmq-plugins list; \
# no stale cookies
  rm "$RABBITMQ_DATA_DIR/.erlang.cookie"

# Added for backwards compatibility - users can simply COPY custom plugins to /plugins
RUN ln -sf /opt/rabbitmq/plugins /plugins

# set home so that any `--user` knows where to put the erlang cookie
ENV HOME $RABBITMQ_DATA_DIR
# Hint that the data (a.k.a. home dir) dir should be separate volume
VOLUME $RABBITMQ_DATA_DIR

# warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
# Setting all environment variables that control language preferences, behaviour differs - https://www.gnu.org/software/gettext/manual/html_node/The-LANGUAGE-variable.html#The-LANGUAGE-variable
# https://docs.docker.com/samples/library/ubuntu/#locales
ENV LANG=C.UTF-8 LANGUAGE=C.UTF-8 LC_ALL=C.UTF-8

COPY --chown=rabbitmq:rabbitmq docker-entrypoint.sh /usr/local/bin/

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 4369 5671 5672 25672
CMD ["rabbitmq-server"]
