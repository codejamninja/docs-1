---
title: Auto Backup Overview | Stash
description: An overview on how auto backup works in Stash.
menu:
  product_stash_0.8.3:
    identifier: auto-backup-overview
    name: What is Auto Backup?
    parent: auto-backup
    weight: 10
product_name: stash
menu_name: product_stash_0.8.3
section_menu_id: guides
---

# Auto Backup in Stash

Stash can be configured to automatically backup of any stateful workloads in your cluster. Stash enables cluster administrators to deploy backup configuration templates ahead of time so that application owners can easily backup any types of workload with a few annotations. This allows Enterprises to stay prepared for disaster scenarios and recover from offsite secure backups in case of a disaster on public cloud and on-premises datacenters.

## What is Auto Backup

Stash uses 1-1 mapping among `Repository`, `BackupConfiguration` and the target. So, whenever you want to backup a target(workload/PVC/database), you have to create a `Repository` and `BackupConfiguration` object. This could become tiresome when you are trying to backup similar types of target and the `Repository` and `BackupConfiguration` have only a slight difference. To mitigate this problem, Stash provides a way to specify a template for these two objects via `BackupConfigurationTemplate` crd. In Stash parlance, we call this process as **Auto Backup**.

You have to create only one `BackupConfigurationTemplate` for all similar types of target. For example, you need only one `BackupConfigurationTemplate` for Deployment, DaemonSet, StatefulSet etc. Similarly, you have to create only one `BackupconfigurationTemplate` for all PostgreSQL databases. Then, you just need to add some annotations in the target. Stash will automatically create respective `Repository` and `BackupConfiguration` objects using the template and perform backups on pre-defined schedule.

## How Auto Backup Works?

The following diagram shows how automatic backup works in Stash. Open the image in a new tab to see the enlarged version.

<figure align="center">
  <img alt="Auto Backup Overview" src="/docs/images/guides/latest/auto-backup/default_backup.svg">
  <figcaption align="center">Fig: Auto Backup Overview</figcaption>
</figure>

The automatic backup process consists of the following steps:

1. A user creates a storage secret with necessary credentials of the backend where the backed up data will be stored.
2. Then, he creates a `BackupConfigurationTemplate` crd that specifies a template for `Repository` and `BackupConfiguration` object.
3. Then, he creates a workload with some specific annotations for automatic backup.
4. Stash operator watches for workloads. When it finds a workload with annotations for automatic backup, it finds out the respective `BackupConfigurationTemplate`.
5. Then, Stash operator resolves the template by replacing variable fields of the template with respective information from the workload.
6. Then, it creates a `Repository` and a `BackupConfiguration` object for the workload according to the resolved template.
7. Finally, Stash starts rest of the standard backup process as discussed in [here](/docs/guides/latest/workload/overview.md).

## Next Step

- Learn how to configure automatic backup for workloads from [here](/docs/guides/latest/auto-backup/workload.md).
- Learn how to configure automatic backup for PVCs from [here](/docs/guides/latest/auto-backup/pvc.md).
- Learn how to configure automatic backup for databases from [here](/docs/guides/latest/auto-backup/database.md).
