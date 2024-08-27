---
layout: global
title: Release Notes
---

# Alluxio Edge 1.1

Alluxio is pleased to introduce Edge 1.1! In this version, a more robust licensing system is introduced. 
As part of the improvement, an [ETCD cluster](https://etcd.io/) is required.
See deployment instruction for details. 

## Configuration Changes

### Property keys added and their default values
* `alluxio.etcd.endpoints=<YOUR_ETCD_ENDPOINTS>`
* `alluxio.license=<YOUR LICENSE STRING>`