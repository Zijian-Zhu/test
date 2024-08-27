---
layout: global
title: Compatibility
---

## Compute Engines
* Trino (Up to 431)
* PrestoDB

## Data Lake Support
* Hive
* Iceberg
* Delta Lake
* Hudi (experimental)

## Data Formats
Below are all the data formats Alluxio Edge supports

* Parquet
* ORC
* CSV
* Txt
* JSON
* Avro

## Storage Types

<table class="table table-striped">
  <thead>
    <tr>
      <th>Storage Type</th>
      <th>When to use</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Memory</td>
      <td>If memory is large like 384G or 512G and above，ok to use.</td>
    </tr>
    <tr>
      <td>NVME (DDR or DIMM) - preferred</td>
      <td>If performance improvement is more important than cost saving</td>
    </tr>
    <tr>
      <td>Local SSD (PCIe/SATA SSD)</td>
      <td>if cost saving is more important than cost saving</td>
    </tr>
    <tr>
      <td>HDD</td>
      <td>Only if cost saving is the main goal and performance is a none goal</td>
    </tr>
  </tbody>
</table>