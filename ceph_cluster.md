# Introduction

This document is intended to document design decisions and implementation of 
ceph storage infrastructure within jiocloud.

# Overview

Ceph cluster is used to provide block and object storage for openstack cloud within jiocloud.

It provide reliable, replicated, clustered storage system with block, file sytem and object storage interfaces.

Ceph cluster has three basic daemons

1. ceph monitor

	A Ceph Monitor maintains a master copy of the cluster map. One or more instances of ceph-mon
form a Paxos part-time parliament cluster that provides extremely reliable and durable storage of 
cluster membership, configuration, and state. 

2. Ceph OSDs

	Ceph-osd is the object storage daemon for ceph distributed storage system.It is responsible for storing
objects on a local file system and providing access to them over the network.

3. Rados Gateway

	Radosgw is an HTTP REST gateway for the RADOS object store, a part of the Ceph distributed storage system. 
It is implemented as a FastCGI module using libfcgi, and can be used in conjunction with any FastCGI capable web server.
Radosgw support both S3 and swift compable APIs to access the storage.

4. Ceph MDS

	Ceph-mds is the metadata server daemon for the Ceph distributed file system. 
One or more instances of ceph-mds collectively manage the file system namespace, coordinating access to the shared OSD cluster.


# Current implementation

Storage nodes hardware

* 40 Core CPU, 64 GB Server
* 22 Disks (20x1TB SATA disks, 2x400GB SSDs) 
* 2x10GB NICs (One for storage access, One for storage cluster)

Storage cluster configuration

* Two NICs on storage boxes which are connected to different networks
    Two networks used for ceph cluster - one is for storage access, to access the storage
from compute nodes - this is the path to get block disks accessed. 

    Another NIC is connected to storage cluster network, which is used for ceph storage cluster
communication and replications.

* Three ceph monitor nodes per storage cluster
    Three ceph monitors are configured per system to have redundant ceph monitor cluster. This will assure the 
high availability of ceph monitors.

* 20 OSDs per node - only SATA Used, no SSDs are being used.

	Each disk has one ceph osd (a ceph daemon to serve the content in disk storage) configured. 
Decision was made to avoid using SSDs for datastorage to reduce the cost. Also SSDs will not be used
as journal, as adding more than 4 OSD per SSD can cause contention on SSD. Also consolidating more OSDs
per SSD can cause SSD to be single point of failure, as SSD failure can cause all OSDs hosted on that SSD to fail.

* OSD journal is configured with first partition of each disk

     First partition of each disk is configured as OSD Journal. There are two other options for journals

    1. Filesystem journal: This is avoided because of poor performance
    2. SSD Journal: SSD journal perform well, but keeping more than 4-5 OSDs per SSD can cause
       contention in SSD, and consolidating more OSDs in SSD can cause SSD to be single point of failure.
       i.e, in case of one SSD failed, all OSDs backed by that SSD will go down.

* Three Radosgw servers
    Radosgw is implemented as fastcgi application on apache server. Added three Radosgw servers. 
Radosgw is integrated with Openstack keystone so that one can use Openstack credentials to connect
to ceph radosgw.

    The data flow in Radosgw is protected using SSL, all traffic using https which are offloaded right on 
the radosgw server.

* Number of object replicas is three
     We use replica facter of 3 for all pools, i.e all data stored in ceph cluster will be replicated to three 
individual systems depending on the cluster map. Because of this configuration, ceph cluster need at least three
servers.

# Infrastructure automation using puppet

We used enovance ceph puppet module to deploy ceph cluster. This module has been customized to support radosgw,
https support, ceph OSD journal in first partition etc. Also added support to automatically detect the available disks
for OSDs and for testing, added loopback device support.

## Tips for puppet code testing

* In order to use loopback disks (autogenerate the disks), one have to set hiera rjil::ceph::osd::autogenerate to true
    and optionally set rjil::ceph::osd::autodisk_size with appropriate disk size in GB. The detault size of autodisk_size
    is 10GB which is minimum disk to run the ceph cluster properly. The reason why it need 10GB disk is that:
  * ceph journal need to be minimum 2GB size, otherwise  journal creation fail
  * Ceph osd disk need at least 7-8 GB, otherwise data write to ceph osds fail.
* As default number of object replica for all pools are 3, by default the cluster need at least 3 servers. In order to 
    test ceph with one machine, the pull request #82 in puppet-rjil to be merged and set hiera data "rjil::ceph::pool_default_size: 1"
* With current ceph module, it need 2-3 puppet runs before the cluster to be up. This is because ceph module use facter to determine
certain actions.

# TODO

* Implement proper clustermap
    Appropriate clustermap to be implemented in ceph cluster. Currently default ceph clustermap is used.
* General performance and Security improvements.
* Use stackforge ceph module
* Start the cluster (ceph monitor specifically) with single node and grow - this is to be done using etcd/consul.
* Cache Tiering POC
   Cache Tiering improves the performance by creating a relatively fast storage devices configured to act as cache tier 
and a backing pool of cheaper, high capacity disks. 

