---
layout: global
title: Configuring Cache Filter
---

## What is Cache Filter

Sometimes we may not want every single data to go into the cache, and customized admission is necessary. For example
* There is limited space for the cache storage, and there is known "hotness" of the data (or tables).
* Certain data are critical and needs to be fresh always, so we don't want to cache them. ie. Data lake manifest files.

Cache Filter is such an advanced feature that provides a way for the Admin to define customized logic to control
what data can be admitted into the cache. 

## Filter Types

Currently, the supported filter types are
* `CACHE ALL`: Admit all data into the cache. 
* `ALLOW LIST`: Admit only data that matches the regex in the list into the cache.
* `BLOCK LIST`: Do not admit data that matches the regex in the list into the cache.

## Configuring Cache Filter

### Enable Cache Filter 

Set the following configuration in `alluxio-site.properties`.
Please refer to [installation guide]({{ '/en/installation/DeployAlluxioEdgeWithK8s.html' | relativize_url }})
for instruction on how to update `alluxio-site.properties`.

```
alluxio.user.client.cache.filter.enabled=true
alluxio.user.client.cache.filter.class=alluxio.client.file.cache.filter.PatternBasedCacheFilter
``` 

### Set Policy 

The policies are set in `cache_filter.properties`.
Please refer to [installation guide]({{ '/en/installation/DeployAlluxioEdgeWithK8s.html' | relativize_url }})
for instruction on how to use this file. 

Below are some sample policies for each filter type.

#### Example - Cache All

```
{
  "filterType": "CACHE_ALL"
}
```
#### Example - Allow List

```
{
  "filterType": "ALLOW_LIST",
  "regxPatternStrList": ["s3a://jiamingmai-test/.*test.*", "s3a://jiamingmai-test/.*NOTICE.*"]
}
```

#### Example - Block List

```
{
  "filterType": "BLOCK_LIST",
  "regxPatternStrList": ["s3a://jiamingmai-test/.*test.*", "s3a://jiamingmai-test/.*NOTICE.*"]
}
```