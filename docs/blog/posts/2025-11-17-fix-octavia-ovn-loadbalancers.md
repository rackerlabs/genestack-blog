---
date: 2025-11-17
title: Resolving Corrupted Octavia OVN Loadbalancing Config Issues
authors:
  - the2hill
description: >
  This document describes how to resolve some Octavia OVN loadbalancer config issues
categories:
  - octavia
  - ovn
  - loadbalancers

---

# Octavia OVN Overview

Octavia provides the ability to utilize OVN([Open Virtual Networking](https://www.ovn.org/en/)) as a provider driver to deploy layer 4 loadbalancing.
While Octavia OVN loadbalancers may not be as feature rich as Amphora based loadbalancing OVN based loadbalancers provide a resource efficient, fast deploying and highly availble loadbalancing solution. 
View the [Amphora and OVN](https://docs.openstack.org/octavia/latest/user/feature-classification/index.html) loadbalancing comparison matrix for more info about which features are supported. 

For additional information regarding Octavia OVN loadbalancers view the [Octavia OVN Docs](https://docs.openstack.org/ovn-octavia-provider/latest/contributor/loadbalancer.html).

<!-- more -->
## Resolving Common Octavia OVN configuration Issues

While rare, there's a chance that the Octavia OVN configurations that define a specified loadbalancer as part of OVN's northbound and southbound databases(NBDB/SBDB) become corrupted or inconsistent. 
For example, in kube-ovn versions prior to v1.14.10, garbage collection may consider Octavia OVN loadbalancers as part of the cleanup process and completely destroy the loadbalancer configurations.
Or it's possible that the NBDB or SBDB were wiped for some other reason and simply need to be restored.

For these scenarios we can make use of a command line utility to sync Octavia database definitions to OVN loadbalancer configurations in the appropriate OVN databases. 
You can find the utility as part of the [ovn-octavia-provider](https://github.com/openstack/ovn-octavia-provider/blob/master/ovn_octavia_provider/cmd/octavia_ovn_db_sync_util.py) repo. 

This utility should work for just about all cases where Octavia loadbalancer definitions are retained in the Octavia database as it will treat the Octavia database as the source of truth and attempt to re-configure OVN loadbalancers to match the details defined there. 

As noted in the doc above, running the sync script will output something similar to below:

``` shell
(venv) stack@ubuntu2404:~/ovn-octavia-provider$ octavia-ovn-db-sync-util
```

``` log
INFO ovn_octavia_provider.cmd.octavia_ovn_db_sync_util [-] OVN Octavia DB sync start.
INFO ovn_octavia_provider.driver [-] Starting sync OVN DB with Loadbalancer filter {'provider': 'ovn'}
INFO ovn_octavia_provider.driver [-] Starting sync OVN DB with Loadbalancer lb1
DEBUG ovn_octavia_provider.driver [-] OVN loadbalancer 5bcaab92-3f8e-4460-b34d-4437a86909ef not found. Start create process. {{(pid=837681) _ensure_loadbalancer /opt/stack/ovn-octavia-provider/ovn_octavia_provider/driver.py:684}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): DbCreateCommand(_result=None, table=Load_Balancer, columns={'name': '5bcaab92-3f8e-4460-b34d-4437a86909ef', 'protocol': [], 'external_ids': {'neutron:vip': '192.168.100.188', 'neutron:vip_port_id': 'e60041e8-01e8-459b-956e-a55608eb5255', 'enabled': 'True'}, 'selection_fields': ['ip_src', 'ip_dst', 'tp_src', 'tp_dst']}, row=False) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): LsLbAddCommand(_result=None, switch=000a1a3e-edff-45ad-9241-5ab8894ac0e0, lb=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, may_exist=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=1): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('external_ids', {'ls_refs': '{"neutron-000a1a3e-edff-45ad-9241-5ab8894ac0e0": 1}'}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): LrLbAddCommand(_result=None, router=f17e58b5-37d2-4daf-a02f-82fb4974f7b8, lb=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, may_exist=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=1): LsLbAddCommand(_result=None, switch=neutron-000a1a3e-edff-45ad-9241-5ab8894ac0e0, lb=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, may_exist=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=2): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('external_ids', {'lr_ref': 'neutron-d2dd599c-76c7-43c1-8383-1bae5593681a'}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('protocol', 'tcp'),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('external_ids', {'listener_30ac9d4e-4fdd-4885-8949-6a2e7355beb2': '80:pool_5814b9e6-db7e-425d-a4cf-1cb668ba7080'}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=1): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('protocol', 'tcp'),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=2): DbClearCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, column=vips) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=3): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('vips', {}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('external_ids', {'enabled': 'True', 'neutron:vip': '192.168.100.188', 'neutron:vip_port_id': 'e60041e8-01e8-459b-956e-a55608eb5255', 'ls_refs': '{"neutron-000a1a3e-edff-45ad-9241-5ab8894ac0e0": 1}', 'lr_ref': 'neutron-d2dd599c-76c7-43c1-8383-1bae5593681a', 'listener_30ac9d4e-4fdd-4885-8949-6a2e7355beb2': '80:pool_5814b9e6-db7e-425d-a4cf-1cb668ba7080', 'pool_5814b9e6-db7e-425d-a4cf-1cb668ba7080': ''}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovn_octavia_provider.helper [-] no member status on external_ids: None {{(pid=837681) _find_member_status /opt/stack/ovn-octavia-provider/ovn_octavia_provider/helper.py:2490}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('external_ids', {'pool_5814b9e6-db7e-425d-a4cf-1cb668ba7080': 'member_94ceacd8-1a81-4de9-ac0e-18b8e41cf80f_192.168.100.194:80_b97280a1-b19f-4989-a56c-2eb341c23171'}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=1): DbClearCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, column=vips) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=2): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('vips', {'192.168.100.188:80': '192.168.100.194:80'}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('external_ids', {'ls_refs': '{"neutron-000a1a3e-edff-45ad-9241-5ab8894ac0e0": 2}'}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): LrLbAddCommand(_result=None, router=f17e58b5-37d2-4daf-a02f-82fb4974f7b8, lb=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, may_exist=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=1): LsLbAddCommand(_result=None, switch=neutron-000a1a3e-edff-45ad-9241-5ab8894ac0e0, lb=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, may_exist=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Transaction caused no change {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:129}}
DEBUG ovn_octavia_provider.helper [-] no member status on external_ids: None {{(pid=837681) _update_external_ids_member_status /opt/stack/ovn-octavia-provider/ovn_octavia_provider/helper.py:2521}}
DEBUG ovsdbapp.backend.ovs_idl.transaction [-] Running txn n=1 command(idx=0): DbSetCommand(_result=None, table=Load_Balancer, record=d69e29cd-0069-4d7f-a1ed-08c246bfb3da, col_values=(('external_ids', {'neutron:member_status': '{"94ceacd8-1a81-4de9-ac0e-18b8e41cf80f": "NO_MONITOR"}'}),), if_exists=True) {{(pid=837681) do_commit /opt/stack/ovn-octavia-provider/venv/lib/python3.12/site-packages/ovsdbapp/backend/ovs_idl/transaction.py:89}}
DEBUG ovn_octavia_provider.helper [-] Updating status to octavia: {'loadbalancers': [{'id': '5bcaab92-3f8e-4460-b34d-4437a86909ef', 'provisioning_status': 'ACTIVE', 'operating_status': 'ONLINE'}], 'listeners': [{'id': '30ac9d4e-4fdd-4885-8949-6a2e7355beb2', 'provisioning_status': 'ACTIVE', 'operating_status': 'ONLINE'}], 'pools': [{'id': '5814b9e6-db7e-425d-a4cf-1cb668ba7080', 'provisioning_status': 'ACTIVE', 'operating_status': 'ONLINE'}], 'members': [{'id': '94ceacd8-1a81-4de9-ac0e-18b8e41cf80f', 'provisioning_status': 'ACTIVE', 'operating_status': 'NO_MONITOR'}]} {{(pid=837681) _update_status_to_octavia /opt/stack/ovn-octavia-provider/ovn_octavia_provider/helper.py:428}}
INFO ovn_octavia_provider.driver [-] Starting sync floating IP for loadbalancer 5bcaab92-3f8e-4460-b34d-4437a86909ef
WARNING ovn_octavia_provider.driver [-] Floating IP not found for loadbalancer 5bcaab92-3f8e-4460-b34d-4437a86909ef
INFO ovn_octavia_provider.cmd.octavia_ovn_db_sync_util [-] OVN Octavia DB sync finish.
```


## Special Failure Cases

In some exceedingly rare cases the Octavia database could become corrupted, most notably by attempted manual cleanup efforts that leaves Octavia OVN loadbalancers in a bad state. 
The scenario we've run in to began as a networking issue that caused OVN loadbalancer failures. In attempt to resolve these failures the Octavia database was altered to cleanup resources.
During this period a loadbalancer DELETE call was issued causing other parts of the loadbalancer resources to be removed from the database while the OVN NB/SB DB's retained the configurations. 
The DELETE call ultimately failed due to this with log messages similar to:

``` log
ERROR ovn_octavia_provider.helper [-] Exception occurred during deletion of loadbalancer: AttributeError: 'NoneType' object has no attribute 'admin_state_up'
ERROR ovn_octavia_provider.helper Traceback (most recent call last):
ERROR ovn_octavia_provider.helper   File \"/var/lib/openstack/lib/python3.12/site-packages/ovn_octavia_provider/helper.py\", line 1752, in lb_delete
ERROR ovn_octavia_provider.helper     status = self._lb_delete(loadbalancer, ovn_lb, status)
ERROR ovn_octavia_provider.helper              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ERROR ovn_octavia_provider.helper   File \"/var/lib/openstack/lib/python3.12/site-packages/ovn_octavia_provider/helper.py\", line 1794, in _lb_delete
ERROR ovn_octavia_provider.helper     self.member_delete(member)
ERROR ovn_octavia_provider.helper   File \"/var/lib/openstack/lib/python3.12/site-packages/ovn_octavia_provider/helper.py\", line 2776, in member_delete
ERROR ovn_octavia_provider.helper     status = self._get_current_operating_statuses(ovn_lb)
ERROR ovn_octavia_provider.helper              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ERROR ovn_octavia_provider.helper   File \"/var/lib/openstack/lib/python3.12/site-packages/ovn_octavia_provider/helper.py\", line 4044, in _get_current_operating_statuses
ERROR ovn_octavia_provider.helper     if not _pool.admin_state_up or not member_statuses:
ERROR ovn_octavia_provider.helper            ^^^^^^^^^^^^^^^^^^^^"
ERROR ovn_octavia_provider.helper AttributeError: 'NoneType' object has no attribute 'admin_state_up'
```
This indicated that the loadbalancer members were already removed from the database leaving Octavia OVN unable to properly parse the DELETE call to continue its operations.

Depending on the resources removed the error message could be slightly different, but the root of the issue remains that the Octavia database no long has data that the OVN driver expects to be mapped so it can clean up its OVN resources.

If this loadbalancer were meant to be an ACTIVE loadbalancer then to resolve the broken configurations we would have to manually add back the missing Octavia database resource entries, such as member references, and run the sync util that we noted above.

In this particular case of a loadbalancer meant to be DELETED we need to purge the OVN NBDB resources ourselves and then re-issue a loadbalancer delete call to clean everything up.

The next section will describe how to clean up the OVN NBDB. 

## Cleaning Up The OVN Database

First we need to find our loadbalancer in the OVN NBDB. This is much easier if we are using the [`ko`](https://docs.rackspacecloud.com/k8s-tools/?h=ko#install-kubectl) plugin. 
If we're not, we need to shell in to the ovn-central pod and run list load_balancer command and grep for our loadbalancer ID. 

``` shell
root@kubernetes03:/kube-ovn# ovn-nbctl list load_balancer | grep ff558e93-9ce0-4b42-a229-b0509d349fda
```

The loadbalancer ID that's stored in the Octavia database will be found in the OVN NBDB under the name field. Example:

``` shell
root@kubernetes03:/kube-ovn# ovn-nbctl list load_balancer | grep ff558e93-9ce0-4b42-a229-b0509d349fda
_uuid               : 50239c74-0538-4f9b-b8a6-ba3c16cd8c3b
external_ids        : {enabled=True, listener_02651559-25a5-4e4e-ab2b-4b2e8f51b9cc="80:pool_6173717f-fe98-4c10-b7b9-d4fcc1219754", listener_28e4eef6-7a6a-48a7-99cd-fc7cf0b3f488="443:pool_2de93fe2-93f8-4be5-ac7e-2a62157efc6f", lr_ref=neutron-0e20b461-f780-4af2-9ae4-2c104df85904, ls_refs="{\"neutron-6430e184-151e-4c7d-96a9-a064c92b04d8\": 3}", "neutron:member_status"="{\"1c9f2de4-201c-4c30-af50-665d8a2ccbc2\": \"NO_MONITOR\", \"51662e2c-262d-492c-810b-4f4131138fdc\": \"NO_MONITOR\"}", "neutron:vip"="10.0.2.39", "neutron:vip_fip"="65.17.193.37", "neutron:vip_port_id"="ee7411b3-9a39-4f8a-938f-7c3a0437e7fc", pool_2de93fe2-93f8-4be5-ac7e-2a62157efc6f="member_51662e2c-262d-492c-810b-4f4131138fdc_10.0.0.255:30092_ba56856a-cd30-4a8d-8d1b-620b5ccbb617", pool_6173717f-fe98-4c10-b7b9-d4fcc1219754="member_1c9f2de4-201c-4c30-af50-665d8a2ccbc2_10.0.0.255:31131_ba56856a-cd30-4a8d-8d1b-620b5ccbb617"}
health_check        : []
ip_port_mappings    : {}
name                : "ff558e93-9ce0-4b42-a229-b0509d349fda"
options             : {}
protocol            : tcp
selection_fields    : [ip_dst, ip_src, tp_dst, tp_src]
vips                : {"10.0.2.39:443"="10.0.0.255:30092", "10.0.2.39:80"="10.0.0.255:31131", "65.17.111.11:443"="10.0.0.255:30092", "65.17.111.12:80"="10.0.0.255:31131"}
```

If you do have `ko` installed on a jumpbox the command would simply be: 

``` shell
kubectl ko nbctl find load_balancer name=ff558e93-9ce0-4b42-a229-b0509d349fda

_uuid               : 50239c74-0538-4f9b-b8a6-ba3c16cd8c3b
external_ids        : {enabled=True, listener_02651559-25a5-4e4e-ab2b-4b2e8f51b9cc="80:pool_6173717f-fe98-4c10-b7b9-d4fcc1219754", listener_28e4eef6-7a6a-48a7-99cd-fc7cf0b3f488="443:pool_2de93fe2-93f8-4be5-ac7e-2a62157efc6f", lr_ref=neutron-0e20b461-f780-4af2-9ae4-2c104df85904, ls_refs="{\"neutron-6430e184-151e-4c7d-96a9-a064c92b04d8\": 3}", "neutron:member_status"="{\"1c9f2de4-201c-4c30-af50-665d8a2ccbc2\": \"NO_MONITOR\", \"51662e2c-262d-492c-810b-4f4131138fdc\": \"NO_MONITOR\"}", "neutron:vip"="10.0.2.39", "neutron:vip_fip"="65.17.193.37", "neutron:vip_port_id"="ee7411b3-9a39-4f8a-938f-7c3a0437e7fc", pool_2de93fe2-93f8-4be5-ac7e-2a62157efc6f="member_51662e2c-262d-492c-810b-4f4131138fdc_10.0.0.255:30092_ba56856a-cd30-4a8d-8d1b-620b5ccbb617", pool_6173717f-fe98-4c10-b7b9-d4fcc1219754="member_1c9f2de4-201c-4c30-af50-665d8a2ccbc2_10.0.0.255:31131_ba56856a-cd30-4a8d-8d1b-620b5ccbb617"}
health_check        : []
ip_port_mappings    : {}
name                : "ff558e93-9ce0-4b42-a229-b0509d349fda"
options             : {}
protocol            : tcp
selection_fields    : [ip_dst, ip_src, tp_dst, tp_src]
vips                : {"10.0.2.39:443"="10.0.0.255:30092", "10.0.2.39:80"="10.0.0.255:31131", "65.17.111.11:443"="10.0.0.255:30092", "65.17.111.12:80"="10.0.0.255:31131"}
```

You'll get the same results that we can now collect the OVN NBDB UUID from so that we can issue a `destroy` command and purge this entry from OVN. 

First let's verify our router and switch config using `ko` to make it a bit easier.

First we use `ls_refs` secction to find the name in the external_ids of the loadbalancer output prepended with `neutron` to search for our switch config.

``` shell 
kubectl ko nbctl find logical_switch name=neutron-6430e184-151e-4c7d-96a9-a064c92b04d8

_uuid               : 9c190e67-f8ee-4701-b2d8-d7923c85711e
acls                : []
copp                : []
dns_records         : []
external_ids        : {"neutron:availability_zone_hints"=az1, "neutron:mtu"="3942", "neutron:network_name"=n1, "neutron:provnet-network-type"=geneve, "neutron:revision_number"="2"}
forwarding_groups   : []
load_balancer       : [50239c74-0538-4f9b-b8a6-ba3c16cd8c3b]
load_balancer_group : []
name                : neutron-6430e184-151e-4c7d-96a9-a064c92b04d8
other_config        : {mcast_flood_unregistered="false", mcast_snoop="false", vlan-passthru="false"}
ports               : [6ae5593a-2e85-462a-a4df-bf63d24e2a64, 826e0d4f-f3ef-48b5-8a49-1d51301234bb, a0a2df02-4d9d-459d-81a6-ceee43bf34c7, de61d484-644f-44ff-aaab-c047b96fc03a]
qos_rules           : []
```

Then we can do the same for our router using the `lr_ref` found in the load_balancer external_ids. 

``` shell
kubectl ko nbctl find logical_router name=neutron-74a4770b-820c-474d-9e1a-35d268c1ce95
_uuid               : c4400f2a-6007-4e80-b3ad-58a7cb9149b6
copp                : []
enabled             : true
external_ids        : {"neutron:availability_zone_hints"=az1, "neutron:revision_number"="3", "neutron:router_name"=router-1}
load_balancer       : [50239c74-0538-4f9b-b8a6-ba3c16cd8c3b]
load_balancer_group : []
name                : neutron-0e20b461-f780-4af2-9ae4-2c104df85904
nat                 : [0c1803be-cabf-4049-ad1f-f43bb9030b6f, a66d4389-ae6c-4f7d-aba6-d8f964f0840a, ce07fb8d-575b-4e01-b01e-eeabc23d6407]
options             : {always_learn_from_arp_request="false", dynamic_neigh_routers="true", mac_binding_age_threshold="0"}
policies            : []
ports               : [3b90397f-cd3a-4309-bf26-f975c82c38dd, bee55835-a807-4004-b4b8-8eda7e37dd42]
static_routes       : [7e3c520f-e8c3-433e-8ae2-819f633e1952]
```

This shows our router and switch that exists outside of Octavia and is what's used to provide the specified communcations channels over the OVN system. 
In both cases here we see that within the configuration there is a `load_balancer` field that maps to our loadbalancer.
We want to ensure that the load_balancer reference is removed from the router and switch also, so we'll keep these commands handy for future reference. 

Now we can `destroy` the OVN loadbalancer configurations. 

``` shell
root@kubernetes03:/kube-ovn# ovn-nbctl destroy load_balancer 50239c74-0538-4f9b-b8a6-ba3c16cd8c3b
```

or with `ko`

``` shell
kubectl ko nbctl destroy load_balancer 50239c74-0538-4f9b-b8a6-ba3c16cd8c3b
```

This will remove the OVN load_balancer configuration and also the references in the router and switch tables. Let's verify!

Make sure the load_balancer config is gone and returns empty.

``` shell
kubectl ko nbctl find load_balancer name=ff558e93-9ce0-4b42-a229-b0509d349fda

```

Verify that the references in the router and switch are empty also. 

``` shell 
kubectl ko nbctl find logical_switch name=neutron-6430e184-151e-4c7d-96a9-a064c92b04d8

_uuid               : 9c190e67-f8ee-4701-b2d8-d7923c85711e
acls                : []
copp                : []
dns_records         : []
external_ids        : {"neutron:availability_zone_hints"=az1, "neutron:mtu"="3942", "neutron:network_name"=n1, "neutron:provnet-network-type"=geneve, "neutron:revision_number"="2"}
forwarding_groups   : []
load_balancer       : []
load_balancer_group : []
name                : neutron-6430e184-151e-4c7d-96a9-a064c92b04d8
other_config        : {mcast_flood_unregistered="false", mcast_snoop="false", vlan-passthru="false"}
ports               : [6ae5593a-2e85-462a-a4df-bf63d24e2a64, 826e0d4f-f3ef-48b5-8a49-1d51301234bb, a0a2df02-4d9d-459d-81a6-ceee43bf34c7, de61d484-644f-44ff-aaab-c047b96fc03a]
qos_rules           : []
```

and 

``` shell
kubectl ko nbctl find logical_router name=neutron-74a4770b-820c-474d-9e1a-35d268c1ce95
_uuid               : c4400f2a-6007-4e80-b3ad-58a7cb9149b6
copp                : []
enabled             : true
external_ids        : {"neutron:availability_zone_hints"=az1, "neutron:revision_number"="3", "neutron:router_name"=router-1}
load_balancer       : []
load_balancer_group : []
name                : neutron-0e20b461-f780-4af2-9ae4-2c104df85904
nat                 : [0c1803be-cabf-4049-ad1f-f43bb9030b6f, a66d4389-ae6c-4f7d-aba6-d8f964f0840a, ce07fb8d-575b-4e01-b01e-eeabc23d6407]
options             : {always_learn_from_arp_request="false", dynamic_neigh_routers="true", mac_binding_age_threshold="0"}
policies            : []
ports               : [3b90397f-cd3a-4309-bf26-f975c82c38dd, bee55835-a807-4004-b4b8-8eda7e37dd42]
static_routes       : [7e3c520f-e8c3-433e-8ae2-819f633e1952]
```

We can see that the references are cleaned up from the switch and router tables and the load_balancer configuration no longer exists.

Now our OVN loadbalancer definition has been removed we can continue cleaning up the Octavia side by issuing a cascading delete command.

``` shell
openstack loadbalancer delete ff558e93-9ce0-4b42-a229-b0509d349fda --cascade
```

That's it! We have successfully cleaned up a broken Octavia OVN loadbalancer configuration.
