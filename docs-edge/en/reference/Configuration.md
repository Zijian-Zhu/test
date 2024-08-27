---
layout: global
title: Configuration
---

In this documentation we focus on the content, please see deployment instruction for where to put them.

## Alluxio Site Properties

Alluxio Edge can be configured by setting the values of configuration properties within `alluxio-site.properties`.

### Specify the Root Under File System

{% navtabs ufs %}

{% navtab AWS S3 %}

This sample code specifies AWS S3 as the root under file system that Alluxio Edge will be accessing.

The minimal configuration needed is to set credentials for S3 so that Alluxio Edge can access S3. It can be done with
[Instance Profiles](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#ec2-instance-profile)
or if Instance Profile is not used, secret key and access key are needed. Please see the sample configuration below
for setting the secret key and access key. Please update `s3a.accessKeyId` and `s3a.secretKey` corresponding to your
own S3 credentials mentioned in prerequisites.
```
s3a.accessKeyId=<MY_KEY_ID>
s3a.secretKey=<MY_SECRET_KEY>
```

(Optional) To connect to S3 from a different region.
```
# For S3 endpoints
alluxio.underfs.s3.endpoint=s3.<PUT_YOUR_AWS_REGION_HERE>.amazonaws.com
alluxio.underfs.s3.endpoint.region=<PUT_YOUR_AWS_REGION_HERE> # Example: us-east-1
# For S3 buckets
alluxio.underfs.s3.region=<PUT_YOUR_AWS_REGION_HERE> # Example: us-east-1
```

(Optional) To enable TLS in Alluxio Edge if server side encryption is on.
```
alluxio.underfs.s3.secure.http.enabled=true
alluxio.underfs.s3.server.side.encryption.enabled=true
```

(Optional) If assume role is enabled in S3.
```
alluxio.underfs.s3.assumerole.enabled=true
alluxio.underfs.s3.assumerole.rolearn=<PUT_YOUR_ASSUME_ROLE_ARN> # Example: arn:aws:iam::...
```

{% endnavtab %}

{% endnavtabs %}

### Specify the Cache Storage Directories

Trino worker and coordinator needs to know the Alluxio Edge cache size `alluxio.user.client.cache.size` and
path `alluxio.user.client.cache.dirs`. This sample code specifies the cache storage directories Alluxio Edge will use.

```
# Enable edge cache on client (RAM disk only)
#
#alluxio.user.client.cache.enabled=true
#alluxio.user.client.cache.size=1GB
#alluxio.user.client.cache.dirs=/dev/shm/alluxio_cache

# Enable edge cache on client (example with 2 NVMe volumes)
#
alluxio.user.client.cache.enabled=true
alluxio.user.client.cache.size=600GB,600GB
alluxio.user.client.cache.dirs=/mnt/nvme0/alluxio_cache,/mnt/nvme1/alluxio_cache
```

### Other configurations

```
# Disable DORA
#
alluxio.dora.enabled=false

# Prevent Alluxio Edge from creating 0 byte objects
#
alluxio.underfs.object.store.breadcrumbs.enabled=false

# (Optional) Enable edge metrics collection
#
alluxio.user.metrics.collection.enabled=true
```

