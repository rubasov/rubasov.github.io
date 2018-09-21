---
layout: post
title: "Guaranteed Minimum Bandwidth in OpenStack: a Preview"
date: 2018-09-21 13:53:20+02:00
tags: openstack placement qos
published: yes
---
## Intro

This post is a tech preview
of the *guaranteed minimum bandwidth* feature of OpenStack
in its current (as of September 2018) work-in-progress status.
It is also a written record of the demo
we gave to fellow OpenStack developers in Denver, Colorado on the
[OpenStack Project Teams Gathering (PTG)](https://www.openstack.org/ptg)
on September 13th, 2018.

For those unfamiliar with the specifications,
this feature allows the OpenStack user
to place (a.k.a. schedule) virtual machines (servers in nova parlance)
according to the availability of resources
like the bandwidth of the physical network interface card of a compute host.
Ideally such a guarantee should be enforced
both at the placement of virtual machines onto compute hosts
and also in the traffic of a physical network interface card.
This work is scoped to cover only the enforcement at virtual machine placement.
However there is support available for data plane enforcement
since the Newton release of OpenStack for
[egress traffic on SR-IOV cards](https://specs.openstack.org/openstack/neutron-specs/specs/newton/ml2-qos-minimum-egress-bw-support.html).
Hopefully we will also have other networking backends supporting
data plane enforcement soon.

Despite giving an atypical session at the PTG
we had a suprisingly large audience at the demo
and very positive feedback - thank you all.
The primary reason for the big interest was - I guess -
because this feature is the first major user
of the new Placement service of OpenStack and nested resource providers
outside of Nova (and to some extent Neutron routed networks),
therefore it is one of the first to prove
the viability of cross-component resource management in OpenStack.

Beware, what follows is work-in-progress.
This is not documentation of an upcoming feature at this point.
We hope to merge a usable,
but likely not yet fully fledged version
to the Stein release of OpenStack scheduled for April 2019.
All code exercised here is available publicly,
but a large chunk of it is still under review.
However the results below should be reproducible by anyone.

While I list the patches used to put together this environment
at the end of this post,
the main reason for that is reproducibility.
If you want to try this code
I encourage you take the most recent version of the changes provided
and give us feedback based on that.

Please also note that we'll have
[another presentation in November 2018 in Berlin](https://www.openstack.org/summit/berlin-2018/summit-schedule/events/21987/guaranteed-minimum-bandwidth-feature-demo)
on the OpenStack Summit that'll be bigger, better and more complete.
That is meant to cater to other audiences too.
Not only developers, but also cloud operators and users.

Before jumping into the gory details let me say thanks to my colleagues
Gibi and Lajos Katona, and all the Nova, Neutron and Placement developers
whose work we're building on.

## Demo

For the sake of simplicity we work as admin,
of course that's not how this will be used in real life.

```
$ ./stack.sh
$ source openrc admin admin
```

The first new Neutron API extension marks the new port attribute ```resource_request```.

```
$ openstack extension show port-resource-request
+-------------+---------------------------------+
| Field       | Value                           |
+-------------+---------------------------------+
| alias       | port-resource-request           |
| description | Expose resource request to Port |
| id          | port-resource-request           |
| links       | []                              |
| location    | None                            |
| name        | Port Resource Request           |
| updated_at  | 2018-05-08T10:00:00-00:00       |
+-------------+---------------------------------+
```

The second new Neutron API extension marks the new direction ```ingress```
for ```minimum-bandwidth``` type QoS rules.

```
$ openstack extension show qos-bw-minimum-ingress
+-------------+-----------------------------------------------------------------------+
| Field       | Value                                                                 |
+-------------+-----------------------------------------------------------------------+
| alias       | qos-bw-minimum-ingress                                                |
| description | Allow to configure QoS minumum bandwidth rule with ingress direction. |
| id          | qos-bw-minimum-ingress                                                |
| links       | []                                                                    |
| location    | None                                                                  |
| name        | Ingress direction for QoS minimum bandwidth rule                      |
| updated_at  | 2018-07-09T10:00:00-00:00                                             |
+-------------+-----------------------------------------------------------------------+
```

We are running with support up to microversion
[```1.29```](https://docs.openstack.org/nova/latest/user/placement.html#support-allocation-candidates-with-nested-resource-providers)
of the Placement API, that is actually required for this feature.

```
$ export TOKEN="$( openstack token issue -f value -c id )"
$ curl \
    --silent \
    --header "Accept: application/json" \
    --header "Content-Type: application/json" \
    --header "X-Auth-Token: $TOKEN" \
    --header "OpenStack-API-Version: placement latest" \
    'http://127.0.0.1/placement/' \
  | json_pp
{
   "versions" : [
      {
         "max_version" : "1.29",
         "status" : "CURRENT",
         "min_version" : "1.0",
         "links" : [
            {
               "href" : "",
               "rel" : "self"
            }
         ],
         "id" : "v1.0"
      }
   ]
}
```

Neutron server is configured to reach the Placement API.

```
neutron.conf:

[placement]
project_domain_name = Default
project_name = service
user_domain_name = Default
password = devstack
username = nova
auth_url = http://127.0.0.1/identity
auth_type = password
```

Neutron agents ovs and sr-iov are configured
to report the bandwidth available
on the physical network interfaces/bridges they manage (in kilobit per sec).
The format is ```INTERFACE:EGRESS-TOTAL-BW:INGRESS-TOTAL-BW```.
The numbers here are made up.

```
ml2_conf.ini:

[ovs]
resource_provider_bandwidths = br-test:1500:1500,br-ex::1
bridge_mappings = public:br-ex,physnet0:br-test
```

```
sriov_agent.ini:

[sriov_nic]
physical_device_mappings = physnet0:ens5
resource_provider_bandwidths = ens5:100000:100000
```

Neutron agents running. We'll work with Open vSwitch agent and NIC Switch
agent (ie. neutron-sriov-agent).

```
$ openstack network agent list
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 121b9246-3309-4fa9-99ac-5eefe355c761 | L3 agent           | devstack0 | nova              | :-)   | UP    | neutron-l3-agent          |
| 4ad98428-9d0c-40e4-add9-bb7aa94acc9d | NIC Switch agent   | devstack0 | None              | :-)   | UP    | neutron-sriov-nic-agent   |
| 88134cd2-401e-4823-9652-aace67b2419a | DHCP agent         | devstack0 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| c589094a-d3a0-49a0-912c-a7747b91d1ec | Metadata agent     | devstack0 | None              | :-)   | UP    | neutron-metadata-agent    |
| de263c2b-0033-4539-97fb-86385a00cc7e | Open vSwitch agent | devstack0 | None              | :-)   | UP    | neutron-openvswitch-agent |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
```

The configuration of neutron-ovs-agent as reported to neutron-server
in the agent heartbeat.
Please note the inclusion of
```bridge_mappings```, ```rp_bandwidths``` and ```rp_inventory_defaults```.
We defaulted the latter to something sensible,
but we could have set it in the config file.
```rp_bandwidths``` controls the ```total``` of an inventory,
while other inventory parameters can be tuned via ```rp_inventory_defaults```.

```
$ openstack network agent list --host devstack0 --agent-type open-vswitch -f value -c ID \
  | xargs -r -n1 openstack network agent show -f value -c configuration \
  | python -c 'import pprint ; import sys ; pprint.pprint(eval(sys.stdin.readline()))'
{u'arp_responder_enabled': False,
 u'bridge_mappings': {u'physnet0': u'br-test', u'public': u'br-ex'},
 u'datapath_type': u'system',
 u'devices': 0,
 u'enable_distributed_routing': False,
 u'extensions': [u'qos'],
 u'in_distributed_mode': False,
 u'l2_population': False,
 u'log_agent_heartbeats': False,
 u'ovs_capabilities': {u'datapath_types': [u'netdev', u'system'],
                       u'iface_types': [u'geneve',
                                        u'gre',
                                        u'internal',
                                        u'lisp',
                                        u'patch',
                                        u'stt',
                                        u'system',
                                        u'tap',
                                        u'vxlan']},
 u'ovs_hybrid_plug': False,
 u'rp_bandwidths': {u'br-ex': {u'egress': None, u'ingress': 1},
                    u'br-test': {u'egress': 1500, u'ingress': 1500}},
 u'rp_inventory_defaults': {u'allocation_ratio': 1.0,
                            u'min_unit': 1,
                            u'reserved': 0,
                            u'step_size': 1},
 u'tunnel_types': [u'vxlan'],
 u'tunneling_ip': u'192.168.2.169',
 u'vhostuser_socket_dir': u'/var/run/openvswitch'}
```

The configuration of neutron-sriov-agent as reported to neutron-server
in the agent heartbeat.
Instead of ```bridge_mappings``` look out for ```device_mappings```.

```
$ openstack network agent list --host devstack0 --agent-type nic -f value -c ID \
  | xargs -r -n1 openstack network agent show -f value -c configuration \
  | python -c 'import pprint ; import sys ; pprint.pprint(eval(sys.stdin.readline()))'
{u'device_mappings': {u'physnet0': [u'ens5']},
 u'devices': 0,
 u'extensions': [u'qos'],
 u'rp_bandwidths': {u'ens5': {u'egress': 100000, u'ingress': 100000}},
 u'rp_inventory_defaults': {u'allocation_ratio': 1.0,
                            u'min_unit': 1,
                            u'reserved': 0,
                            u'step_size': 1}}
```

The resource information provided in the agent config files
is replicated from Neutron agents to Neutron server to Placement,
so the whole system has a consistent view of it.
At this point the replication is complete,
first I show the end results.

Custom traits to represent supported vnic_types and physical interfaces
being connected to physnets.
This is needed because Placement must take over some of Neutron's
port binding logic, so Neutron can influence virtual machine scheduling.

The vnic_type traits are to be standardized in
[os-traits](https://github.com/openstack/os-traits).

```
$ openstack --os-placement-api-version 1.17 trait list \
  | awk '/CUSTOM_/ { print $2 }' \
  | sort
CUSTOM_PHYSNET_PHYSNET0
CUSTOM_PHYSNET_PUBLIC
CUSTOM_VNIC_TYPE_DIRECT
CUSTOM_VNIC_TYPE_DIRECT_PHYSICAL
CUSTOM_VNIC_TYPE_MACVTAP
CUSTOM_VNIC_TYPE_NORMAL
```

The UUIDs of the new resource providers are deterministic v5 UUIDs:
* for agents: ```uuid5(host, agent_type)```
* for physical network interfaces: ```uuid5(host, agent_type, interface)```

```
$ openstack --os-placement-api-version 1.17 resource provider list
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
| uuid                                 | name                                 | generation | root_provider_uuid                   | parent_provider_uuid                 |
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
| 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | devstack0                            |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | None                                 |
| 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 | devstack0:NIC Switch agent           |          0 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| 1c7e83f0-108d-5c35-ada7-7ebebbe43aad | devstack0:NIC Switch agent:ens5      |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 |
| 89ca1421-5117-5348-acab-6d0e2054239c | devstack0:Open vSwitch agent         |          0 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| f9c9ce07-679d-5d72-ac5f-31720811629a | devstack0:Open vSwitch agent:br-test |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 89ca1421-5117-5348-acab-6d0e2054239c |
| 193134fd-464c-5545-9d20-df7d58c0166f | devstack0:Open vSwitch agent:br-ex   |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 89ca1421-5117-5348-acab-6d0e2054239c |
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
```

For the visually oriented the resource providers are structured in a tree,
that needs support for
[nested resource providers](https://specs.openstack.org/openstack/nova-specs/specs/ocata/approved/nested-resource-providers.html).

![resource provider tree](/assets/img/post/openstack-qos-min-bw-demo-rp-tree.svg)

Agent resource providers have no traits or inventories,
therefore everything about them can be seen already in the table above.
Physical network interface resource providers are embellished properly.

```
$ for rp in $( openstack --os-placement-api-version 1.17 resource provider list \
  | awk '/devstack0:(NIC |Open v)Switch agent:/ { print $2 }' )
  do
      echo BEGIN RP $rp
      openstack --os-placement-api-version 1.17 resource provider show "$rp"
      openstack --os-placement-api-version 1.17 resource provider trait list "$rp"
      openstack --os-placement-api-version 1.17 resource provider inventory list "$rp"
      echo END RP $rp
      echo
  done
BEGIN RP 1c7e83f0-108d-5c35-ada7-7ebebbe43aad
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| uuid                 | 1c7e83f0-108d-5c35-ada7-7ebebbe43aad |
| name                 | devstack0:NIC Switch agent:ens5      |
| generation           | 2                                    |
| root_provider_uuid   | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| parent_provider_uuid | 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 |
+----------------------+--------------------------------------+
+----------------------------------+
| name                             |
+----------------------------------+
| CUSTOM_PHYSNET_PHYSNET0          |
| CUSTOM_VNIC_TYPE_DIRECT          |
| CUSTOM_VNIC_TYPE_MACVTAP         |
| CUSTOM_VNIC_TYPE_DIRECT_PHYSICAL |
+----------------------------------+
+-------------------------------------------+------------------+------------+----------+-----------+----------+--------+
| resource_class                            | allocation_ratio |   max_unit | reserved | step_size | min_unit |  total |
+-------------------------------------------+------------------+------------+----------+-----------+----------+--------+
| NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND  |              1.0 | 2147483647 |        0 |         1 |        1 | 100000 |
| NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND |              1.0 | 2147483647 |        0 |         1 |        1 | 100000 |
+-------------------------------------------+------------------+------------+----------+-----------+----------+--------+
END RP 1c7e83f0-108d-5c35-ada7-7ebebbe43aad

BEGIN RP f9c9ce07-679d-5d72-ac5f-31720811629a
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| uuid                 | f9c9ce07-679d-5d72-ac5f-31720811629a |
| name                 | devstack0:Open vSwitch agent:br-test |
| generation           | 2                                    |
| root_provider_uuid   | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| parent_provider_uuid | 89ca1421-5117-5348-acab-6d0e2054239c |
+----------------------+--------------------------------------+
+-------------------------+
| name                    |
+-------------------------+
| CUSTOM_PHYSNET_PHYSNET0 |
| CUSTOM_VNIC_TYPE_DIRECT |
| CUSTOM_VNIC_TYPE_NORMAL |
+-------------------------+
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
| resource_class                            | allocation_ratio |   max_unit | reserved | step_size | min_unit | total |
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
| NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND  |              1.0 | 2147483647 |        0 |         1 |        1 |  1500 |
| NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND |              1.0 | 2147483647 |        0 |         1 |        1 |  1500 |
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
END RP f9c9ce07-679d-5d72-ac5f-31720811629a

BEGIN RP 193134fd-464c-5545-9d20-df7d58c0166f
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| uuid                 | 193134fd-464c-5545-9d20-df7d58c0166f |
| name                 | devstack0:Open vSwitch agent:br-ex   |
| generation           | 2                                    |
| root_provider_uuid   | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| parent_provider_uuid | 89ca1421-5117-5348-acab-6d0e2054239c |
+----------------------+--------------------------------------+
+-------------------------+
| name                    |
+-------------------------+
| CUSTOM_VNIC_TYPE_DIRECT |
| CUSTOM_PHYSNET_PUBLIC   |
| CUSTOM_VNIC_TYPE_NORMAL |
+-------------------------+
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
| resource_class                            | allocation_ratio |   max_unit | reserved | step_size | min_unit | total |
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
| NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND |              1.0 | 2147483647 |        0 |         1 |        1 |     1 |
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
END RP 193134fd-464c-5545-9d20-df7d58c0166f
```

Currently it is a problem for us that multiple Neutron ML2 mechanism drivers
report support for the direct vnic_type.
See
[this code comment](https://review.openstack.org/#/c/580672/9/neutron/services/placement_report/plugin.py@118)
and
[that change](https://review.openstack.org/601600).

Resource information can be re-synchronized if needed.
To prove it, I'll delete resource providers (including their trait
associations and inventories) belonging to neutron-ovs-agent.

```
$ openstack --os-placement-api-version 1.17 resource provider list \
  | awk '/devstack0:Open vSwitch agent/ { print $2 }' \
  | tac \
  | xargs -r -n1 openstack --os-placement-api-version 1.17 resource provider delete
```

The remaining resource providers.

```
$ openstack --os-placement-api-version 1.17 resource provider list
+--------------------------------------+---------------------------------+------------+--------------------------------------+--------------------------------------+
| uuid                                 | name                            | generation | root_provider_uuid                   | parent_provider_uuid                 |
+--------------------------------------+---------------------------------+------------+--------------------------------------+--------------------------------------+
| 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | devstack0                       |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | None                                 |
| 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 | devstack0:NIC Switch agent      |          0 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| 1c7e83f0-108d-5c35-ada7-7ebebbe43aad | devstack0:NIC Switch agent:ens5 |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 |
+--------------------------------------+---------------------------------+------------+--------------------------------------+--------------------------------------+
```

Trigger re-sync by agent restart.
There are multiple triggers, config changes should be picked up automatically.

```
$ sudo systemctl restart devstack@neutron-agent
```

Neutron server debug logs show the placement client calls to re-create
all placement objects.

```
$ sudo journalctl -u devstack@neutron-api --since '10 minutes ago' | egrep 'DEBUG.*placement client:'
szept 19 15:49:48 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_trait(name=u'CUSTOM_PHYSNET_PHYSNET0') {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:48 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_trait(name=u'CUSTOM_PHYSNET_PUBLIC') {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:48 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_trait(name='CUSTOM_VNIC_TYPE_NORMAL') {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:48 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_trait(name='CUSTOM_VNIC_TYPE_DIRECT') {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:48 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: ensure_resource_provider(resource_provider={'name': u'devstack0:Open vSwitch agent', 'parent_provider_uuid': u'3b36d91e-bf60-460f-b1f8-3322dee5cdfd', 'uuid': UUID('89ca1421-5117-5348-acab-6d0e2054239c')}) {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:48 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: ensure_resource_provider(resource_provider={'name': u'devstack0:Open vSwitch agent:br-test', 'parent_provider_uuid': UUID('89ca1421-5117-5348-acab-6d0e2054239c'), 'uuid': UUID('f9c9ce07-679d-5d72-ac5f-31720811629a')}) {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:48 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: ensure_resource_provider(resource_provider={'name': u'devstack0:Open vSwitch agent:br-ex', 'parent_provider_uuid': UUID('89ca1421-5117-5348-acab-6d0e2054239c'), 'uuid': UUID('193134fd-464c-5545-9d20-df7d58c0166f')}) {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:49 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_resource_provider_traits(resource_provider_uuid=UUID('f9c9ce07-679d-5d72-ac5f-31720811629a'), traits=[u'CUSTOM_PHYSNET_PHYSNET0', 'CUSTOM_VNIC_TYPE_NORMAL', 'CUSTOM_VNIC_TYPE_DIRECT']) {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:49 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_resource_provider_traits(resource_provider_uuid=UUID('193134fd-464c-5545-9d20-df7d58c0166f'), traits=[u'CUSTOM_PHYSNET_PUBLIC', 'CUSTOM_VNIC_TYPE_NORMAL', 'CUSTOM_VNIC_TYPE_DIRECT']) {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:49 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_resource_provider_inventories(resource_provider_uuid=UUID('f9c9ce07-679d-5d72-ac5f-31720811629a'), inventories={'NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND': {'total': 1500, u'min_unit': 1, u'allocation_ratio': 1.0, u'step_size': 1, u'reserved': 0}, 'NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND': {'total': 1500, u'min_unit': 1, u'allocation_ratio': 1.0, u'step_size': 1, u'reserved': 0}}) {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
szept 19 15:49:49 devstack0 neutron-server[6919]: DEBUG neutron.services.placement_report.plugin [-] placement client: update_resource_provider_inventories(resource_provider_uuid=UUID('193134fd-464c-5545-9d20-df7d58c0166f'), inventories={'NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND': {'total': 1, u'min_unit': 1, u'allocation_ratio': 1.0, u'step_size': 1, u'reserved': 0}}) {{(pid=7143) _execute_deferred /opt/stack/neutron/neutron/services/placement_report/plugin.py:63}}
```

At demo time there was a bug here
(no fault, but producing noise in the logs)
that's already supposed to be fixed
[in this change](https://review.openstack.org/599376).

All resource providers re-created (including trait associations and
inventories).

```
$ openstack --os-placement-api-version 1.17 resource provider list
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
| uuid                                 | name                                 | generation | root_provider_uuid                   | parent_provider_uuid                 |
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
| 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | devstack0                            |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | None                                 |
| 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 | devstack0:NIC Switch agent           |          0 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| 1c7e83f0-108d-5c35-ada7-7ebebbe43aad | devstack0:NIC Switch agent:ens5      |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 |
| 89ca1421-5117-5348-acab-6d0e2054239c | devstack0:Open vSwitch agent         |          0 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| f9c9ce07-679d-5d72-ac5f-31720811629a | devstack0:Open vSwitch agent:br-test |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 89ca1421-5117-5348-acab-6d0e2054239c |
| 193134fd-464c-5545-9d20-df7d58c0166f | devstack0:Open vSwitch agent:br-ex   |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 89ca1421-5117-5348-acab-6d0e2054239c |
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
```

Setting up the environment to boot a virtual machine.
Will not use all in this demo.

```
$ cat local.sh
set -x

openstack network create net0 \
    --provider-network-type vlan \
    --provider-physical-network physnet0 \
    --provider-segment 100 \
    ##
openstack subnet create subnet0 \
    --network net0 \
    --subnet-range 10.0.4.0/24 \
    ##

openstack network qos policy create qp0

openstack network qos rule create qp0 \
    --type minimum-bandwidth \
    --min-kbps 1000 \
    --egress \
    ##
openstack network qos rule create qp0 \
    --type minimum-bandwidth \
    --min-kbps 1000 \
    --ingress \
    ##

openstack port create port-normal \
    --network net0 \
    --vnic-type normal \
    ##
openstack port create port-normal-qos \
    --network net0 \
    --vnic-type normal \
    --qos-policy qp0 \
    ##

openstack port create port-direct \
    --network net0 \
    --vnic-type direct \
    ##
openstack port create port-direct-qos \
    --network net0 \
    --vnic-type direct \
    --qos-policy qp0 \
    ##
```

```net0``` is a provider network on ```physnet0```.

```
$ openstack network show net0
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2018-09-17T14:05:35Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e55618be-7d30-4623-8c1f-f7d85738feab |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | net0                                 |
| port_security_enabled     | True                                 |
| project_id                | 30effa46196843479fc533034de8e955     |
| provider:network_type     | vlan                                 |
| provider:physical_network | physnet0                             |
| provider:segmentation_id  | 100                                  |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 225159c4-2447-4347-a8a8-a3ad6142cb8d |
| tags                      |                                      |
| updated_at                | 2018-09-17T14:05:37Z                 |
+---------------------------+--------------------------------------+
```

We have one QoS policy.

```
$ openstack network qos policy list
+--------------------------------------+------+--------+---------+----------------------------------+
| ID                                   | Name | Shared | Default | Project                          |
+--------------------------------------+------+--------+---------+----------------------------------+
| 5c543f8c-d513-48bb-a6e4-602eb728209a | qp0  | False  | False   | 30effa46196843479fc533034de8e955 |
+--------------------------------------+------+--------+---------+----------------------------------+
```

With two ```minimum_bandwidth``` rules for egress and ingress directions.
Directions are meant from VM perspective.

```
$ openstack network qos rule list qp0
+--------------------------------------+--------------------------------------+-------------------+----------+-----------------+----------+-----------+-----------+
| ID                                   | QoS Policy ID                        | Type              | Max Kbps | Max Burst Kbits | Min Kbps | DSCP mark | Direction |
+--------------------------------------+--------------------------------------+-------------------+----------+-----------------+----------+-----------+-----------+
| dea6c72e-fd80-44fb-8975-949d88bf1007 | 5c543f8c-d513-48bb-a6e4-602eb728209a | minimum_bandwidth |          |                 |     1000 |           | egress    |
| 74048477-5fe3-43a4-a678-39c154e9ed97 | 5c543f8c-d513-48bb-a6e4-602eb728209a | minimum_bandwidth |          |                 |     1000 |           | ingress   |
+--------------------------------------+--------------------------------------+-------------------+----------+-----------------+----------+-----------+-----------+
```

*normal* and *direct* in port names stand for the vnic_type of those ports.
We have ports with and without a QoS policy associated.

```
$ openstack port list
+--------------------------------------+-----------------+-------------------+-----------------------------------------------------------------------------------------------------+--------+
| ID                                   | Name            | MAC Address       | Fixed IP Addresses                                                                                  | Status |
+--------------------------------------+-----------------+-------------------+-----------------------------------------------------------------------------------------------------+--------+
| 307bec3e-8de5-4874-8797-7e551a2d1fdc | port-direct     | fa:16:3e:4f:bd:71 | ip_address='10.0.4.22', subnet_id='225159c4-2447-4347-a8a8-a3ad6142cb8d'                            | DOWN   |
| 658d2094-f837-4017-951a-6a9b4396483d | port-normal     | fa:16:3e:10:d2:f9 | ip_address='10.0.4.5', subnet_id='225159c4-2447-4347-a8a8-a3ad6142cb8d'                             | DOWN   |
| 99a97b12-6364-4d10-8d8e-c3dab4161691 | port-normal-qos | fa:16:3e:10:08:95 | ip_address='10.0.4.15', subnet_id='225159c4-2447-4347-a8a8-a3ad6142cb8d'                            | DOWN   |
| fbf69868-29e7-42c9-8ace-c9fd9ee9583d | port-direct-qos | fa:16:3e:6f:d3:ec | ip_address='10.0.4.10', subnet_id='225159c4-2447-4347-a8a8-a3ad6142cb8d'                            | DOWN   |
  ...
+--------------------------------------+-----------------+-------------------+-----------------------------------------------------------------------------------------------------+--------+
```

Ports associated with ```minimum_bandwidth``` QoS rules expose their resource
needs (including resource types, amounts and traits) to Nova
by the new ```resource_request``` read-only and admin-only attribute.
At the moment Neutron client is used here because openstack client
is not yet updated to display these new port attributes.
The resource request of a port is meant to be incorporated into the bigger
resource request of a virtual machine as a
[granular resource request group](https://specs.openstack.org/openstack/nova-specs/specs/queens/approved/granular-resource-requests.html).

```
$ neutron port-show port-normal-qos
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                                                                                    |
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                                                                                                     |
| allowed_address_pairs |                                                                                                                                                                                          |
| binding:host_id       |                                                                                                                                                                                          |
| binding:profile       | {}                                                                                                                                                                                       |
| binding:vif_details   | {}                                                                                                                                                                                       |
| binding:vif_type      | unbound                                                                                                                                                                                  |
| binding:vnic_type     | normal                                                                                                                                                                                   |
| created_at            | 2018-09-17T14:05:46Z                                                                                                                                                                     |
| description           |                                                                                                                                                                                          |
| device_id             |                                                                                                                                                                                          |
| device_owner          |                                                                                                                                                                                          |
| extra_dhcp_opts       |                                                                                                                                                                                          |
| fixed_ips             | {"subnet_id": "225159c4-2447-4347-a8a8-a3ad6142cb8d", "ip_address": "10.0.4.15"}                                                                                                         |
| id                    | 99a97b12-6364-4d10-8d8e-c3dab4161691                                                                                                                                                     |
| ip_allocation         | immediate                                                                                                                                                                                |
| mac_address           | fa:16:3e:10:08:95                                                                                                                                                                        |
| name                  | port-normal-qos                                                                                                                                                                          |
| network_id            | e55618be-7d30-4623-8c1f-f7d85738feab                                                                                                                                                     |
| port_security_enabled | True                                                                                                                                                                                     |
| project_id            | 30effa46196843479fc533034de8e955                                                                                                                                                         |
| qos_policy_id         | 5c543f8c-d513-48bb-a6e4-602eb728209a                                                                                                                                                     |
| resource_request      | {"required": ["CUSTOM_PHYSNET_PHYSNET0", "CUSTOM_VNIC_TYPE_NORMAL"], "resources": {"NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND": 1000, "NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND": 1000}} |
| revision_number       | 2                                                                                                                                                                                        |
| security_groups       | b1293461-e193-4958-b448-136ecaf0a374                                                                                                                                                     |
| status                | DOWN                                                                                                                                                                                     |
| tags                  |                                                                                                                                                                                          |
| tenant_id             | 30effa46196843479fc533034de8e955                                                                                                                                                         |
| updated_at            | 2018-09-17T14:05:46Z                                                                                                                                                                     |
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

To recap we reported 1500 kbps total bandwidth available
for one of the physical network interfaces,
while one port like above requests 1000 kbps of bandwidth.
I chose these numbers to make the first request succeed
and the second to fail because of insufficient resources.

```
$ openstack --os-placement-api-version 1.17 resource provider list \
  | awk '/devstack0:Open vSwitch agent:br-test/ { print $2 }' \
  | xargs -r -n1 openstack resource provider inventory list
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
| resource_class                            | allocation_ratio |   max_unit | reserved | step_size | min_unit | total |
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
| NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND  |              1.0 | 2147483647 |        0 |         1 |        1 |  1500 |
| NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND |              1.0 | 2147483647 |        0 |         1 |        1 |  1500 |
+-------------------------------------------+------------------+------------+----------+-----------+----------+-------+
```

You may want to see a regression test of booting a virtual machine
without requesting guaranteed bandwidth.

```
$ openstack server create --flavor cirros256 --image cirros-0.3.5-x86_64-disk --nic port-id=port-normal --wait vm0
# check if it went to ACTIVE
$ openstack server delete vm0
```

Then move on to boot a virtual machine requesting guaranteed bandwidth.

```
$ openstack server create --flavor cirros256 --image cirros-0.3.5-x86_64-disk --nic port-id=port-normal-qos --wait vm0

+-------------------------------------+-----------------------------------------------------------------+
| Field                               | Value                                                           |
+-------------------------------------+-----------------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                          |
| OS-EXT-AZ:availability_zone         | nova                                                            |
| OS-EXT-SRV-ATTR:host                | devstack0                                                       |
| OS-EXT-SRV-ATTR:hypervisor_hostname | devstack0                                                       |
| OS-EXT-SRV-ATTR:instance_name       | instance-00000002                                               |
| OS-EXT-STS:power_state              | Running                                                         |
| OS-EXT-STS:task_state               | None                                                            |
| OS-EXT-STS:vm_state                 | active                                                          |
| OS-SRV-USG:launched_at              | 2018-09-19T16:00:25.000000                                      |
| OS-SRV-USG:terminated_at            | None                                                            |
| accessIPv4                          |                                                                 |
| accessIPv6                          |                                                                 |
| addresses                           | net0=10.0.4.15                                                  |
| adminPass                           | CCj9btyYGjub                                                    |
| config_drive                        |                                                                 |
| created                             | 2018-09-19T16:00:17Z                                            |
| flavor                              | cirros256 (c1)                                                  |
| hostId                              | 88fd036bb84cf5090e882d262d8b1b2f9d9566e87ba7f2816f9a0b97        |
| id                                  | 71cb7e04-3d5a-40f9-a99f-07f0d0831741                            |
| image                               | cirros-0.3.5-x86_64-disk (e3062404-1006-4d62-b595-ba1bd752ce0a) |
| key_name                            | None                                                            |
| name                                | vm0                                                             |
| progress                            | 0                                                               |
| project_id                          | 30effa46196843479fc533034de8e955                                |
| properties                          |                                                                 |
| security_groups                     | name='default'                                                  |
| status                              | ACTIVE                                                          |
| updated                             | 2018-09-19T16:00:26Z                                            |
| user_id                             | a5e21a1f5ddf478c85b0297476e9aad8                                |
| volumes_attached                    |                                                                 |
+-------------------------------------+-----------------------------------------------------------------+
```

After nova makes the allocation of resources requested for a virtual machine
atomically in Placement it finds the resource provider allocated for the
Neutron port and tells that to Neutron via setting
```binding:profile.allocation``` to the UUID of the chosen resource provider.

```
$ neutron port-show port-normal-qos
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                                                                                    |
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                                                                                                     |
| allowed_address_pairs |                                                                                                                                                                                          |
| binding:host_id       | devstack0                                                                                                                                                                                |
| binding:profile       | {"allocation": "f9c9ce07-679d-5d72-ac5f-31720811629a"}                                                                                                                                   |
| binding:vif_details   | {"port_filter": true, "datapath_type": "system", "ovs_hybrid_plug": false}                                                                                                               |
| binding:vif_type      | ovs                                                                                                                                                                                      |
| binding:vnic_type     | normal                                                                                                                                                                                   |
| created_at            | 2018-09-17T14:05:46Z                                                                                                                                                                     |
| description           |                                                                                                                                                                                          |
| device_id             | 71cb7e04-3d5a-40f9-a99f-07f0d0831741                                                                                                                                                     |
| device_owner          | compute:nova                                                                                                                                                                             |
| extra_dhcp_opts       |                                                                                                                                                                                          |
| fixed_ips             | {"subnet_id": "225159c4-2447-4347-a8a8-a3ad6142cb8d", "ip_address": "10.0.4.15"}                                                                                                         |
| id                    | 99a97b12-6364-4d10-8d8e-c3dab4161691                                                                                                                                                     |
| ip_allocation         | immediate                                                                                                                                                                                |
| mac_address           | fa:16:3e:10:08:95                                                                                                                                                                        |
| name                  | port-normal-qos                                                                                                                                                                          |
| network_id            | e55618be-7d30-4623-8c1f-f7d85738feab                                                                                                                                                     |
| port_security_enabled | True                                                                                                                                                                                     |
| project_id            | 30effa46196843479fc533034de8e955                                                                                                                                                         |
| qos_policy_id         | 5c543f8c-d513-48bb-a6e4-602eb728209a                                                                                                                                                     |
| resource_request      | {"required": ["CUSTOM_PHYSNET_PHYSNET0", "CUSTOM_VNIC_TYPE_NORMAL"], "resources": {"NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND": 1000, "NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND": 1000}} |
| revision_number       | 5                                                                                                                                                                                        |
| security_groups       | b1293461-e193-4958-b448-136ecaf0a374                                                                                                                                                     |
| status                | ACTIVE                                                                                                                                                                                   |
| tags                  |                                                                                                                                                                                          |
| tenant_id             | 30effa46196843479fc533034de8e955                                                                                                                                                         |
| updated_at            | 2018-09-19T16:00:25Z                                                                                                                                                                     |
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

To recall the resource providers, ```f9c9ce07``` is
```devstack0:Open vSwitch agent:br-test```.

```
$ openstack --os-placement-api-version 1.17 resource provider list
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
| uuid                                 | name                                 | generation | root_provider_uuid                   | parent_provider_uuid                 |
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
| 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | devstack0                            |          5 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | None                                 |
| 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 | devstack0:NIC Switch agent           |          0 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| 1c7e83f0-108d-5c35-ada7-7ebebbe43aad | devstack0:NIC Switch agent:ens5      |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 4a8a819d-61f9-5822-8c5c-3e9c7cb942d6 |
| 89ca1421-5117-5348-acab-6d0e2054239c | devstack0:Open vSwitch agent         |          0 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd |
| f9c9ce07-679d-5d72-ac5f-31720811629a | devstack0:Open vSwitch agent:br-test |          3 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 89ca1421-5117-5348-acab-6d0e2054239c |
| 193134fd-464c-5545-9d20-df7d58c0166f | devstack0:Open vSwitch agent:br-ex   |          2 | 3b36d91e-bf60-460f-b1f8-3322dee5cdfd | 89ca1421-5117-5348-acab-6d0e2054239c |
+--------------------------------------+--------------------------------------+------------+--------------------------------------+--------------------------------------+
```

The part of the resource allocation that belongs to
the physical network interface resource provider.

```
$ openstack --os-placement-api-version 1.17 resource provider list \
  | awk '/devstack0:Open vSwitch agent:br-test/ { print $2 }' \
  | xargs -r -n1 openstack resource provider show --allocations
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field       | Value                                                                                                                                                              |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| uuid        | f9c9ce07-679d-5d72-ac5f-31720811629a                                                                                                                               |
| name        | devstack0:Open vSwitch agent:br-test                                                                                                                               |
| generation  | 3                                                                                                                                                                  |
| allocations | {u'71cb7e04-3d5a-40f9-a99f-07f0d0831741': {u'resources': {u'NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND': 1000, u'NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND': 1000}}} |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

With only one allocation the resource usage total is the same.

```
$ openstack --os-placement-api-version 1.17 resource provider list | awk '/devstack0:Open vSwitch agent:br-test/ { print $2 }' | xargs -r -n1 openstack resource provider usage show
+-------------------------------------------+-------+
| resource_class                            | usage |
+-------------------------------------------+-------+
| NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND  |  1000 |
| NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND |  1000 |
+-------------------------------------------+-------+
```

Create a second port requesting another 1000 kbps of bandwidth.

```
$ openstack port create port-normal-qos-2 --network net0 --vnic-type normal --qos-policy qp0
+-----------------------+--------------------------------------------------------------------------+
| Field                 | Value                                                                    |
+-----------------------+--------------------------------------------------------------------------+
| admin_state_up        | UP                                                                       |
| allowed_address_pairs |                                                                          |
| binding_host_id       |                                                                          |
| binding_profile       |                                                                          |
| binding_vif_details   |                                                                          |
| binding_vif_type      | unbound                                                                  |
| binding_vnic_type     | normal                                                                   |
| created_at            | 2018-09-19T16:02:28Z                                                     |
| data_plane_status     | None                                                                     |
| description           |                                                                          |
| device_id             |                                                                          |
| device_owner          |                                                                          |
| dns_assignment        | None                                                                     |
| dns_domain            | None                                                                     |
| dns_name              | None                                                                     |
| extra_dhcp_opts       |                                                                          |
| fixed_ips             | ip_address='10.0.4.23', subnet_id='225159c4-2447-4347-a8a8-a3ad6142cb8d' |
| id                    | 8fb64520-6e31-4ad4-ae22-5b205b78c830                                     |
| mac_address           | fa:16:3e:7a:72:f7                                                        |
| name                  | port-normal-qos-2                                                        |
| network_id            | e55618be-7d30-4623-8c1f-f7d85738feab                                     |
| port_security_enabled | True                                                                     |
| project_id            | 30effa46196843479fc533034de8e955                                         |
| qos_policy_id         | 5c543f8c-d513-48bb-a6e4-602eb728209a                                     |
| revision_number       | 2                                                                        |
| security_group_ids    | b1293461-e193-4958-b448-136ecaf0a374                                     |
| status                | DOWN                                                                     |
| tags                  |                                                                          |
| trunk_details         | None                                                                     |
| updated_at            | 2018-09-19T16:02:28Z                                                     |
+-----------------------+--------------------------------------------------------------------------+
```

Try booting a virtual machine with it. Expect failure.

```
$ openstack server create --flavor cirros256 --image cirros-0.3.5-x86_64-disk --nic port-id=port-normal-qos-2 --wait vm1
Error creating server: vm1
Error creating server
```

Expecting reason ```No valid host```.

```
$ openstack server show vm1 -f value -c fault \
  | python -c 'import pprint ; import sys ; pprint.pprint(eval(sys.stdin.readline()))' \
  | sed -e 's/\\n/\n/g'
{u'code': 500,
 u'created': u'2018-09-19T16:02:54Z',
 u'details': u'  File "/opt/stack/nova/nova/conductor/manager.py", line 1227, in schedule_and_build_instances
    instance_uuids, return_alternates=True)
  File "/opt/stack/nova/nova/conductor/manager.py", line 728, in _schedule_instances
    return_alternates=return_alternates)
  File "/opt/stack/nova/nova/scheduler/utils.py", line 960, in wrapped
    return func(*args, **kwargs)
  File "/opt/stack/nova/nova/scheduler/client/__init__.py", line 53, in select_destinations
    instance_uuids, return_objects, return_alternates)
  File "/opt/stack/nova/nova/scheduler/client/__init__.py", line 37, in __run_method
    return getattr(self.instance, __name)(*args, **kwargs)
  File "/opt/stack/nova/nova/scheduler/client/query.py", line 42, in select_destinations
    instance_uuids, return_objects, return_alternates)
  File "/opt/stack/nova/nova/scheduler/rpcapi.py", line 158, in select_destinations
    return cctxt.call(ctxt, \'select_destinations\', **msg_args)
  File "/usr/local/lib/python2.7/dist-packages/oslo_messaging/rpc/client.py", line 179, in call
    retry=self.retry)
  File "/usr/local/lib/python2.7/dist-packages/oslo_messaging/transport.py", line 133, in _send
    retry=retry)
  File "/usr/local/lib/python2.7/dist-packages/oslo_messaging/_drivers/amqpdriver.py", line 584, in send
    call_monitor_timeout, retry=retry)
  File "/usr/local/lib/python2.7/dist-packages/oslo_messaging/_drivers/amqpdriver.py", line 575, in _send
    raise result
',
 u'message': u'No valid host was found. '}
```

In Nova debug logs we can see there were no allocation candidates in the
Placement response.

```
$ sudo journalctl -u devstack@n-sch --since '10 minutes ago' \
  | egrep 'Got no allocation candidates from the Placement API.'
szept 19 16:02:53 devstack0 nova-scheduler[17193]: DEBUG nova.scheduler.manager [None req-32866d5b-f474-480a-9db0-37f413c8af2c admin admin] Got no allocation candidates from the Placement API. This could be due to insufficient resources or a temporary occurrence as compute nodes start up. {{(pid=20493) select_destinations /opt/stack/nova/nova/scheduler/manager.py:150}}
```

Clean up.

```
$ openstack server delete vm1
$ openstack server delete vm0
```

Expecting ```binding:profile.allocation``` to be cleared.

```
$ neutron port-show port-normal-qos
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                                                                                    |
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                                                                                                     |
| allowed_address_pairs |                                                                                                                                                                                          |
| binding:host_id       |                                                                                                                                                                                          |
| binding:profile       | {}                                                                                                                                                                                       |
| binding:vif_details   | {}                                                                                                                                                                                       |
| binding:vif_type      | unbound                                                                                                                                                                                  |
| binding:vnic_type     | normal                                                                                                                                                                                   |
| created_at            | 2018-09-17T14:05:46Z                                                                                                                                                                     |
| description           |                                                                                                                                                                                          |
| device_id             |                                                                                                                                                                                          |
| device_owner          |                                                                                                                                                                                          |
| extra_dhcp_opts       |                                                                                                                                                                                          |
| fixed_ips             | {"subnet_id": "225159c4-2447-4347-a8a8-a3ad6142cb8d", "ip_address": "10.0.4.15"}                                                                                                         |
| id                    | 99a97b12-6364-4d10-8d8e-c3dab4161691                                                                                                                                                     |
| ip_allocation         | immediate                                                                                                                                                                                |
| mac_address           | fa:16:3e:10:08:95                                                                                                                                                                        |
| name                  | port-normal-qos                                                                                                                                                                          |
| network_id            | e55618be-7d30-4623-8c1f-f7d85738feab                                                                                                                                                     |
| port_security_enabled | True                                                                                                                                                                                     |
| project_id            | 30effa46196843479fc533034de8e955                                                                                                                                                         |
| qos_policy_id         | 5c543f8c-d513-48bb-a6e4-602eb728209a                                                                                                                                                     |
| resource_request      | {"required": ["CUSTOM_PHYSNET_PHYSNET0", "CUSTOM_VNIC_TYPE_NORMAL"], "resources": {"NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND": 1000, "NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND": 1000}} |
| revision_number       | 6                                                                                                                                                                                        |
| security_groups       | b1293461-e193-4958-b448-136ecaf0a374                                                                                                                                                     |
| status                | DOWN                                                                                                                                                                                     |
| tags                  |                                                                                                                                                                                          |
| tenant_id             | 30effa46196843479fc533034de8e955                                                                                                                                                         |
| updated_at            | 2018-09-19T16:05:11Z                                                                                                                                                                     |
+-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Expecting the allocation to be freed.

```
$ openstack --os-placement-api-version 1.17 resource provider list \
  | awk '/devstack0:Open vSwitch agent:br-test/ { print $2 }' \
  | xargs -r -n1 openstack resource provider show --allocations
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| uuid        | f9c9ce07-679d-5d72-ac5f-31720811629a |
| name        | devstack0:Open vSwitch agent:br-test |
| generation  | 4                                    |
| allocations | {}                                   |
+-------------+--------------------------------------+
```

Expecting no usage reported.

```
$ openstack --os-placement-api-version 1.17 resource provider list \
  | awk '/devstack0:Open vSwitch agent:br-test/ { print $2 }' \
  | xargs -r -n1 openstack resource provider usage show
+-------------------------------------------+-------+
| resource_class                            | usage |
+-------------------------------------------+-------+
| NET_BANDWIDTH_EGRESS_KILOBITS_PER_SECOND  |     0 |
| NET_BANDWIDTH_INGRESS_KILOBITS_PER_SECOND |     0 |
+-------------------------------------------+-------+
```

That's all.

## Reference

Feature specifications and code (both merged and under review):

* Nova
  * spec:
    [final](https://specs.openstack.org/openstack/nova-specs/specs/rocky/approved/bandwidth-resource-provider.html),
    [with comments](https://review.openstack.org/502306)
  * code:
    [topic:bp/bandwidth-resource-provider](https://review.openstack.org/#/q/topic:bp/bandwidth-resource-provider)
* Neutron
  * spec:
    [final](https://specs.openstack.org/openstack/neutron-specs/specs/rocky/minimum-bandwidth-allocation-placement-api.html),
    [with comments](https://review.openstack.org/508149)
  * code:
    [topic:minimum-bandwidth-allocation-placement-api](https://review.openstack.org/#/q/topic:minimum-bandwidth-allocation-placement-api)

The code used to build this devstack environment.

devstack:
```
66ca7f55 Merge "Remove master only job"
```

neutron-lib (all merged to master when publishing this post):
```
b0f92bf Introduce Port resource request extension (cherry-pick 584903/9)
d672850 Mechanism driver API: resource_provider_uuid5_namespace
a792e02 Placement: utils (577220/17)
b76931f Placement: constants (580666/8)
f6d1286 Placement client: move to neutron_lib.placement
9402009 Placement client: optional RP generations
7b88af1 Placement client: ensure_resource_provider
a5619e3 Placement client: always return body
d24aa98 (on master)
```

neutron (some git commit hashes are wrong on the left side, but the gerrit ids on the right are correct):
```
+ logging fixes in 21d27e3 (not yet uploaded)
0c25b49 WIP: fill port-resource-request (cherry-pick 590363,3)
c8528d9 Introduce Port resource request extension (cherry-pick 584906,4)
f51a81e Enable ingress direction for min_bw rule (cherr-pick 584927,2)
41d5f86 Ingress direction for min bandwidth rule (cherry-pick 581029,3)
21d27e3 Drive binding by placement allocation (574783/13)
ea711bc WIP Placement reporting service plugin
c840080 Class to represent Placement state and sync
99cdb1f ovs/sriov mech drivers: resource_provider_uuid5_namespace (586597/4)
af5bd57 notification: Add 'status' to agent after_create/update
9224e62 sriov-agent: Report resource info in heartbeat
b08915a ovs-agent: Report resource info in heartbeat
5226e50 (tag: 13.0.0.0rc1, on master)
```

nova:
```
f9dbd79 Test boot with more ports with bandwidth request (573317/17)
8d7cc5c Send resource allocations in the port binding
4158c6a Transfer port.resource_request to the scheduler
1b05845 Add bandwidth related standard resource classes
4192366 Add requested_resources field to RequestSpec
485843d Add request_spec.RequestGroup versioned object
ed1ce2a Functional test for moving with nested resources
9ccb9c8 Functional test for booting with nested resources
7285027 Enable nested allocation candidates in scheduler
1a7671b Use placement 1.28 in scheduler report client
16f89fd (on master)
```

At this time Placement was still hosted in the Nova repository.
There were no open changes to Placement.
