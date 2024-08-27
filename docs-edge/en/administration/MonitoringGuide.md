---
layout: global
title: Monitoring Guide
---

Alluxio exposes metrics via the Java Management Extensions (JMX). Once it is enabled
according to the installation instructions, you can use the metrics to monitor the cluster.
See full [list of metrics]({{ '/en/reference/MetricList.html' | relativize_url }}). Below we list a
scenarios where you may find certain metrics helpful.

## Monitoring the Cluster

It is recommended to monitor the following metrics for particular presto workers in question and cluster-wide.

### Monitoring Cache Health
To check the healthiness of the cache, please check out the metric `CacheState` :
* 0 `NOT_IN_USE`: failed to init or restore and can not recover, e.g., disk is not accessible
* 1 `READ_ONLY`: get operations work, not for put and delete, e.g., cache is still recovering from previous start
* 2 `READ_WRITE`: normal and expected state

Each cache will report its own status, so that it is easy to determine if there is any or which
cache is in the undesired state.

### Understanding PUT Errors (Writing Into Cache)
In case of high number of PUT errors seen, collecting following metrics to understand reasons of puts

* `CachePutErrors`: total number of put errors.

This should be the summation of following detailed put errors
* `CachePutNotReadyErrors`: Cache is still recovering, not in READ_WRITE mode.
* `CachePutStoreWriteErrors`: Failed to write new pages to local disk.
* `CachePutStoreDeleteErrors`: Failed to delete evicted pages (data) from local disk.
* `CachePutBenignRacingErrors`: Certain race condition happen but doesn't affect the system. i.e. 2 threads trying to delete the same page.
* `CachePutEvictionErrors`: Cache metadata eviction failures.
* `CachePutAsyncRejectionErrors`: Async write is enabled and thread pool limit is reached. This should not happen
  with the latest implementation. If this metrics is > 0 there might be an issue. 

### Understanding GET Errors (Reading From Cache)

In case of high number of GET errors seen, collecting following metrics to understand reasons of puts
* `CacheGetErrors`: Total number of get errors.

This should be the summation of following detailed get errors
* `CacheGetNotReadyErrors`: Cache is in NOT_IN_USE mode
* `CacheGetStoreReadErrors`: Failed to read pages from local disk

### Understanding Cache Hit Ratio

There are two kinds of cache hit ratio here. They may differ slightly, but in general we expect them to track
each other closely.

<table class="table table-striped">
  <thead>
    <tr>
      <th>Name</th>
      <th>Expression</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Data Cache Hit Ratio</td>
      <td>CacheBytesReadCache / (CacheBytesReadCache + CacheBytesRequestedExternal)</td>
      <td>Percentage of bytes the cache is able to fulfill compared to total bytes being requested</td>
    </tr>
    <tr>
      <td>Data Request Cache Hit Ratio</td>
      <td>CacheHitRequests / (CacheHitRequests + CacheExternalRequests)</td>
      <td>Percentage of data request the cache is able to fulfill compared to total data requests</td>
    </tr>
  </tbody>
</table>

We want these two rates to be 80% or higher to make sure the cache is used efficiently. A rate of 90% or 95%+ is
ideal.

### Hardware metrics

These hardware metrics indicates the cache utilization level
* `CacheSpaceUsed`
* `CacheSpaceAvailable`

If the cache utilization rate is high, and the cache hit rate is low, may consider increase the size of the cache,
or use filter to control what goes into the cache to make it more efficient.

### Others

* `Young GC` and `old GC` (part of your JVM setup): Spike in this may indicate abnormality.
  May need to check your GC spec. 
* `cacheByteExternalRequest` and `cacheExteranlRequest`: Spike in this may lead to read failure
* `CacheBytesEvicted`: We expect constant and steady eviction to happen, but if there is spike, 
  it may indicate issue, or it may lead to failure.
* `CachePages`: We expect this to be normally full. But during initialization of the worker, we will reload the cache,
  so you should see this metrics to start from 0 and gradually go up to almost full during initialization. 

## Understanding the Impact of Cache Financially

Cloud vendors charges for requests made against your cloud storage and objects.
Usually request costs are based on the request type (2 tiers of pricing), and are charged on the quantity of requests.
We have provided the count of such relevant requests to assist you understand the impact of cache financially.

Currently, the Cloud vendors defines the tiers as following (ex. [AWS S3 pricing](https://aws.amazon.com/s3/pricing/)):
* Tier 1: PUT, COPY, POST, LIST requests
* Tier 2: GET, SELECT, and all other requests

In Alluxio Edge context, the relevant ones are:
* Tier 1: `ListStatusExternalRequests` (LIST)
* Tier 2: `CacheExternalRequests` (GET), `GetFileInfoExternalRequests` (GET)

For each request comes to Alluxio Edge, it can either be a cache hit and served by Alluxio Edge (free), 
or a cache miss and goes directly to the understorage (same as before adding Alluxio Edge).
So the cost after adding Alluxio can be calculated by removing the cache hit request counts. For example: 
* Original Tier 1 cost: (`ListStatusHitRequests` + `ListStatusExternalRequests`) * <price of tier 1 requests> 
* Original Tier 2 cost: (`CacheHitRequests` + `CacheExternalRequests` + `GetFileInfoHitRequests` + `GetFileInfoExternalRequests`) * <price of tier 1 requests>
* New Tier 1 cost with cache: (`ListStatusExternalRequests`) * <price of tier 1 requests>
* New Tier 2 cost with cache: (`CacheExternalRequests` + `GetFileInfoExternalRequests`) * <price of tier 1 requests>

NOTE:
* This is an estimation for guidance purpose.
* Different data lake format may affect the kind of requests are sent to Alluxio Edge. 