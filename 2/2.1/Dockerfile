FROM gcr.io/google-appengine/debian8:latest

# Get the list of packages from the base image.
COPY download_source /tmp
RUN /tmp/download_source blacklist-all-current

# explicitly set user/group IDs
RUN groupadd -r cassandra --gid=999 && useradd -r -g cassandra --uid=999 cassandra

# grab gosu for easy step-down from root
ENV GOSU_VERSION 1.10
RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& wget -q -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -q -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
# copy source code
	&& wget -q -O /usr/local/src/gosu.tar.gz "https://github.com/tianon/gosu/archive/$GOSU_VERSION.tar.gz" \
# extract gosu binary and check signature
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver pgp.mit.edu --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true \
	&& apt-get purge -y --auto-remove ca-certificates wget

# solves warning: "jemalloc shared library could not be preloaded to speed up memory allocations"
RUN apt-get update && apt-get install -y --no-install-recommends libjemalloc1 && rm -rf /var/lib/apt/lists/*

# https://wiki.apache.org/cassandra/DebianPackaging#Adding_Repository_Keys
ENV GPG_KEYS 514A2AD631A57A16DD0047EC749D6EEC0353B12C A26E528B271F19B9E5D8E19EA278B781FE4B2BDA

RUN set -ex; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		gpg --keyserver pgp.mit.edu --recv-keys "$key"; \
	done; \
	gpg --export $GPG_KEYS > /etc/apt/trusted.gpg.d/cassandra.gpg; \
	rm -r "$GNUPGHOME"; \
	apt-key list

RUN echo 'deb http://www.apache.org/dist/cassandra/debian 21x main' >> /etc/apt/sources.list.d/cassandra.list

RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/* \
	&& curl -s http://dl.bintray.com/apache/cassandra/dists/21x/main/binary-amd64/Packages | awk '/Version/ { print "export CASSANDRA_FULLVERSION=" $2 }' > /tmp/cassandra-version \
	&& apt-get purge -y --auto-remove curl


ENV CASSANDRA_VERSION 21

RUN set -x; \
	. /tmp/cassandra-version \
	&& if [ "${CASSANDRA_FULLVERSION%%.*}" = "3" ]; then \
		echo deb http://httpredir.debian.org/debian jessie-backports main > /etc/apt/sources.list.d/backports.list ; \
		apt-get update && apt-get upgrade; \
		apt-get -t jessie-backports install -y openjdk-8-jre-headless; \
		apt-get install -y ntp; \
	fi

RUN apt-get update \
	&& . /tmp/cassandra-version \
	&& apt-get install -y cassandra="$CASSANDRA_FULLVERSION" \
	&& rm -rf /var/lib/apt/lists/*

RUN rm /tmp/cassandra-version

# https://issues.apache.org/jira/browse/CASSANDRA-11661
RUN sed -ri 's/^(JVM_PATCH_VERSION)=.*/\1=25/' /etc/cassandra/cassandra-env.sh

ENV CASSANDRA_CONFIG /etc/cassandra

# download source code for GPL packages
RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& BIND9_VER=$(apt-cache policy libdns-export100 | grep "Installed:" | sed 's/.*Installed: 1:\(.*\.dfsg\).*/\1/') \
	&& wget -O bind.tar.gz "http://security.debian.org/debian-security/pool/updates/main/b/bind9/bind9_${BIND9_VER}.orig.tar.gz" \
	&& tar -xvf bind.tar.gz -C /usr/local/src \
	&& rm bind.tar.gz \
	&& apt-get purge -y --auto-remove wget

COPY docker-entrypoint.sh /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]

RUN mkdir -p /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chown -R cassandra:cassandra /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chmod 777 /var/lib/cassandra "$CASSANDRA_CONFIG"
VOLUME /var/lib/cassandra

# 7000: intra-node communication
# 7001: TLS intra-node communication
# 7199: JMX
# 9042: CQL
# 9160: thrift service
EXPOSE 7000 7001 7199 9042 9160
CMD ["cassandra", "-f"]

# Download source code for GPL packages.
RUN /tmp/download_source download /usr/local/src \
	&& rm /tmp/download_source
