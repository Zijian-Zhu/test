---
layout: global
title: List of Features
---

## Caching Capability
* Metadata Cache (Hive metadata store only)
  * Helps during query planning stage (mainly LIST / GET requests)
* Data Cache
  * Helps during data read stage



## Cache Eviction Methods

When the cache is at capacity, more space is created by evicting data according to the Least 
Recently Used (LRU) algorithm.

## Monitoring Dashboard
Metrics and Grafana template are provided for
* Cluster summary
* Cost saving
* Resource status

## Advanced Features

### Soft affinity scheduling

This feature helps to ensure the subsequent request is routed to the previous worker to improve on cache hit ratio.

### Cache Filter

Sometimes we may not want every single data to go into the cache, and customized admission is necessary. For example
* There is limited space for the cache storage, and there is known "hotness" of the data (or tables).
* Certain data are critical and needs to be fresh always, so we don't want to cache them. ie. Data lake manifest file.

Cache Filter is such an advanced feature that provides a way for the Admin to define customized logic to control
what data can be admitted into the cache.

See [Cache Filter configuration instruction]({{ '/en/administration/ConfiguringCacheFilter.html' | relativize_url }}).

