################################################################################
# Hardened Splunk with Python 3.11 & Updated Ansible Environment
# — plus replacement of all vulnerable Go binaries & a patched hive-exec JAR
################################################################################

FROM splunk/splunk:latest AS hardened
USER root

###############################################################################
# 1) Update OS and install Python 3.11 & pip
###############################################################################
RUN microdnf update -y && \
    microdnf install -y python3.11 python3.11-pip && \
    microdnf clean all

###############################################################################
# 2) Verify Python 3.11
###############################################################################
RUN python3.11 --version && pip3.11 --version

###############################################################################
# 3) Purge old Ansible from system Python (3.9)
###############################################################################
RUN python3 -m pip uninstall -y ansible-core ansible && \
    rm -f /usr/bin/ansible* /usr/local/bin/ansible* || true

###############################################################################
# 4) Prep Ansible temp dirs
###############################################################################
ENV ANSIBLE_REMOTE_TEMP=/opt/container_artifact/ansible_remote \
    ANSIBLE_LOCAL_TEMP=/opt/container_artifact/ansible_local \
    TMPDIR=/opt/container_artifact/tmp

RUN mkdir -p \
      /opt/container_artifact/ansible_remote \
      /opt/container_artifact/ansible_local \
      /opt/container_artifact/tmp && \
    chown -R splunk:splunk /opt/container_artifact

###############################################################################
# 5) Install Ansible under Python 3.11
###############################################################################
RUN python3.11 -m pip install --upgrade "pip>=23.3" wheel && \
    python3.11 -m pip install --no-cache-dir "ansible-core>=2.16.14"

###############################################################################
# 6) Install Collections & Splunk API deps with retry for Galaxy timeouts
###############################################################################
RUN set -eux; \
    for i in 1 2 3; do \
      ansible-galaxy collection install \
        ansible.posix community.general ansible.windows \
        && break; \
      echo "Galaxy timed out, retrying ($i)..."; \
      sleep 10; \
    done; \
    python3.11 -m pip install --no-cache-dir requests-unixsocket

###############################################################################
# 7) Remove old Go binaries shipped by Splunk
###############################################################################
RUN rm -f /opt/splunk/bin/etcd* \
          /opt/splunk/bin/traefik \
          /opt/splunk/bin/mongodump \
          /opt/splunk/bin/mongorestore \
          /opt/splunk/bin/bsondump \
          /opt/splunk/bin/mongo* \
          /opt/splunk/bin/compsup \
          /opt/splunk/bin/spl2-orchestrator || true

###############################################################################
# 8) Install fresh etcd v3.6.1 binaries
###############################################################################
RUN curl -sSL \
      https://github.com/etcd-io/etcd/releases/download/v3.6.1/etcd-v3.6.1-linux-amd64.tar.gz \
    | tar xz -C /tmp etcd-v3.6.1-linux-amd64/etcd \
                etcd-v3.6.1-linux-amd64/etcdctl \
                etcd-v3.6.1-linux-amd64/etcdutl && \
    mv /tmp/etcd-v3.6.1-linux-amd64/etcd* /opt/splunk/bin/ && \
    rm -rf /tmp/etcd-v3.6.1-linux-amd64

###############################################################################
# 9) Install fresh Traefik v2.11.25 binary
###############################################################################
RUN curl -sSL \
      https://github.com/traefik/traefik/releases/download/v2.11.25/traefik_v2.11.25_linux_amd64.tar.gz \
    | tar xz -C /opt/splunk/bin traefik

###############################################################################
# 10) Replace vulnerable hive-exec JAR
###############################################################################
RUN rm -f /opt/splunk/bin/jars/thirdparty/hive_4_0/hive-exec-4.0.0.jar && \
    curl -sSL \
      https://repo1.maven.org/maven2/org/apache/hive/hive-exec/4.0.1/hive-exec-4.0.1.jar \
    -o /opt/splunk/bin/jars/thirdparty/hive_4_0/hive-exec-4.0.1.jar

###############################################################################
# 11) Configure MongoDB repo & install Database Tools
###############################################################################
RUN echo "[mongodb-org-7]" > /etc/yum.repos.d/mongodb-org-7.repo && \
    echo "name=MongoDB 7.0 Repository" >> /etc/yum.repos.d/mongodb-org-7.repo && \
    echo "baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/7.0/x86_64/" >> /etc/yum.repos.d/mongodb-org-7.repo && \
    echo "gpgcheck=1" >> /etc/yum.repos.d/mongodb-org-7.repo && \
    echo "enabled=1" >> /etc/yum.repos.d/mongodb-org-7.repo && \
    echo "gpgkey=https://pgp.mongodb.com/server-7.0.asc" >> /etc/yum.repos.d/mongodb-org-7.repo && \
    microdnf install -y mongodb-database-tools && \
    cp /usr/bin/mongodump /opt/splunk/bin/ && \
    cp /usr/bin/mongorestore /opt/splunk/bin/ && \
    cp /usr/bin/bsondump /opt/splunk/bin/ && \
    microdnf clean all

###############################################################################
# 12) Remove Java/OpenJDK RPMs and bundled JRE
###############################################################################
RUN microdnf remove -y 'java-*' 'openjdk-*' || true && \
    rm -rf /opt/splunk/jre && microdnf clean all

###############################################################################
# 13) Patch OS & strip debug symbols
###############################################################################
RUN microdnf update -y && microdnf clean all && \
    strip --strip-unneeded /opt/splunk/bin/* || true

###############################################################################
# 14) Upgrade system Python (3.9) libraries
###############################################################################
RUN python3 -m pip install --upgrade pip wheel && \
    python3 -m pip install --upgrade --no-cache-dir \
        cryptography>=42.0.2 setuptools>=78.1.1 \
        protobuf>=4.25.8 certifi>=2023.7.22 \
        requests>=2.32.4 urllib3>=2.5.0 \
        mako>=1.2.2 azure-identity>=1.16.1 \
        oauthlib>=3.2.2 zipp>=3.19.1

###############################################################################
# 15) Prepare storage dirs & permissions
###############################################################################
RUN mkdir -p /opt/container_artifact && \
    chown splunk:splunk /opt/container_artifact && \
    mkdir -p /opt/splunk/var/{log,run,lib/splunk/kvstore/mongo} && \
    chown -R splunk:splunk /opt/splunk/var

###############################################################################
# 16) Remove Splunk’s bundled jsdom module
###############################################################################
RUN rm -rf /opt/splunk/lib/node_modules/jsdom*

###############################################################################
# 17) Expose ports & set entrypoint
###############################################################################
EXPOSE 8000 8089 9997
ENTRYPOINT ["/sbin/entrypoint.sh"]
CMD ["start-service"]
