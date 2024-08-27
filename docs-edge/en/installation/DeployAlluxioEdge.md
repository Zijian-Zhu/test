---
layout: global
title: Deploy Alluxio Edge for Trino
---

This document describes how to deploy Alluxio Edge for Trino to work with AWS S3 as UFS. The key step is to place Alluxio client jar files
to the Trino path, update a few configurations, then proceed with the normal deployment of Trino.

## Prerequisite

- Your AWS S3 credentials
- Preparation of storage for Alluxio Edge: After identifying the storage mounted for Alluxio Edge cache, please note:
  - The size of the local storage to be provisioned for Alluxio Edge `alluxio.user.client.cache.size`
  - The path where it is mounted `alluxio.user.client.cache.dirs`
- Assuming you already have Trino in your environment, and set up similarly as
  [Trino deployment documentation](https://trino.io/docs/current/installation/deployment.html). Note down the
  installation directory of Trino. We will refer to it as `${TRINO_HOME}` throughout this document which
  you will need to update.

Request a trial version of Alluxio Edge. Contact your Alluxio account representative at `sales@alluxio.com` to request
a trial version of Alluxio Edge. Follow their instructions to download the installation tar file into the directory you prepared.

The tar file follows the naming convention `alluxio-enterprise-*.tar.gz`. For example, if the tarball is named
`alluxio-enterprise-3.x-4.0.0-bin-97a746cf8c.tar.gz`, the alluxio version is `3.x-4.0.0`.

## Package Alluxio Edge with Trino

### Remove any old Alluxio client jar files from the Trino directories 

Running a command like this will usually work:

```console
$ find ${TRINO_HOME} -name alluxio*shaded* -exec rm {} \;
```

### Extract Alluxio Edge jars and Place into the Trino Directories

Two Alluxio Edge Java JAR files must be installed on each Trino node.

First, extract the Alluxio Edge client JAR file from the tar file using this command:

```console
$ tar xf alluxio-enterprise-*.tar.gz alluxio-enterprise-*/client/alluxio-emon-*-client.jar
$ cp alluxio-enterprise-*/*/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/hive
$ cp alluxio-enterprise-*/*/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/hudi
$ cp alluxio-enterprise-*/*/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/delta-lake
$ cp alluxio-enterprise-*/*/alluxio-emon-*-client.jar ${TRINO_HOME}/plugin/iceberg
```

Then, extract the Alluxio Edge S3 under store filesystem integration JAR file using this command:

```console
$ tar xf alluxio-enterprise-*.tar.gz alluxio-enterprise-*/lib/alluxio-underfs-emon-s3a-*.jar
$ cp alluxio-enterprise-*/*/alluxio-underfs-emon-s3a-*.jar ${TRINO_HOME}/lib/.
```

### Download the Prometheus jar

[Prometheus](https://github.com) is the recommended database for observing the metrics that Alluxio Edge emits.
If the JMX exporter for Prometheus is not already set up in your environment, you can download the Java agent
JAR from https://github.com/prometheus/jmx_exporter/releases .
The file is named similarly to `jmx_prometheus_javaagent-0.20.0.jar`. Place it in the `${TRINO_HOME}/lib/` directory.

```console
$ cp jmx_prometheus_javaagent-0.20.0.jar ${TRINO_HOME}/lib/.
```

## Update Configurations

### Update Alluxio Configurations in Trino JVM Config

You can configure Alluxio configuration properties via the Trino `jvm.config` file, which is usually inside
`${TRINO_HOME}/etc/`, with the following format

```
  -Dalluxio.<property name>=<value>
```

#### Defines the Alluxio Edge conf directory. 

For example, to set `${TRINO_HOME}/etc/alluxio/` as the configuration directory for Alluxio Edge under, you would do the following

```
# Reference the Alluxio property file
-Dalluxio.conf.dir=${TRINO_HOME}/etc/alluxio/
```

We will use 
#### Configure Alluxio Edge metrics for Prometheus integration.

Create `jmx_export_config.yaml` in configuration directory `${TRINO_HOME}/etc/alluxio/` with the following sample content.

```
  ---
  startDelaySeconds: 0
  ssl: false
  global:
    scrape_interval:     15s
    evaluation_interval: 15s
  rules:
  - pattern: ".*"
```

In the `jvm.config` file, add the following:

```
# Setup Alluxio Edge cache metrics
-Dalluxio.metrics.conf.file=${TRINO_HOME}/etc/alluxio/alluxio-metrics.properties
-javaagent:${TRINO_HOME}/lib/jmx_prometheus_javaagent-0.20.0.jar=9696:${TRINO_HOME}/etc/alluxio/jmx_export_config.yaml
```

### Create Alluxio Edge Properties File

Create the `alluxio-site.properties` file and place it in the config directory set in previous step. In our
example, it is `${TRINO_HOME}/etc/alluxio/`.

```
# FILE: alluxio-site.properties
#
# DESC: This is the main Alluxio Edge properties file and should
#      be placed in the Alluxio Edge config directory, for example ${TRINO_HOME}/etc/alluxio/
#

#
# Insert custom definitions here
#

# end of file
```

Please refer to [configuration settings]({{ '/en/reference/Configuration.html#alluxio-site-properties' | relativize_url }}) for details.

### Enable S3AFileSystem

Add the following to `${TRINO_HOME}/etc/alluxio/alluxio-core-site.xml` to include the fs.s3a.impl property to ensure that Trino
uses the S3AFileSystem when a Hive table LOCATION is set to the `s3a://` scheme.

```
 <!-- Enable the Alluxio Edge Cache Integration for s3 URIs -->
  <property>
    <name>fs.s3.impl</name>
    <value>alluxio.emon.hadoop.FileSystemEE</value>
  </property>

  <!-- Enable the Alluxio Edge Cache Integration for s3a URIs -->
  <property>
    <name>fs.s3a.impl</name>
    <value>alluxio.emon.hadoop.FileSystemEE</value>
  </property>
```

### Update Catalogs

Configure the Trino catalog (such as HIVE and Delta Lake catalog) to reference the Alluxio enabled 
`core-site.xml` file to the resources. You can likely find the catalog files in `${TRINO_HOME}/etc/catalog/`.

{% navtabs catalog %}

{% navtab HIVE %}

Modify the `hive.properties` file and add:

```
    connector.name=hive
    hive.s3-file-system-type=HADOOP_DEFAULT
    hive.metastore.uri=thrift://<MY_HIVE_METASTORE_HOSTNAME>:9083
    hive.config.resources=${TRINO_HOME}/etc/alluxio/alluxio-core-site.xml
```

{% endnavtab %}

{% navtab DELTA LAKE %}

Modify the `deltalake.properties` file and add:

```
    connector.name=delta_lake
    delta.hive-catalog-name=deltalake
    delta.enable-non-concurrent-writes=true
    delta.vacuum.min-retention=1h
    delta.register-table-procedure.enabled=true
    delta.extended-statistics.collect-on-write=false
    
    hive.s3-file-system-type=HADOOP_DEFAULT
    hive.metastore.uri=thrift://<MY_HIVE_METASTORE_HOSTNAME>:9083
    hive.config.resources=${TRINO_HOME}/etc/alluxio/alluxio-core-site.xml
```

{% endnavtab %}

{% navtab ICEBERG %}

Modify the `iceberg.properties` file and add:

```
    connector.name=iceberg
    iceberg.catalog.type=hive_metastore
    iceberg.file-format=PARQUET

    hive.s3-file-system-type=HADOOP_DEFAULT
    hive.metastore.uri=thrift://<MY_HIVE_METASTORE_HOSTNAME>:9083
    hive.config.resources=${TRINO_HOME}/etc/alluxio/alluxio-core-site.xml
```

{% endnavtab %}

{% endnavtabs %}

Some optional configurations 

```  
    hive.non-managed-table-writes-enabled=true
    hive.s3select-pushdown.enabled=true
    hive.storage-format=PARQUET
    hive.allow-drop-table=true
```

## Deploy Trino

Now you can relaunch Trino with Alluxio Edge.