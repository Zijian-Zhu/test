---
layout: global
title: Deploy Alluxio Edge for Trino
---

This document describes how to deploy Alluxio Edge for Trino to work with AWS S3 as UFS.
The key step is to place Alluxio jar files to the Trino path, update configuration files,
then proceed with the normal deployment of Trino.

## Prerequisites

- Your AWS S3 credentials
- Preparation of storage for Alluxio Edge: After identifying the storage mounted for Alluxio Edge cache, please note:
  - The size of the local storage to be provisioned for Alluxio Edge `alluxio.user.client.cache.size`
  - The path where it is mounted `alluxio.user.client.cache.dirs`
- Assuming you already have Trino in your environment, and set up similarly as
  [Trino deployment documentation](https://trino.io/docs/current/installation/deployment.html). Note down the
  installation directory of Trino. We will refer to it as `${TRINO_HOME}` throughout this document which
  you will need to update.
- A running [ETCD cluster](https://etcd.io/docs/v3.5/install/); the set of endpoint URLs is a required configuration setting.

Request a trial version of Alluxio Edge.
Contact your Alluxio account representative at `sales@alluxio.com` to request a trial version of Alluxio Edge.
Follow their instructions to download the installation tar file into the directory you prepared.

The tar file follows the naming convention `alluxio-enterprise-edge-*.tar.gz`. For example, if the tarball is named
`alluxio-enterprise-edge-1.1-6.0.0.tar.gz`, the alluxio version is `edge-1.1-6.0.0`.

## Package Alluxio Edge with Trino

### Remove any old Alluxio client jar files from the Trino directories

Running a command like this will usually work:

```console
$ find ${TRINO_HOME} -name alluxio*shaded* -exec rm {} \;
```

### Extract Alluxio Edge jars and Place into the Trino Directories

Three Alluxio Edge Java JAR files must be installed on each Trino node.

Extract the Alluxio Edge S3 under store filesystem integration JAR file using this command:

```console
$ tar xf alluxio-enterprise-edge-*.tar.gz alluxio-enterprise-edge-*/lib/alluxio-underfs-s3a-*.jar
$ cp alluxio-enterprise-edge-*/*/alluxio-underfs-s3a-*.jar ${TRINO_HOME}/lib/.
```

Extract the client and prod jar files from the tarball using this command:

```console
$ tar xf alluxio-enterprise-edge-*.tar.gz alluxio-enterprise-edge-*/client/alluxio-*-client.jar
$ tar xf alluxio-enterprise-edge-*.tar.gz alluxio-enterprise-edge-*/assembly/alluxio-prod-*.jar
```

Depending on your the Trino version, the jars need to be copied to different destination.
- For Trino versions 434 or later, copy to `${TRINO_HOME}/plugin/<pluginName>/hdfs`
- For Trino versions older than 434, copy to `${TRINO_HOME}/plugin/<pluginName>`

> Note there is no impact in copying jars to both destinations

The following example shows the copy commands for version 434+ into the plugin directories for hive, hudi, delta lake, and iceberg.
```console
$ cp alluxio-enterprise-edge-*/*/alluxio-*-client.jar ${TRINO_HOME}/plugin/hive/hdfs
$ cp alluxio-enterprise-edge-*/*/alluxio-prod-*.jar ${TRINO_HOME}/plugin/hive/hdfs
$ cp alluxio-enterprise-edge-*/*/alluxio-*-client.jar ${TRINO_HOME}/plugin/hudi/hdfs
$ cp alluxio-enterprise-edge-*/*/alluxio-prod-*.jar ${TRINO_HOME}/plugin/hudi/hdfs
$ cp alluxio-enterprise-edge-*/*/alluxio-*-client.jar ${TRINO_HOME}/plugin/delta-lake/hdfs
$ cp alluxio-enterprise-edge-*/*/alluxio-prod-*.jar ${TRINO_HOME}/plugin/delta-lake/hdfs
$ cp alluxio-enterprise-edge-*/*/alluxio-*-client.jar ${TRINO_HOME}/plugin/iceberg/hdfs
$ cp alluxio-enterprise-edge-*/*/alluxio-prod-*.jar ${TRINO_HOME}/plugin/iceberg/hdfs
```

Along with the Alluxio client and prod jars in the plugin directories, an additional third party jar is needed.
[Download](https://repo1.maven.org/maven2/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.jar) the commons-lang3-3.14.0 jar
and place it in the same directories.
```console
$ wget https://repo1.maven.org/maven2/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.jar
$ cp commons-lang3-3.14.0.jar ${TRINO_HOME}/plugin/hive/hdfs
$ cp commons-lang3-3.14.0.jar ${TRINO_HOME}/plugin/hudi/hdfs
$ cp commons-lang3-3.14.0.jar ${TRINO_HOME}/plugin/delta-lake/hdfs
$ cp commons-lang3-3.14.0.jar ${TRINO_HOME}/plugin/iceberg/hdfs
```

### Download the Prometheus jar

[Prometheus](https://github.com) is the supported tool for observing the metrics that Alluxio Edge emits.
[Download](https://repo.maven.apache.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar)
the java agent jar and save it in the `${TRINO_HOME}/lib/` directory.

```console
$ wget https://repo.maven.apache.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar -O ${TRINO_HOME}/lib/jmx_prometheus_javaagent-0.20.0.jar.
```

## Update Configurations

### Update Alluxio Configurations in Trino JVM Config

Configure Alluxio configuration properties in Trino's `jvm.config` file, which is usually located at
`${TRINO_HOME}/etc/`. The following settings need to be added:
```
-Dalluxio.conf.dir=${TRINO_HOME}/etc/alluxio/
--add-opens=java.management/sun.management=ALL-UNNAMED
-Dalluxio.metrics.conf.file=${TRINO_HOME}/etc/alluxio/alluxio-metrics.properties
-javaagent:${TRINO_HOME}/lib/jmx_prometheus_javaagent-0.20.0.jar=9696:${TRINO_HOME}/etc/alluxio/jmx_export_config.yaml
```

- `alluxio.conf.dir` is the directory containing Alluxio specific configuration files
- The `add-opens` line allows compatibility with Java 17
- `alluxio.metrics.conf.file` is the path to configuration file for metrics
- `javaagent` line starts a metrics agent to pull metrics from. In this example, it runs on port 9696 and the jmx_export_config.yaml file defines its configuration.

Note any additional Alluxio properties can be specified following the `-D<key>=<value>` format.

#### Create metrics.properties and jmx_export_config.yaml for Prometheus integration with metrics

Create `jmx_export_config.yaml` in `${TRINO_HOME}/etc/alluxio/` with the following sample content.

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

Also create `metrics.properties` in `${TRINO_HOME}/etc/alluxio/` with the following sample content.

```
sink.jmx.class=alluxio.metrics.sink.JmxSink
```

### Create alluxio-site.properties for Alluxio Edge configuration

Create `alluxio-site.properties` in `${TRINO_HOME}/etc/alluxio/`.
The following configuration should be set:

```
alluxio.license=<YOUR LICENSE STRING>
# ex. alluxio.etcd.endpoints=http://trino-edge-etcd1:2379,http://trino-edge-etcd2:2379,http://trino-edge-etcd3:2379
alluxio.etcd.endpoints=<YOUR_ETCD_ENDPOINTS>
alluxio.dora.enabled=false
alluxio.underfs.object.store.breadcrumbs.enabled=false

# Alluxio under file system setup (S3)
# ex. alluxio.dora.client.ufs.root=s3://myBucket
alluxio.dora.client.ufs.root=<YOUR_S3_URL>
# credentials to access S3 bucket, optional if credentials are provided in other ways
s3a.accessKeyId=<MY_KEY_ID>
s3a.secretKey=<MY_SECRET_KEY>

# Enable Alluxio Edge cache on Trino nodes (example with 2 SSD volumes)
alluxio.user.client.cache.enabled=true
alluxio.user.client.cache.size=<PUT_YOUR_SSD1_VOL_SIZE_HERE>GB,<PUT_YOUR_SSD2_VOL_SIZE_HERE>GB
alluxio.user.client.cache.dirs=<PUT_YOUR_SSD1_VOL_MOUNT_POINT_HERE>/alluxio_cache,/<PUT_YOUR_SSD2_VOL_MOUNT_POINT_HERE>/alluxio_cache

# Enable metrics collection
alluxio.user.metrics.collection.enabled=true
```

Please refer to [configuration settings]({{ '/en/administration/Configuration.html#alluxio-site-properties' | relativize_url }}) for details.

### Create alluxio-core-site.xml for overriding the S3 filesystem

Create `alluxio-core-site.xml` in `${TRINO_HOME}/etc/alluxio/` to include the `fs.s3a.impl` property to ensure that Trino
uses the `S3AFileSystem` when a Hive table LOCATION is set to the `s3://` or `s3a://` scheme.

```
<configuration>
  <property>
    <name>fs.s3.impl</name>
    <value>alluxio.hadoop.FileSystem</value>
  </property>
  <property>
    <name>fs.s3a.impl</name>
    <value>alluxio.hadoop.FileSystem</value>
  </property>
  <property>
    <name>fs.alluxio.impl</name>
    <value>alluxio.hadoop.FileSystem</value>
  </property>
</configuration>
```

### Update Catalogs

Configure the Trino catalog (such as HIVE and Delta Lake catalog) to include `alluxio-core-site.xml` file to the resources.
You can likely find the catalog files in `${TRINO_HOME}/etc/catalog/`.
Note that the `hive.config.resources` property may already be set with existing xml files, ie. `/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml`
In this case, `alluxio-core-site.xml` should be appended to the end of the list.

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

## Redeploy Trino

If Trino was previously running, it must be restarted to pick up the configuration changes applied.
