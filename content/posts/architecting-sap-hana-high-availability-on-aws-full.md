---
title: "Architecting SAP HANA High Availability on AWS"
date: 2026-06-05
draft: false
description: "Building a production-ready SAP HANA cluster using Pacemaker, AWS-integrated STONITH fencing, overlay IPs, quorum devices, and preload readiness validation."
tags:
  - SAP HANA
  - AWS
  - Pacemaker
  - RHEL
  - High Availability
categories:
  - Infrastructure Architecture
---

# Architecting SAP HANA High Availability on AWS

*Building a production-ready SAP HANA cluster using Pacemaker, AWS-integrated STONITH fencing, overlay IPs, quorum devices, and preload readiness validation.*

![Figure 1 – SAP HANA High Availability Architecture](/images/hana-ha-aws-overview.png)

*Figure 1 – Overall architecture showing SAP HANA System Replication, overlay IP resources, STONITH fencing, and a dedicated quorum device.*

## Introduction

Implementing SAP HANA High Availability on AWS is not particularly difficult.

Implementing it correctly is.

Most documentation focuses on getting SAP HANA System Replication running between two nodes. While replication is certainly a prerequisite, replication alone does not provide High Availability. A production-ready environment must be able to make reliable failover decisions, prevent split-brain scenarios, maintain predictable client connectivity, and recover automatically from infrastructure failures.

This article describes the architecture used to build a highly available SAP HANA environment on AWS using the Red Hat High Availability Add-On, Pacemaker, AWS-integrated fencing, route-table-based overlay IPs, and a dedicated quorum device. It also introduces an additional operational safeguard that validates preload readiness before failover operations are performed.

The goal is not simply to survive failures, but to recover from them predictably and safely.

## Architecture Overview

| Host | Purpose |
|--------|--------|
| hana-node-a | SAP HANA Primary / Secondary |
| hana-node-b | SAP HANA Primary / Secondary |
| hana-qdevice-01 | Corosync QNetd Quorum Device |

The architecture combines:

- SAP HANA System Replication
- Pacemaker
- Corosync
- AWS STONITH fencing
- Read/Write overlay IP
- Read-Only overlay IP
- SAPHanaTopology resource agent
- SAPHana resource agent
- SAPHanaSR replication hooks
- Corosync quorum device
- Custom preload readiness validation

Unlike traditional active/passive clustering solutions, Pacemaker continuously evaluates SAP HANA state before making failover decisions. Replication health, synchronization state, node status, fencing availability, and quorum all contribute to the cluster's decision-making process.

## Why AWS Changes the Design

![Figure 2 – AWS Route Table Overlay IP Design](/images/hana-route-table-overlay-ips.png)

*Figure 2 – AWS route table entries used by the Read/Write and Read-Only overlay IP resources. During failover, Pacemaker updates these routes so traffic automatically follows the active SAP HANA node.*

Instead of moving an IP address, Pacemaker updates AWS route tables so that traffic is redirected toward the network interface currently hosting the resource.

## Preparing the Infrastructure

### IAM Permissions

The EC2 instance profile attached to both SAP HANA nodes must be capable of:

- Describing EC2 resources
- Creating routes
- Replacing routes
- Deleting routes
- Starting instances
- Stopping instances
- Rebooting instances

### Disable Source/Destination Check

Because overlay IP resources depend on route manipulation, this feature must be disabled on both SAP HANA nodes.

### Disable AWS Auto Recovery

Pacemaker should remain the authoritative source of recovery decisions.

## Building the Cluster Foundation

The cluster foundation consists of:

- Pacemaker
- PCS
- Corosync
- fence-agents-aws
- resource-agents-cloud
- resource-agents-sap-hana

```bash
dnf install pcs pacemaker fence-agents-all fence-agents-aws resource-agents-cloud resource-agents-sap-hana
```

```bash
pcs cluster setup --start --enable hana-cluster hana-node-a hana-node-b
```

## STONITH: The Most Important Component

Many engineers focus first on replication. The most important component is actually fencing.

```bash
pcs stonith create aws-cluster-fence fence_aws region=us-west-2 pcmk_host_map="hana-node-a:i-primary;hana-node-b:i-secondary"
```

## Managing Client Connectivity with Overlay IPs

Applications should never connect directly to a specific SAP HANA server.

### Read/Write Overlay IP

```bash
pcs resource create hana-rw-ip aws-vpc-move-ip ip=<rw-overlay-ip> routing_table=<route-table-id>
```

### Read-Only Overlay IP

A second overlay IP can be assigned to the synchronized secondary node.

## Integrating SAP HANA with Pacemaker

### SAPHanaTopology

```bash
pcs resource create SAPHanaTopology_HDB_00 SAPHanaTopology SID=HDB InstanceNumber=00
```

### SAPHana

Recommended settings:

```text
PREFER_SITE_TAKEOVER=true
AUTOMATED_REGISTER=true
DUPLICATE_PRIMARY_TIMEOUT=7200
```

## Replication Hooks

```text
/hana/shared/myHooks
```

```bash
hdbnsutil -reloadHADRProviders
```

## Validating the Cluster

![Figure 3 – Healthy Pacemaker Cluster State](/images/hana-pcs-status.png)

*Figure 3 – Healthy cluster state showing both SAP HANA nodes online, the Read/Write overlay IP hosted on the primary node, and the Read-Only overlay IP hosted on the secondary node.*

## The Role of the Quorum Device

```bash
pcs quorum device add model net host=hana-qdevice-01 algorithm=ffsplit
```

![Figure 5 – Corosync Quorum Validation](/images/hana-quorum-status.png)

*Figure 5 – Quorum validation showing two SAP HANA cluster votes plus the additional vote provided by the dedicated quorum device.*

## Production Hardening: Preload Readiness Validation

The service examines SAP HANA trace files for indicators such as:

```text
isPreloadComplete : 1
```

and publishes the results directly into Pacemaker.

```text
hana_hdb_preload_status
```

Possible values:

```text
complete
pending
```

![Figure 4 – Replication and Preload Validation](/images/hana-crm-mon-preload-status.png)

*Figure 4 – Cluster attributes showing replication state, synchronization health, and preload readiness validation for both SAP HANA nodes.*

This additional validation layer provides:

- Failover readiness visibility
- Safer maintenance operations
- Better patching decisions
- Increased operational confidence

## Lessons Learned

- Disable Source/Destination Check Early
- Let Pacemaker Own Recovery
- Test STONITH
- Validate IAM Permissions Early
- Quorum Devices Are Worth the Effort
- Replication Is Not Readiness

## Conclusion

Building SAP HANA High Availability on AWS requires more than simply enabling replication.

By integrating SAP HANA System Replication, Pacemaker, AWS route-table-based overlay IPs, STONITH fencing, quorum devices, replication hooks, and preload readiness validation, organizations can build a platform capable of recovering automatically from failures while maintaining both data integrity and service availability.

High Availability is not a single feature. It is the result of multiple layers working together—and validating each of those layers before they are needed.
