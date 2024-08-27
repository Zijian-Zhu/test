---
layout: global
title: Deploy Alluxio Edge for Trino with Kubernetes
---

Kubernetes is supported in the major ecosystems such as Amazon AWS EKS. To
deploy Alluxio Edge for Trino with Kubernetes, the key step is to add Alluxio Edge
jar to the Trino image, update a few configurations and the helm chart, then proceed with
the normal deployment of Trino. This document walks through these steps with AWS S3 as UFS
for example.

## Package Alluxio Edge with Trino

To use Alluxio Edge with Trino, we will add several Alluxio Edge JARs to the Trino image via the Dockerfile and then
rebuild the image. Afterward, we'll follow the same steps as those for deploying Trino itself.

### Prerequisites

- The `docker` command line tool
- The `tar` command line tool
- Your AWS S3 credentials
- Note down the installation directory of Trino we will refer to it as `${TRINO_HOME}`
- Preparation of storage for Alluxio Edge: After identifying the storage attached to the Kubernetes Trino container
  for the Alluxio Edge cache, please note:
  - The size of the local storage to be provisioned for Alluxio Edge. It will be needed for variable
    `alluxio.user.client.cache.size` in the configuration step.
  - The path where it is mounted inside the container. It will be needed for variable
    `alluxio.user.client.cache.dirs` in the configuration step.

Request a trial version of Alluxio Edge. Contact your Alluxio account representative at `sales@alluxio.com` to request
a trial version of Alluxio Edge. Follow their instructions to download the installation tar file into the directory you prepared.

The tar file follows the naming convention `alluxio-enterprise-*.tar.gz`. For example, if the tarball is named
`alluxio-enterprise-3.x-4.0.0-bin-97a746cf8c.tar.gz`, the alluxio version is `3.x-4.0.0`.

### Prepare the necessary Jar

#### Extract Alluxio Edge jars

Two Alluxio Edge Java JAR files must be installed on each Trino node.

First, extract the Alluxio Edge client JAR file from the tar file using this command:

```console
$ tar xf alluxio-enterprise-*.tar.gz alluxio-enterprise-*/client/alluxio-emon-*-client.jar
```

Then, extract the Alluxio Edge S3 under store filesystem integration JAR file using this command:

```console
$ tar xf alluxio-enterprise-*.tar.gz alluxio-enterprise-*/lib/alluxio-underfs-emon-s3a-*.jar
```

Create a subdirectory called `jars` in the project's root directory and move all the JARs into `./jars`.

```console
$ mkdir -p jars
$ cp alluxio-enterprise-*/*/*.jar ./jars/
```

#### Download the Prometheus jar

[Prometheus](https://github.com) is the recommended database for observing the metrics that Alluxio Edge emits.
If the JMX exporter for Prometheus is not already set up in your environment, you can download the Java agent
JAR from https://github.com/prometheus/jmx_exporter/releases .
The file is named similarly to `jmx_prometheus_javaagent-0.20.0.jar`. Place it in the `jars` directory.

### Create property file to enable metrics system

In the project root directory create a file named `metrics.properties` with the following contents

```
sink.jmx.class=alluxio.metrics.sink.JmxSink
```

### (Option) Create cache filter policy definition file to enable cache filter

This is an advanced feature. Please see [Cache Filter configuration instruction]({{ '/en/administration/ConfiguringCacheFilter.html' | relativize_url }}).
For this example, we will put the content in `cache_filter.properties`.

### Prepare a Dockerfile file

In the project directory, create a file named `Dockerfile` with the following contents, setting the corresponding values for:

The following `Dockerfile` is an example:

```shell
# Identify the Trino image. It can be from the Trino official website or custom build
ARG IMAGE=trinodb/trino:430
FROM $IMAGE
# Identify the Trino installation path
ARG TRINO_HOME=/user/lib/trino 

# Remove old versions of Alluxio jar files from the container
RUN find ${TRINO_HOME} -name alluxio*shaded* -exec rm {} \;

# pre-req: extract jars and put under jars/
# Copy the Alluxio Edge client jar file to the Trino catalog dirs
COPY jars/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/hive
COPY jars/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/hudi
COPY jars/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/delta-lake
COPY jars/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/iceberg

# Make a home directory for Alluxio Edge
RUN mkdir -p /home/trino/alluxio/lib && mkdir -p /home/trino/alluxio/conf

# Copy the Alluxio Edge under store jar file to the Trino lib dir
COPY jars/alluxio-underfs-emon-s3a-*.jar /home/trino/alluxio/lib

# Copy the metrics property file that enables the Alluxio JmxSink
COPY metrics.properties /home/trino/alluxio/conf/metrics.properties

# Copy the JVX Prometheus agent jar file to the Alluxio lib dir
COPY jars/jmx_prometheus_javaagent-*.jar /home/trino/alluxio/

# (Optional) Enable cache filter 
COPY cache_filter.properties /home/trino/alluxio/conf/cache_filter.properties

USER trino:trino
```

### Build the updated image

At this point, the project root directory prepared in step 1 should have the following files

* `Dockerfile`
* `metrics.properties`
* (Optional) `cache_filter.properties`
* `jars/`
  * Alluxio Edge client jar
  * Alluxio Edge S3 understorage jar
  * Prometheus java agent jar

Run `docker build -t <repository>:<tag> . ` to build Alluxio Edge with Trino.

The repository naming convention is `<path>/<image_name>`. For example:

```console
$ docker build -t mytrino/trino-alluxio-edge:1.0.0 .
```

Please keep a note of the image info `<repository>` and `<tag>`, we will use it later.

### Push the new image to a private image registry

For the cluster to fetch the locally built docker image, it will need to be pushed to a
private image registry (ex. AWS ECR) where the servers have access to fetch images from.

## Update the Trino Helm Chart configuration yaml file

To override the default values of Trino Kubernetes configuration, one can use a YAML
configuration file to define the parameters of your deployment.

If you already have a Trino configuration YAML file, update the corresponding sections
below. If you don’t already have a Trino configuration YAML file, create a new file.
Let’s refer to it as `values_custom.yaml`.

### Update image

We need to update the image path in order to use the new image we built before.
Here Repository and tag corresponding to the one we built. In our example it
looks like this.

```yaml
image:
  repository: mytrino/trino-alluxio-edge
  tag: 1.0.0
```

### Update Trino worker and coordinator

In this step we will update the configuration for the worker and coordinator. 

Please refer to
[configuration settings]({{ '/en/reference/Configuration.html' | relativize_url }}) for full list of properties
to configure for Alluxio Edge in `alluxio-site.properties` and explanation. See below for template.

```yaml
worker:
  additionalConfigFiles:

    core-site.xml: |
      <configuration>
      <property>
        <name>fs.s3.impl</name>
        <value>alluxio.emon.hadoop.FileSystemEE</value>
      </property>
      <property>
        <name>fs.s3a.impl</name>
        <value>alluxio.emon.hadoop.FileSystemEE</value>
      </property>
      </configuration>


    alluxio-site.properties: |
      #
      # Alluxio under file system setup (S3)
      #
      s3a.accessKeyId=<PUT_YOUR_AWS_ACCESS_KEY_ID_HERE>
      s3a.secretKey=<PUT_YOUR_AWS_SECRET_KEY_HERE>
      #
      # Enable Alluxio Edge cache on Trino nodes (example with 2 SSD volumes)
      #
      alluxio.user.client.cache.enabled=true
      alluxio.user.client.cache.size=<PUT_YOUR_SSD1_VOL_SIZE_HERE>GB,<PUT_YOUR_SSD2_VOL_SIZE_HERE>GB
      alluxio.user.client.cache.dirs=<PUT_YOUR_SSD1_VOL_MOUNT_POINT_HERE>/alluxio_cache,/<PUT_YOUR_SSD2_VOL_MOUNT_POINT_HERE>/alluxio_cache
      #
      # Enable edge metrics collection
      alluxio.user.metrics.collection.enabled=true
      #
      # Disable DORA
      alluxio.dora.enabled=false
      #


    jmx_export_config.yaml: |
      ---
      startDelaySeconds: 0
      ssl: false
      global:
        scrape_interval:     15s
        evaluation_interval: 15s
      rules:
      - pattern: ".*"

  additionalJVMConfig:
    - "-XX:+UnlockDiagnosticVMOptions"
    - "-XX:+UseAESCTRIntrinsics"
    # Add following two lines to Enable Alluxio Edge integration
    - "-Dalluxio.home=/home/trino/alluxio"
    - "-Dalluxio.conf.dir=/etc/trino/"
    # Add following two lines to enable Alluxio Edge JMX/Prometheus export support
    -Dalluxio.metrics.conf.file=/home/trino/alluxio/conf/metrics.properties
    -javaagent:/home/trino/alluxio/lib/jmx_prometheus_javaagent-0.20.0.jar=9696:/etc/trino/jmx_export_config.yaml


coordinator:
  additionalConfigFiles:

    core-site.xml: |
      <configuration>
      <property>
        <name>fs.s3.impl</name>
        <value>alluxio.emon.hadoop.FileSystemEE</value>
      </property>
      <property>
        <name>fs.s3a.impl</name>
        <value>alluxio.emon.hadoop.FileSystemEE</value>
      </property>
      </configuration>


    alluxio-site.properties: |
      #
      # Alluxio under file system setup (S3)
      #
      s3a.accessKeyId=<PUT_YOUR_AWS_ACCESS_KEY_ID_HERE>
      s3a.secretKey=<PUT_YOUR_AWS_SECRET_KEY_HERE>
      #
      # Enable Alluxio Edge cache on Trino nodes (example with 2 SSD volumes)
      #
      alluxio.user.client.cache.enabled=true
      alluxio.user.client.cache.size=<PUT_YOUR_SSD1_VOL_SIZE_HERE>GB,<PUT_YOUR_SSD2_VOL_SIZE_HERE>GB
      alluxio.user.client.cache.dirs=<PUT_YOUR_SSD1_VOL_MOUNT_POINT_HERE>/alluxio_cache,/<PUT_YOUR_SSD2_VOL_MOUNT_POINT_HERE>/alluxio_cache
      #
      # Enable edge metrics collection
      alluxio.user.metrics.collection.enabled=true
      #
      # Disable DORA
      alluxio.dora.enabled=false
      #
      # soft affinity schedule (optional)
      #

    jmx_export_config.yaml: |
      ---
      startDelaySeconds: 0
      ssl: false
      global:
        scrape_interval:     15s
        evaluation_interval: 15s
      rules:
      - pattern: ".*"

	additionalJVMConfig:
	  - "-XX:+UnlockDiagnosticVMOptions"
	  - "-XX:+UseAESCTRIntrinsics"
	  # Add following two lines to Enable Alluxio Edge integration
	  - "-Dalluxio.home=/home/trino/alluxio"
	  - "-Dalluxio.conf.dir=/etc/trino/"
	  # Add following two lines to enable Alluxio Edge JMX/Prometheus export support
	  -Dalluxio.metrics.conf.file=/home/trino/alluxio/conf/metrics.properties
	  -javaagent:/home/trino/alluxio/lib/jmx_prometheus_javaagent-0.20.0.jar=9696:/etc/trino/jmx_export_config.yaml
```

### Update catalogs

Copy the corresponding configuration for the catalog you are using from the section below.

```yaml
additionalCatalogs:
  hive: |-
    connector.name=hive
    hive.metastore.uri=thrift://hive-metastore:9083
    hive.s3-file-system-type=HADOOP_DEFAULT
    hive.config.resources=/home/trino/alluxio/conf/core-site.xml
  iceberg: |-
    connector.name=iceberg
    iceberg.catalog.type=hive_metastore
    iceberg.file-format=PARQUET
    hive.metastore.uri=thrift://hive-metastore:9083
    hive.s3-file-system-type=HADOOP_DEFAULT
    hive.config.resources=/home/trino/alluxio/conf/core-site.xml
  deltalake: |-
    connector.name=delta_lake
    delta.hive-catalog-name=deltalake
    delta.enable-non-concurrent-writes=true
    delta.vacuum.min-retention=1h
    delta.register-table-procedure.enabled=true
    delta.extended-statistics.collect-on-write=false
    hive.metastore.uri=thrift://hive-metastore:9083
    hive.s3-file-system-type=HADOOP_DEFAULT
    hive.config.resources=/home/trino/alluxio/conf/core-site.xml
```

Some optional configurations

```yaml
    hive.non-managed-table-writes-enabled=true
    hive.s3select-pushdown.enabled=true
    hive.storage-format=PARQUET
    hive.allow-drop-table=true
```

## Use helm chart to deploy Trino with Kubernetes

Now you can follow [Trino on Kubernetes with Helm](https://trino.io/docs/current/installation/kubernetes.html)
instruction by Trino with the new image. All the steps are the same, except that when we do helm install we
will use the Trino configuration YAML file in section 2. In the example of Trino document example
`helm install -f example.yaml example-trino-cluster trino/trino`, we will replace `example.yaml` with `values_custom.yaml`.
