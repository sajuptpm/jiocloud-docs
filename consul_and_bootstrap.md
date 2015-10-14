# About Consul

  Consul is a reliable, distributed service discovery toolset which have built-in
distributed key-value store, healthchecking, and with multi-datacenter
capability. consul have one rest based api and standard dns based interfaces for
service discovery - all active services and its server IP addresses will be
exposed using these interfaces.

  Consul has integrated the service discovery with built-in healthchecking
mechanism, so that only healthy service instances would be exposed to the
infrastructure - if the healthcheck fail for a service instance, consul will
take down that instance from its service catalog automatically.

# Consul Features
* Service Discovery: Clients of Consul can provide a service, such as api or
mysql, and other clients can use Consul to discover providers of a given
service. Using either DNS or HTTP, applications can easily find the services
in the network which they depend upon.

* Health Checking: Consul clients can provide any number of health checks, either
associated with a given service ("is the webserver returning 200 OK"), or with
the local node ("is memory utilization below 90%"). This information can be used
by an operator to monitor cluster health, and it is used by the service
discovery components to route traffic away from unhealthy hosts.

* Distributed Key/Value Store: Applications can make use of Consuls hierarchical
key/value store for any number of purposes, including dynamic configuration,
feature flagging, coordination, leader election, and more. The simple HTTP API 
makes it easy to use.

* Multi Datacenter: Consul supports multiple datacenters out of the box. This
means users of Consul do not have to worry about building additional layers of
abstraction to grow to multiple regions.

# Consul components and Basic building blocks

* Consul agent
  The Consul agent is the core process of Consul. The agent maintains membership
information, registers services, runs checks, responds to queries, and more. The
agent must run on every node that is part of a Consul cluster. Any agent may run
in one of two modes: client or server.

  * Server mode
  A Server mode takes additional responsibilities of participating in the consensus
quorum for servers, storing cluster state, creating updating service catalog,
k/v pairs and handling queries. 

  * Bootstrap Server mode
A server may be in "bootstrap" mode which is required to bootstrap a consul
cluster - bootstrap server will be the initial agent which start consul cluster
and other servers would be joining bootstrap server to form a cluster. After
forming a cluster, bootstrap mode should be disabled to avoid putting cluster in
inconsistant state. Also only one node must be in "bootstrap" mode in a cluster
in any given point of time.

  * Client mode
Client nodes make up the majority of the cluster, and they are very lightweight
as they interface with the server nodes for most operations and maintain very
little state of their own.

# Jiocloud implementation

## Consul cluster/server auto discovery
  Consul agent need at least one other consul agent IP address to join the
cluster. As we would be creating number of consul clusters as part of creating/deleting
automated test systems, it is difficuilt to distribute the initial server
(bootstrap server) IP address to other servers/clients. So we built a small
application based on consul which would be used to auto discover consul servers
using dns (https://github.com/sorenh/consuldiscovery/). Currently it is hosted
on http://consuldiscovery.linux2go.dk/.

This system will be setup on the server preferably on an internet facing system
and a dns pointers on domain for discovery service (in case of public dns) will
be pointed to the discovery server. Also a dnsmasq/bind to be configured to accept
the dns requests from clients and forward appropriate requests to consul dns
interface port (8600). In case this system is setup with internet facing,
anybody from internet can use this system to do consul auto discovery.

  Before one create a consul cluster or jiocloud system, one will create a
new consul discovery token from discovery service by calling
http://consuldiscovery.linux2go.dk/new (or any other url of discovery service) -
this will return a uuid based token which should be shared with all nodes in the
cluster. Jiocloud deployment and orchestration system (puppet-rjil and other
related systems) will use this token for:

1. use this token as datacenter name
2. All consul servers use this token to register themselves with discovery
system by posting its own ip and consul port to the url
http://consuldiscovery.linux2go.dk/<token/.
3. Orchestration system to use this token to create the dns query (say
mysql.service.<token>.consul is for mysql service)

## Consul components placement
  Consul agent is installed and configured on all nodes as part of userdata
script on booting the node, so that consul is available before it try to
configure any other services.

Also dnsmasq service is setup on all nodes to run dns service, which performs:

1. redirect the dns for consul to consul agent port (localhost:8600)
2. Create cname records for services endpoints especially for those services
which have ssl endpoints (ssl endpoints need specific domain name to be setup).
3. Send other dns requests to datacenter wide caching dns servers

### Cluster Bootstrap node
  The node bootstrap1 run consul server with bootstrap mode. This is the node
which initialize the consul cluster as well as entire jiocloud system.

  On booting this node, consul will be deployed and configured with bootstrap
mode and once configured, it will register itself with auto discovery service.

  Other nodes will just query autodiscovery service and wait consul server
available, and once available start deploying/configuring consul on them with
appropriate mode (server or client) and start bootstrapping jiocloud system.

### Consul server nodes
  Consul servers are running on both bootstrap1 (bootstrap mode), oc and ocdb
nodes. One of them will be the raft cluster leader which would be handling all
read/write requests.

### Consul client nodes
  All rest of the nodes in the jiocloud system will be running consul agent in
client mode. 

## Brief about current nodes in jiocloud system and their roles
### External systems
* Consul discovery server: This system have consul discovery server configured
in it as discussed above.
* DNS Servers: This is to provide external and public dns resolution to all
jiocloud systems (and to the VMs run on top of jiocloud). It may be a best
practice to use different dns server to customer VMs, in case of other dns
server is hosting any internal dns records - currently it is not the case btw.
* NTP Servers: To provide ntp service to the systems
* Others: Other nodes like log aggregation etc.

### Jiocloud systems
* bootstrap1: This node is used to bootstrap the entire system. This node is the
first system to be configured (automatically), and setup consul with bootstrap
server mode on it. Rest of the nodes will just wait till consul server is
available in the discovery server. This node only host consul bootstrap server
and not do anything else.
  It doesnt have any dependancies and is the one which finsh setup at first.
* haproxy: This node host haproxy nodes which function as load balancer for
entire api services which is accessible to external systems and end users.

  This node doesnt have any dependencies as such, except the fact that the
services on the load balancer is only registered as and when the services are
available on consul service catalog.

Note: The contrail specific api services are load balanced using haproxy running
on ct nodes, and is not there in haproxy node. This is because contrail apis are
not exposed to public and is used internally by neutron - neutron api is load
balanced from haproxy node.

* ocdb: This node has roles of both db server and openstack controller services.
It setup mysql server, memcached, and all openstack controller services:
  * cinder services - cinder api, scheduler, volume, backup
  * glance services - glance api, glance registry
  * nova services - all nova services except nova compute
  * zmq reciever - To recieve zmq messages

* oc: These nodes will have memcached and all openstack controller services
mentioned above.
* ct: contrail node - this node runs all contrail and neutron services. It has
an haproxy running to load balance contrail internal apis.
* cp: compute nodes - this run nova-compute, contrail vrouter, and zmq reciever.
Compute nodes run all VMs.
* gcp: Special compute node which has contrail vgw setup - this special setup
(vgw) is only used in virtual node environment where mx router cannot be used as
edge router, and rather vgw is act as edge router. In production and staging
this node doesnt have any special meaning than cp nodes.
* stmonleader: This is running ceph mon which initiate mon cluster. Rest of the
ceph mon  will just wait for stmonleader to be up in consul to configure
themselves. This node also host both radosgw and ceph osds at this moment.
* stmon: This has same role as stmonleader except initializing mon cluster which
is done by stmonleader.
* st: These nodes have ceph osds

## Jiocloud bootstrapping

**All nodes know what it need to do and its dependencies**: The system assume
that any nodes can come up in any order or simultaenously, every node know what
role it has (what it need to do in terms of deployment) and what are the
dependencies (inter-host service dependencies it has. All services are
registered and all service dependencies resolved by consul so every nodes will
look for any services in consul catalog. Before provisioning the nodes, a consul
discovery token is created with discovery service and shared to all nodes, which
will be used for consul server discovery (using dns), registering consul servers
with discovery server, and use this token as the consul datacenter name.

**Nodes provisioned with uesrdata which prepare the node to bootstrap**: 
Individual nodes are provisioned with userdata script which is run by cloudinit
on first boot. Cloudinit will do the basic system configuration like setting up
proxy, install/setup puppet and puppet manifests and modules, make sure swap
is added. It also configure consul agent in appropriate mode, setup dnsmasq so
dns query to consul agent will be forwarded  and install the packages which is
required for the node. Finally it create a cron job entry to run maybe_upgrade.sh.

The node bootstrap1 will bootstrap the consul cluster - consul agent in that
node is running in bootstrap server mode. Every other nodes will query dns to
see any consul server is exposed (by discovery server) and just wait till one
available for its own datacenter (datacenter name is equal to consul discovery
token) - This all happen on initial booting of the node, executed by cloudinit. 

*The complete consul cluster is formed during initial node boot.*

Note: We install the packages required for the node  is just to reduce total
bootstrap time, there is no other reason for it. This step can be done on
standard puppet run also.

**maybe_upgrade.sh takeover the control**: Once cloudinit prepare the system for
bootstrap, and setup consul cluster, maybe_upgrade.sh which run as  cronjob will
take the deployment further.

maybe_upgrade.sh use python-jiocloud python module to do orchestration. It first
check if there any updates available to the system, which in fact check consul
KV store for the same. So in fact, an upgrade can be triggered by just
incrementing consul kv "/current_verion" which is done by python jiocloud
trigger_update. 

maybe_upgrade.sh does:

* Run validation tests installed under /usr/lib/jiocloud/tests every execution.
* In case of validation failure and/or local health failures, it run puppet to
fix the failures
* Run apt-get update, apt-get upgrade in case of update available and on
initialization/bootstrap.
* Run puppet after apt-get upgrade
* It update consul KV with its statuses - puppet and validation status
* Update /etc/current_version to the version which the system got upgraded
(updated local version)
* Finally it update consul KV for running versions (/running_version/<version
number>/<hostname>)

maybe_upgrade.sh will upgrade the nodes and run puppet. At this moment all nodes
query consul - either to dns or rest api interface for any service dependencies
and to get various hostnames or IP address to be configured for various
services.

We implemented two helper methods in puppet, using which the puppet code will
resolve inter-host service dependencies:
 
1. to help puppet to stop further execution on configuring a resources until the
service dependencies available (on consul).
2. To help puppet to just fail the dependent resources to be configured until
the service dependencies available.

Every node knows what it need to do and what are their dependencies. All
resource configurations which have any dependencies will either wait or fail
till the dependencies got up in consul. 

All resources which have satisfied the service requirements (and which dont have
any dependencies) will just configure themselves, validation scripts to validate
them, and register themselves to consul with a healthcheck script. Consul then
do healthcheck (it keep on checking the health on mostly every 10 seconds), for
the service and make it available in consul service catalog which will be
available for other services/dependants to use. 

Consul also run the healthcheck in every check duration (which is every 10
seconds), and in case of failure it will remove that service instance from the
catalog. So consul will only expose healthy service instances to others.
Sometimes consul is asked to just wait to get a service health status from
external systems (ttl checks) and fail in case it did not get any response for
certain duration (equal to ttl value).

Jiocloud system will be setup in two phases - 

1. bootstrapping consul by cloudinit coded in userdata
2. bootstrapping rest of the jiocloud system by puppet run by maybe_upgrade

In current jiocloud system, haproxy, mysql, stmonleader, and ct (and ofcourse bootstrap
node which is setup in phase #1 itself), dont have any dependencies and they
setup themselves and install validation scripts, and register themselves to
consul with a healthcheck script. 

Stmon and st are depend on stmonleader to initialize ceph cluster, and once it
is available, they start configuring themsleves (both ceph mon, osd, and
radosgw), and register appropriate services in consul.

Once radosgw is up in consul service catalog, haproxy will take it and configure
radosgw service with available backend radosgw servers.

keystone on oc and ocdb node was waiting for mysql service and once it is up, it
will configure itself, and register keystone service to consul. Eventually
haproxy detect availability of keystone service and configure itself for
keystone service.

Glance would be waiting for its deps - mysql, keystone and ceph cluster once
they are up, glance will be started and registered in consul and eventually
haproxy detect that service and configure itself with glance api service.

Cinder is waiting for mysql, keystone, glance and ceph cluster and once they are
up, cinder services will be setup and registered with consul and then haproxy
configured for cinder.

ct will configure contrail services and neutron, register neutron with consul.
Also haproxy detect neutron availability and configure itself with neutron.

Nova controller servies which were waiting for its dependencies - mysql,
keystone, glance, cinder, neutron and once up, the will configure themseleves
and register appropriate services on consul and haproxy detect those services
and configure haproxy to include them in LB.

Puppet will get certain openstack services (nova api, nova scheduler, nova
compute, cinder scheduler, cinder volume etc) from consul and create
/etc/matchmaker/matchmaker.json file which is used by zmq driver to send the
messages to appropriate topics/nodes.

Cp and gcp nodes dependent on ceph mons, so once they are up configure
themselves and register in consul.
