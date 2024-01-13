---
title: "24,000 Fabrics and Counting - Fixing MaaS Kubernetes Fabric Sprawl"
date: 2024-01-13 00:00:00 -0000
excerpt_separator: "<!--more-->"
categories:
  - Technical Blog
tags:
  - homelab
  - maas
  - bare-metal
  - kubernetes
header:
  image: /assets/images/2024-01-13-09-59-01.png
---

On a snowy Saturday morning, I enter my office intending to test out-of-band ingress-nginx on [MicroK8s](https://microk8s.io/). I log into my desktop and open up the [MaaS](https://maas.io/) dashboard to check the network configuration of my bare-metal Kubernetes nodes. When I open the page, my Fixfox window slows to a stall with spinning icons next to the fabrics.

I kept refreshing the page and opening it in new tabs, but the issue persisted. I checked the subnets page to see if that would load. I was expecting to see my standard subnets, and I did, but they were accompanied by nearly a 1,000 pages of generic `fabric-xxxxx` entries, over 24,000 of them.

![2024-01-13-09-59-01.png](/assets/images/2024-01-13-09-59-01.png)

In a discord call with me, as I was tearing out my hair, was Noah, who, after some brief searching, found [this thread on the MaaS forum](https://discourse.maas.io/t/maas-3-2-9-creates-calico-interfaces-80-000-fabrics/7625/6). Another user found that “[the [periodic] hardware sync](https://maas.io/docs/machines#heading--about-updating-hardware) recognises Calico interfaces from the K8 cluster every 15 minutes and creates a fabric for each interface.” In my case, I have four physical hosts managed by MaaS with periodic sync enabled, where three of which are in a Kubernetes cluster running Microk8s.

![Xnapper-2024-01-13-14.00.31.png](/assets/images/Xnapper-2024-01-13-14.00.31.png)

The thread acknowledges that it may be a bug. However, the proposed **workaround** is to **disable hardware sync** for the hosts running Kubernetes and **edit the database to remove the erroneous fabric entries**. The problem is that once hardware sync is enabled on a host, it can’t be disabled from the UI without releasing and deploying it again. But, you can edit the database entries for the hosts on MaaS to disable it.

First, let’s connect to the `maasdb` as the `postgres` user.

```
sudo su postgres
psql
postgres-# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 maasdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
postgres=# \c maasdb;
You are now connected to database "maasdb" as user "postgres".
```

Then select the hosts by `id` and `enable_hw_sync` from the `maasserver_node` table to see what the current status is.

```
maasdb=# select id,enable_hw_sync from maasserver_node;
 id | enable_hw_sync 
----+----------------
  1 | f
  2 | f
  7 | f
  5 | t
  6 | t
  8 | t
(6 rows)
```

In my case, I want to disable hardware sync (set `enable_hw_sync` to `false`) for all of my hosts, so I used an UPDATE statement for all records in `maasserver_node`.

```
maasdb=# UPDATE maasserver_node SET enable_hw_sync = false;
UPDATE 6
maasdb=# select id,enable_hw_sync from maasserver_node;
 id | enable_hw_sync 
----+----------------
  1 | f
  2 | f
  7 | f
  5 | f
  6 | f
  8 | f
(6 rows)
```

Turns out, there is also a systemd timer named `maas_hardware_sync` on each host to clean up. I found that out by watching the fabrics get added in real-time after I thought I had cleaned them up.

```
ubuntu@rubrik-c:~$ sudo systemctl list-units --type=timer maas_hardware_sync.timer
  UNIT                     LOAD   ACTIVE SUB     DESCRIPTION                                      
  maas_hardware_sync.timer loaded active waiting Timer for periodically running MAAS hardware sync
```

On each host, disable and remove the timer.

```
sudo systemctl disable maas_hardware_sync.timer
sudo systemctl stop maas_hardware_sync.timer
sudo rm /lib/systemd/system/maas_hardware_sync.timer
```

In my case, these are non-production hosts, used for testing the infrastructure deployment, I chose to simply redeploy the nodes.

![Xnapper-2024-01-13-14.55.55.png](/assets/images/Xnapper-2024-01-13-14.55.55.png)

Then clean up the fabrics and VLANs. Using these SQL commands, two DELETE statements intended to remove records from the `maasserver_vlan` and `maasserver_fabric` tables.

```
maasdb=# DELETE FROM maasserver_vlan
	WHERE maasserver_vlan.id NOT IN (
	    SELECT maasserver_vlan.id 
	    FROM maasserver_vlan
	    LEFT JOIN maasserver_interface ON maasserver_vlan.id = maasserver_interface.vlan_id
	    WHERE maasserver_interface.vlan_id IS NOT NULL  
	)
	AND maasserver_vlan.id NOT IN (
	    SELECT maasserver_vlan.id 
	    FROM maasserver_vlan
	    JOIN maasserver_subnet ON maasserver_vlan.id = maasserver_subnet.vlan_id
	);

	DELETE FROM maasserver_fabric
	WHERE maasserver_fabric.id NOT IN (
	    SELECT maasserver_fabric.id 
	    FROM maasserver_fabric
	    LEFT JOIN maasserver_vlan ON maasserver_vlan.fabric_id = maasserver_fabric.id
	    WHERE maasserver_vlan.fabric_id IS NOT NULL
	);
```

Let's break down each DELETE statement:

1. `DELETE FROM maasserver_vlan`**:** This statement deletes VLANs (from the table `maasserver_vlan`) that are not associated with a network interface of a server (left join with `maasserver_interface` on the `vlan_id`) or used by a subnet (inner join with `maasserver_subnet` on the `vlan_id`).

2. `DELETE FROM maasserver_fabric`**:** This statement deletes fabrics (from the table `maasserver_fabric`) that are not associated with a VLAN (left join with `maasserver_vlan` on the `fabric_id`).

In my case, that was `23902` VLANs and `23899` fabrics.

```
DELETE 23902
DELETE 23899
```

I had a few fabrics left over that I had to delete the subnet for first, but once I did and ran the DELETE statements again, my list was clear of extra fabrics. Now with the hosts reconfigured with my Ansible playbook, and the MicroK8s nodes joined together I have my cluster back. The periodic hardware sync is disabled and I have only the one intended fabric.

![Xnapper-2024-01-13-14.55.55.png](/assets/images/2024-01-13-16-55-08.png)

I am still learning how to best utilize MaaS, and sometimes I reconsider if it’s worth the added complexity. Yet, I am still using it instead of writing my own scripts to interact with the IPMI/iL02 APIs or host PXE servers. So it must be, at least for now.