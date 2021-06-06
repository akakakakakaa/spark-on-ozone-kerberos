ARG BASE_CONTAINER=jupyter/scipy-notebook:python-3.9.2
FROM $BASE_CONTAINER

LABEL maintainer="spark-on-ozone-kerberos <mansu_kim@tmax.co.kr>"

# Fix DL4006
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# installation
## 1. install open jdk
ARG openjdk_version="11"
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    "openjdk-${openjdk_version}-jre-headless" \
    ca-certificates-java && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

## 2. install ozone
ARG ozone_version="1.1.0"
ARG ozone_checksum="76de4f4e5916ad0ac47da0d90429f8e826f553bb0da590273e2c09ec373b9df4b4e8524de8f0aa30ca20eef23c3fb35f11834a4be1b5002d0ff5379bc51af612"
ENV APACHE_OZONE_VERSION="${ozone_version}"

WORKDIR /tmp
RUN wget -q "https://archive.apache.org/dist/ozone/${APACHE_OZONE_VERSION}/ozone-${APACHE_OZONE_VERSION}.tar.gz" && \
    echo "${ozone_checksum} *ozone-${APACHE_OZONE_VERSION}.tar.gz" | sha512sum -c - && \
    tar -xzf "ozone-${APACHE_OZONE_VERSION}.tar.gz" "ozone-${APACHE_OZONE_VERSION}/share/ozone/lib/hadoop-ozone-filesystem-hadoop2-${APACHE_OZONE_VERSION}.jar" && \
    mv "ozone-${APACHE_OZONE_VERSION}/share/ozone/lib/hadoop-ozone-filesystem-hadoop2-${APACHE_OZONE_VERSION}.jar" /opt && \
    chown root:root /opt/hadoop-ozone-filesystem-hadoop2-${APACHE_OZONE_VERSION}.jar && \
    rm "ozone-${APACHE_OZONE_VERSION}.tar.gz" && \
    rm "ozone-${APACHE_OZONE_VERSION}"

## 3. install spark
ARG spark_version="3.1.2"
ARG spark_checksum="7a1affa94d27c9b71d19f443aab0cf2bea3dc921d7319d6c013caf91ba4f8e11182f9c3a80492ba5e058c4b92c5ed0850f6acdcd5d7af19cafed11e18408a45b"
ENV APACHE_SPARK_VERSION="${spark_version}"

RUN wget -q "https://archive.apache.org/dist/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}.tgz" && \
    echo "${spark_checksum} *spark-${APACHE_SPARK_VERSION}.tgz" | sha512sum -c - && \
    tar xzf "spark-${APACHE_SPARK_VERSION}.tgz" -C /usr/local --owner root --group root --no-same-owner && \
    rm "spark-${APACHE_SPARK_VERSION}.tgz"

## 4. install pyarrow
RUN conda install --quiet --yes --satisfied-skip-solve \
    'pyarrow=4.0.*' && \
    conda clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# Set Envs
WORKDIR /usr/local

## 1. Configure Spark
ENV SPARK_HOME=/usr/local/spark
### ENV SPARK_OPTS="--driver-java-options=-Xms1024M --driver-java-options=-Xmx4096M --driver-java-options=-Dlog4j.logLevel=info" \
### dynamic options(ex. driver-java-options) must allocated in kubernetes yaml
ENV SPARK_OPTS="--driver-java-options=-Dlog4j.logLevel=info --jars /opt/ozone-${APACHE_OZONE_VERSION}.tar.gz"
    PATH=$PATH:$SPARK_HOME/bin

RUN ln -s "spark-${APACHE_SPARK_VERSION}" spark && \
    ### Add a link in the before_notebook hook in order to source automatically PYTHONPATH
    mkdir -p /usr/local/bin/before-notebook.d && \
    ln -s "${SPARK_HOME}/sbin/spark-config.sh" /usr/local/bin/before-notebook.d/spark-config.sh

### Fix Spark installation for Java 11 and Apache Arrow library
### see: https://github.com/apache/spark/pull/27356, https://spark.apache.org/docs/latest/#downloading
RUN cp -p "$SPARK_HOME/conf/spark-defaults.conf.template" "$SPARK_HOME/conf/spark-defaults.conf" && \
    echo 'spark.driver.extraJavaOptions -Dio.netty.tryReflectionSetAccessible=true' >> $SPARK_HOME/conf/spark-defaults.conf && \
    echo 'spark.executor.extraJavaOptions -Dio.netty.tryReflectionSetAccessible=true' >> $SPARK_HOME/conf/spark-defaults.conf

USER $NB_UID

## 2. Configure ozone
### In kubernetes, core-site and ozone-site must be provided based on ozone configMap setting
### Create core-site.xml, ozone-site.xml as configmap and voluemount to /opt/hadoop/conf
#ADD core-site.xml /opt/hadoop/conf/core-site.xml
#ADD ozone-site.xml /opt/hadoop/conf/ozone-site.xml
ENV HADOOP_CONF_DIR=/opt/hadoop/conf
ENV SPARK_EXTRA_CLASSPATH=/opt/hadoop/conf

WORKDIR $HOME