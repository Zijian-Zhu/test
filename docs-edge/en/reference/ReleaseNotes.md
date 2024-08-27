---
layout: global
title: Release Notes
---

Alluxio is pleased to introduce Edge 1.0, a lightweight fast data access caching library for big data analytics compute
engines, such as Trino and PrestoDB. Alluxio Edge provides performance boots by bringing data local while slashing
the cloud cost at the same time.

* Integrates with the popular big data analytics tool Trino (up to version 431) and PrestoDB.
* Able to utilize various types of storages medium as local cache: (preferred) NVME (DDR or DIMM), memory,
  Local SSD (PCIe/SATA SSD), and HDD.
* Rich data lake compatibility: Hive, Iceberg, Delta Lake, and Hudi (experimental).
* Rich data formats compatibility: all formats Trino and PrestoDB support - Parquet, ORC, CSV, Txt, JSON, Avro
* Capable of caching both data, and metadata (hive metadata store only).
* Provides LRU as cache eviction policy.
* Cache filter option provides customized admission to optimize cache usage.
* Accompanied by rich metrics and monitoring guide for monitoring the cluster, and understanding the impact of cache financially.