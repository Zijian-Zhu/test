---
layout: global
title: List of Metrics
---

## Cache Utilization Summary

<table class="table table-striped">
  <thead>
    <tr>
      <th>Name</th>
      <th>Type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>CacheState</td>
      <td>GAUGE</td>
      <td>Cache status for Coordinator and Workers (individually). 
          0: Not Available. 1: Read Only. 2. Fully Ready</td>
    </tr>
    <tr>
      <td>CacheSpaceUsed</td>
      <td>GAUGE</td>
      <td>Used cache space</td>
    </tr>
    <tr>
      <td>CacheSpaceAvailable</td>
      <td>GAUGE</td>
      <td>Total available cache space</td>
    </tr>
    <tr>
      <td>CachePages</td>
      <td>COUNTER</td>
      <td>Number of cached pages</td>
    </tr>
    <tr>
      <td>CachePagesEvicted</td>
      <td>COUNTER</td>
      <td>Number of pages evicted from cache</td>
    </tr>
    <tr>
      <td>CacheBytesEvicted</td>
      <td>COUNTER</td>
      <td>Number of bytes evicted from cache</td>
    </tr>    
    <tr>
      <td>CachePutEvictionErrors</td>
      <td>COUNTER</td>
      <td>Errors count from cache eviction</td>
    </tr>
  </tbody>
</table>

## Data Retrieval Statistics

The following metrics provide the statistics during data retrieval.

<table class="table table-striped">
  <thead>
    <tr>
      <th>Name</th>
      <th>Type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>CacheBytesReadCache</td>
      <td>COUNTER</td>
      <td>Data Read from Cache in bytes (Data Retrieval Cache Hits)</td>
    </tr>
    <tr>
      <td>CacheBytesRequestedExternal</td>
      <td>COUNTER</td>
      <td>Data Read from UFS in bytes (Data Retrieval Cache Misses)</td>
    </tr>
  </tbody>
</table>

## Storage Request Statistics

Cloud vendors charges for requests made against your cloud storage and objects. 
Usually request costs are based on the request type, and are charged on the quantity of requests. 
Here we provide a count of such relevant requests. 

<table class="table table-striped">
  <thead>
    <tr>
      <th>Name</th>
      <th>Type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>CacheHitRequests</td>
      <td>COUNTER</td>
      <td>Data request served by Cache (Storage Request Cache Hits)</td>
    </tr>
    <tr>
      <td>CacheExternalRequests</td>
      <td>COUNTER</td>
      <td>Data request to UFS (Storage Request Cache Miss).</td>
    </tr>
    <tr>
      <td>GetFileInfoHitRequests</td>
      <td>COUNTER</td>
      <td>'GET' request served by Cache (Storage Request Cache Hits).</td>
    </tr>
    <tr>
      <td>GetFileInfoExternalRequests</td>
      <td>COUNTER</td>
      <td>'GET' request to UFS (Storage Request Cache Miss).</td>
    </tr>
    <tr>
      <td>ListStatusHitRequests</td>
      <td>COUNTER</td>
      <td>'LIST' request served by Cache (Storage Request Cache Hits).</td>
    </tr>
    <tr>
      <td>ListStatusExternalRequests</td>
      <td>COUNTER</td>
      <td>'LIST' request to UFS (Storage Request Cache Miss).</td>
    </tr>
  </tbody>
</table>

## Error Messages

### PUT Errors (Writing Into Cache)

<table class="table table-striped">
  <thead>
    <tr>
      <th>Name</th>
      <th>Type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>CachePutErrors</td>
      <td>COUNTER</td>
      <td>Total error count when data write into cache. Summation of following detailed put errors.</td>
    </tr>
    <tr>
      <td>CachePutNotReadyErrors</td>
      <td>COUNTER</td>
      <td>Count of the occurrences of failing to write because cache is still recovering, 
          not in READ_WRITE mode.</td>
    </tr>
    <tr>
      <td>CachePutStoreWriteErrors</td>
      <td>COUNTER</td>
      <td>Count of the occurrences of failing to write new pages to local disk.</td>
    </tr>
    <tr>
      <td>CachePutStoreDeleteErrors</td>
      <td>COUNTER</td>
      <td>Count of the occurrences of failing to delete evicted pages from local disk</td>
    </tr>
    <tr>
      <td>CachePutBenignRacingErrors</td>
      <td>COUNTER</td>
      <td>Count of the occurrences of certain race condition but doesn't affect the system. 
          i.e. 2 threads trying to delete the same page.</td>
    </tr>
    <tr>
      <td>CachePutEvictionErrors</td>
      <td>COUNTER</td>
      <td>Count of the occurrences of cache eviction failures</td>
    </tr>
    <tr>
      <td>CachePutAsyncRejectionErrors</td>
      <td>COUNTER</td>
      <td>Count of the occurrences of thread pool limit reached when async write is enabled.</td>
    </tr>
  </tbody>
</table>

### GET Errors (Reading From Cache)

<table class="table table-striped">
  <thead>
    <tr>
      <th>Name</th>
      <th>Type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>CacheGetErrors</td>
      <td>COUNTER</td>
      <td>Error count when data read from cache. Summation of following detailed GET errors.</td>
    </tr>
    <tr>
      <td>CacheGetNotReadyErrors</td>
      <td>COUNTER</td>
      <td>Count of occurrences of read failure due to cache is in NOT_IN_USE mode.</td>
    </tr>
    <tr>
      <td>CacheGetStoreReadErrors</td>
      <td>COUNTER</td>
      <td>Count of occurrences of failing to read pages from local disk.</td>
    </tr>
  </tbody>
</table>

## Machine Resources

NOTE: The name is a reference, it may vary slightly based on your environment. 

<table class="table table-striped">
  <thead>
    <tr>
      <th>Name</th>
      <th>Type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>CpuLoad</td>
      <td>GAUGE</td>
      <td>CPU Load</td>
    </tr>
    <tr>
      <td>maxFileCount</td>
      <td>GAUGE</td>
      <td>OS Max File Count</td>
    </tr>
    <tr>
      <td>openFileCount</td>
      <td>GAUGE</td>
      <td>OS Open File Count</td>
    </tr>
  </tbody>
</table>
