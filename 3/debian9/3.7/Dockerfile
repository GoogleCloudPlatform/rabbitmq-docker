FROM gcr.io/google-appengine/debian9:latest AS plugins

COPY license-checksums /

RUN set -eux; \
  apt-get update && apt-get install -y ca-certificates wget; \
  \
  mkdir /plugins && cd /plugins; \
  \
  readonly base_url='https://github.com/deadtrickster/prometheus_rabbitmq_exporter/releases/download/v3.7.2.5'; \
  wget "${base_url}/accept-0.3.5.ez"; \
  wget "${base_url}/prometheus-4.2.2.ez"; \
  wget "${base_url}/prometheus_cowboy-0.1.7.ez"; \
  wget "${base_url}/prometheus_httpd-2.1.10.ez"; \
  wget "${base_url}/prometheus_rabbitmq_exporter-3.7.2.5.ez"; \
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

FROM gcr.io/google-appengine/debian9:latest

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		gnupg \
		dirmngr \
	; \
	rm -rf /var/lib/apt/lists/*

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r rabbitmq && useradd -r -d /var/lib/rabbitmq -m -g rabbitmq rabbitmq

# grab gosu for easy step-down from root
ENV GOSU_VERSION 1.10
ENV GOSU_GPG B42F6819007F00F88E364FD4036A9C25BF357DD4
RUN set -eux; \
	\
	fetchDeps=' \
		ca-certificates \
		wget \
	'; \
	apt-get update; \
	apt-get install -y --no-install-recommends $fetchDeps; \
	rm -rf /var/lib/apt/lists/*; \
	\
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
	\
# copy source code
	wget -O /usr/local/src/gosu.tar.gz "https://github.com/tianon/gosu/archive/$GOSU_VERSION.tar.gz"; \
	\
# verify the signature
	export GNUPGHOME="$(mktemp -d)"; \
	found='' && \
	for server in \
		pool.sks-keyservers.net \
		na.pool.sks-keyservers.net \
		eu.pool.sks-keyservers.net \
		oc.pool.sks-keyservers.net \
		ha.pool.sks-keyservers.net \
		hkp://p80.pool.sks-keyservers.net:80 \
		hkp://keyserver.ubuntu.com:80 \
		pgp.mit.edu \
	; do \
		gpg --no-tty --keyserver $server --recv-keys $GOSU_GPG \
			&& found=yes && break; \
	done; \
	test -n "$found"; \
	gpg --no-tty --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	\
	chmod +x /usr/local/bin/gosu; \
# verify that the binary works
	gosu nobody true; \
	\
	apt-get purge -y --auto-remove $fetchDeps
# RabbitMQ 3.6.15+ requires Erlang 19.3+ (and Stretch only has 19.2); https://www.rabbitmq.com/which-erlang.html
# so we'll pull Erlang from Buster instead (not using Erlang Solutions since their multiarch support is extremely limited)
RUN set -eux; \
# add buster sources.list
	sed 's/stretch/buster/g' /etc/apt/sources.list \
		| tee /etc/apt/sources.list.d/buster.list; \
# update apt-preferences such that we get only erlang* from buster (and erlang* only from buster)
	{ \
		echo 'Package: *'; \
		echo 'Pin: release n=buster*'; \
		echo 'Pin-Priority: 1'; \
		echo; \
		echo 'Package: erlang*'; \
		echo 'Pin: release n=buster*'; \
		echo 'Pin-Priority: 999'; \
		echo; \
		echo 'Package: erlang*'; \
		echo 'Pin: release n=stretch*'; \
		echo 'Pin-Priority: -10'; \
	} | tee /etc/apt/preferences.d/buster-erlang
RUN echo 'deb http://http.us.debian.org/debian testing main' >> /etc/apt/sources.list && \
	apt-get update && apt-get install -t testing -y openssl

# install Erlang
RUN set -eux; \
	apt-get update; \
# "erlang-base-hipe" is optional (and only supported on a few arches)
# so, only install it if it's available for our current arch
	if apt-cache show erlang-base-hipe 2>/dev/null | grep -q 'Package: erlang-base-hipe'; then \
		apt-get install -y --no-install-recommends \
			erlang-base-hipe \
		; \
	fi; \
# we start with "erlang-base-hipe" because it and "erlang-base" (non-hipe) are exclusive
	apt-get install -y --no-install-recommends \
		erlang-asn1 \
		erlang-crypto \
		erlang-eldap \
		erlang-inets \
		erlang-mnesia \
		erlang-nox \
		erlang-os-mon \
		erlang-public-key \
		erlang-ssl \
		erlang-xmerl \
	; \
	rm -rf /var/lib/apt/lists/*

# get logs to stdout (thanks @dumbbell for pushing this upstream! :D)
ENV RABBITMQ_LOGS=- RABBITMQ_SASL_LOGS=-
# https://github.com/rabbitmq/rabbitmq-server/commit/53af45bf9a162dec849407d114041aad3d84feaf

# /usr/sbin/rabbitmq-server has some irritating behavior, and only exists to "su - rabbitmq /usr/lib/rabbitmq/bin/rabbitmq-server ..."
ENV PATH /usr/lib/rabbitmq/bin:$PATH

# gpg: key 6026DFCA: public key "RabbitMQ Release Signing Key <info@rabbitmq.com>" imported
ENV RABBITMQ_GPG_KEY 0A9AF2115F4687BD29803A206B73A36E6026DFCA

ENV RABBITMQ_VERSION 3.7.13
ENV RABBITMQ_GITHUB_TAG v3.7.9
ENV RABBITMQ_DEBIAN_VERSION 3.7.9-1

RUN set -eux; \
	\
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates wget; \
	\
	wget -O rabbitmq-server.deb.asc "https://github.com/rabbitmq/rabbitmq-server/releases/download/$RABBITMQ_GITHUB_TAG/rabbitmq-server_${RABBITMQ_DEBIAN_VERSION}_all.deb.asc"; \
	wget -O rabbitmq-server.deb     "https://github.com/rabbitmq/rabbitmq-server/releases/download/$RABBITMQ_GITHUB_TAG/rabbitmq-server_${RABBITMQ_DEBIAN_VERSION}_all.deb"; \
	\
	apt-get purge -y --auto-remove ca-certificates wget; \
	\
	export GNUPGHOME="$(mktemp -d)"; \
	found='' && \
	for server in \
		pool.sks-keyservers.net \
		na.pool.sks-keyservers.net \
		eu.pool.sks-keyservers.net \
		oc.pool.sks-keyservers.net \
		ha.pool.sks-keyservers.net \
		hkp://p80.pool.sks-keyservers.net:80 \
		hkp://keyserver.ubuntu.com:80 \
		pgp.mit.edu \
	; do \
		gpg --no-tty --keyserver $server --recv-keys $RABBITMQ_GPG_KEY \
			&& found=yes && break; \
	done; \
	test -n "$found"; \
	gpg --no-tty --batch --verify rabbitmq-server.deb.asc rabbitmq-server.deb; \
	rm -rf "$GNUPGHOME"; \
	\
	apt install -y --no-install-recommends ./rabbitmq-server.deb; \
	dpkg -l | grep rabbitmq-server; \
	rm -f rabbitmq-server.deb*; \
	\
	rm -rf /var/lib/apt/lists/*
# warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
ENV LANG C.UTF-8

# set home so that any `--user` knows where to put the erlang cookie
ENV HOME /var/lib/rabbitmq

RUN mkdir -p /var/lib/rabbitmq /etc/rabbitmq \
	&& chown -R rabbitmq:rabbitmq /var/lib/rabbitmq /etc/rabbitmq \
	&& chmod -R 777 /var/lib/rabbitmq /etc/rabbitmq
VOLUME /var/lib/rabbitmq

# add a symlink to the .erlang.cookie in /root so we can "docker exec rabbitmqctl ..." without gosu
RUN ln -sf /var/lib/rabbitmq/.erlang.cookie /root/

RUN ln -sf "/usr/lib/rabbitmq/lib/rabbitmq_server-$RABBITMQ_VERSION/plugins" /plugins

# Download RabbitMQ metrics plugins:
RUN \
  mkdir -p /usr/lib/rabbitmq/plugins; \
  mkdir /usr/share/doc/rabbitmq-plugins;

COPY --from=plugins /plugins/* /usr/lib/rabbitmq/plugins/
COPY --from=plugins /licenses/* /usr/share/doc/rabbitmq-plugins/

COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh / # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 4369 5671 5672 25672
CMD ["rabbitmq-server"]
