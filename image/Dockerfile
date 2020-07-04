#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
FROM debian:buster-slim

COPY base-image /container
RUN /container/build.sh

ARG AIRFLOW_OPENLDAP_VERSION
ARG OPENLDAP_VERSION
ARG COMMIT_SHA

ARG LDAP_OPENLDAP_GID
ARG LDAP_OPENLDAP_UID

ARG PQCHECKER_VERSION=2.0.0
ARG PQCHECKER_MD5=c005ce596e97d13e39485e711dcbc7e1

MAINTAINER "Apache Airflow Community <dev@airflow.apache.org>"

ENV LANG="en_US.UTF-8" \
    LANGUAGE="en_US:en" \
    LC_ALL="en_US.UTF-8"


# Add openldap user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
# If explicit uid or gid is given, use it.
RUN if [ -z "${LDAP_OPENLDAP_GID}" ]; then groupadd -g 911 -r openldap; else groupadd -r -g ${LDAP_OPENLDAP_GID} openldap; fi \
    && if [ -z "${LDAP_OPENLDAP_UID}" ]; then useradd -u 911 -r -g openldap openldap; else useradd -r -g openldap -u ${LDAP_OPENLDAP_UID} openldap; fi

# Add buster-backports in preparation for downloading newer openldap components, especially sladp
RUN echo "deb http://ftp.debian.org/debian buster-backports main" >> /etc/apt/sources.list

# Install OpenLDAP, ldap-utils and ssl-tools from the (backported) baseimage and clean apt-get files
# sources: https://github.com/osixia/docker-light-baseimage/blob/stable/image/tool/add-service-available
#          https://github.com/osixia/docker-light-baseimage/blob/stable/image/service-available/:ssl-tools/download.sh
RUN echo "path-include /usr/share/doc/krb5*" >> /etc/dpkg/dpkg.cfg.d/docker && apt-get -y update \
    && /container/tool/add-service-available :ssl-tools \
    && LC_ALL=C DEBIAN_FRONTEND=noninteractive apt-get -t buster-backports install -y --no-install-recommends \
    ca-certificates \
    curl \
    ldap-utils \
    libsasl2-modules \
    libsasl2-modules-db \
    libsasl2-modules-gssapi-mit \
    libsasl2-modules-ldap \
    libsasl2-modules-otp \
    libsasl2-modules-sql \
    openssl \
    slapd \
    slapd-contrib \
    krb5-kdc-ldap \
    && curl -o pqchecker.deb -SL http://www.meddeb.net/pub/pqchecker/deb/8/pqchecker_${PQCHECKER_VERSION}_amd64.deb \
    && echo "${PQCHECKER_MD5} *pqchecker.deb" | md5sum -c - \
    && dpkg -i pqchecker.deb \
    && rm pqchecker.deb \
    && update-ca-certificates \
    && apt-get remove -y --purge --auto-remove curl ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Add service directory to /container/service
ADD service /container/service

# Use baseimage install-service script
# https://github.com/osixia/docker-light-baseimage/blob/stable/image/tool/install-service
RUN /container/tool/install-service

# Add default env variables
ADD environment /container/environment/99-default

# Expose default ldap and ldaps ports
EXPOSE 389 636

LABEL org.apache.airflow.component="openldap"
LABEL org.apache.airflow.openldap.version="${OPENLDAP_VERSION}"
LABEL org.apache.airflow.airflow_openldap.version="${AIRFLOW_OPENLDAP_VERSION}"
LABEL org.apache.airflow.commit_sha="${COMMIT_SHA}"

ENTRYPOINT ["/container/tool/run"]

# Put ldap config and database dir in a volume to persist data.
# VOLUME /etc/ldap/slapd.d /var/lib/ldap
