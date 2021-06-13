FROM jupyter/scipy-notebook:python-3.9.4

LABEL maintainer="spark-on-ozone-kerberos <mansu_kim@tmax.co.kr>"
USER root

# ref from https://github.com/jupyter/docker-stacks/blob/master/pyspark-notebook/Dockerfile
# Fix DL4006
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

###########
# openjdk #
###########
ARG OPENJDK_VERSION="11"
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    "openjdk-${OPENJDK_VERSION}-jre-headless" \
    ca-certificates-java && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

#########
# ozone #
#########
## install ozone
ARG OZONE_VERSION="1.1.0"
ARG OZONE_CHECKSUM="76de4f4e5916ad0ac47da0d90429f8e826f553bb0da590273e2c09ec373b9df4b4e8524de8f0aa30ca20eef23c3fb35f11834a4be1b5002d0ff5379bc51af612"

WORKDIR /tmp
RUN wget -q "https://archive.apache.org/dist/ozone/${OZONE_VERSION}/ozone-${OZONE_VERSION}.tar.gz" && \
    echo "${OZONE_CHECKSUM} *ozone-${OZONE_VERSION}.tar.gz" | sha512sum -c - && \
    tar -xzf "ozone-${OZONE_VERSION}.tar.gz" "ozone-${OZONE_VERSION}/share/ozone/lib/hadoop-ozone-filesystem-hadoop2-${OZONE_VERSION}.jar" && \
    mv "ozone-${OZONE_VERSION}/share/ozone/lib/hadoop-ozone-filesystem-hadoop2-${OZONE_VERSION}.jar" /opt && \
    rm "ozone-${OZONE_VERSION}.tar.gz" && \
    rm -r "ozone-${OZONE_VERSION}"

## Configure ozone
### In kubernetes, core-site and ozone-site must be provided based on ozone configMap setting
### Create core-site.xml, ozone-site.xml as configmap and voluemount to /opt/hadoop/conf
#ADD core-site.xml /opt/hadoop/conf/core-site.xml
#ADD ozone-site.xml /opt/hadoop/conf/ozone-site.xml
ENV HADOOP_CONF_DIR=/opt/hadoop/conf
ENV SPARK_EXTRA_CLASSPATH=/opt/hadoop/conf

#########
# spark #
#########
## install spark
ARG SPARK_VERSION="3.1.2"
ARG SPARK_CHECKSUM="7a1affa94d27c9b71d19f443aab0cf2bea3dc921d7319d6c013caf91ba4f8e11182f9c3a80492ba5e058c4b92c5ed0850f6acdcd5d7af19cafed11e18408a45b"

RUN wget -q "https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}.tgz" && \
    echo "${SPARK_CHECKSUM} *spark-${SPARK_VERSION}.tgz" | sha512sum -c - && \
    tar xzf "spark-${SPARK_VERSION}.tgz" -C /usr/local && \
    rm "spark-${SPARK_VERSION}.tgz"

## Configure Spark
ENV SPARK_HOME=/usr/local/spark
### ENV SPARK_OPTS="--driver-java-options=-Xms1024M --driver-java-options=-Xmx4096M --driver-java-options=-Dlog4j.logLevel=info" \
### dynamic options(ex. driver-java-options) must allocated in kubernetes yaml
ENV PATH=$PATH:$SPARK_HOME/bin
### If you want to add additional spark opt, please append env. ozone jar must be used.
ENV SPARK_OPTS="--jars /opt/ozone-${APACHE_OZONE_VERSION}.tar.gz"

# ref from https://github.com/jupyter/docker-stacks/blob/master/all-spark-notebook/Dockerfile
#################
# R for jupyter #
#################
USER root
# RSpark config
ENV R_LIBS_USER="${SPARK_HOME}/R/lib"

# R pre-requisites
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    fonts-dejavu \
    gfortran \
    gcc && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

USER ${NB_UID}
# R packages including IRKernel which gets installed globally.
RUN mamba install --quiet --yes \
    'r-base=4.1.0' \
    'r-ggplot2=3.3*' \
    'r-irkernel=1.2*' \
    'r-rcurl=1.98*' \
    'r-sparklyr=1.7*' && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

#####################
# scala for jupyter #
#####################
# Spylon-kernel
RUN mamba install --quiet --yes 'spylon-kernel=0.4*' && \
    mamba clean --all -f -y && \
    python -m spylon_kernel install --sys-prefix && \
    rm -rf "/home/${NB_USER}/.local" && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

########################
# additional  packages #
########################
USER root
COPY apt-list.txt .
RUN apt-get update && \
    xargs apt-get install < apt-list.txt

USER ${NB_UID}
COPY requirements.txt .
RUN mamba install --quiet --yes --file requirements.txt && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"


WORKDIR "${HOME}"