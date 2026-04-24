# mysql-oadp
How to backup and restore MySQL replication instances running in OpenShift 4.19 using OADP


## Background

**Set up**
- OpenShift 4.19
- GCP (Google Cloud Platform) for offiste bucket storage
- MySQL 8.4.7 deployed via Chainguard/Bitnami helm charts

**What I knew about backing up MySQL databases**

*NOTHING*

What I learned during my research in to how to back up MySQL databases (regardless if it's running on OpenShift or not):

- Make sure to freeze the database to prevent data corruption
- Minimal to no downtime is required 
- Move data off-site 
- Have point-in-time-recovery

Potential tools that were considered:
- [mysqldump](https://dev.mysql.com/doc/refman/9.6/en/mysqldump.html) - a command line utility that will create logical backups by creating SQL files 
- [MariaDB Operator](https://github.com/mariadb-operator/mariadb-operator) - there is a [Red Hat certified operator](https://catalog.redhat.com/en/software/container-stacks/detail/65789bcbe17f1b31944acb1d) that can deploy the MariaDB instances and handle minimal downtime backups
- Use a different third party tool to handle backups like [Percona XtraBackup](https://www.percona.com/mysql/software/percona-xtrabackup) or [Oracle Enterprise Backup](https://www.mysql.com/products/enterprise/backup.html)

None of these were able to satisfy requirements in my situation because they either:
- took too long
- did not meet developer requirements 
- required paying more money

This led me to [OADP (OpenShift API for Data Protection)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore)
- Is included as part of OpenShift subscription
- Based on [Velero](https://velero.io/) an open source Kubernetes native back up, restore, and migration tool
- Able to use snapshots to have minimal downtime to the databases
- Can backup and restore all OpenShift resources in a project, not just the data
- Able to restore to a completely new cluster if the original is lost

## Installing and Configuring OAPD
[Installing OADP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#installing-oadp) is as easy as selecting the OADP operator from the operator catalog and clicking 'install'

Follow [these instructions](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#configuring-oadp-with-google-cloud) for configuring with GCP (includes set up of storage bucket and service account)

Once OADP is installed, create a `DataProtectionApplication (DPA)`, this is where we define where backups are stored, what credentials to use, and what plugins to enable. This will create a `BackupStorageLocation (BSL)` that connects to the bucket created in GCP
    
---
### Data Mover 
The important thing - this allows us to take advantage of the split second speed of a volume snapshot while still uploading the data to GCP

- Snapshot only - snapshots only reside on the cluster so is vulnerable to cluster wide disasters 

- File system backup only - Very slow to check and upload every file even with kopia. presents an unacceptable amount of downtime to the database every time a backup is taken 

---
DataMover allows OADP to take a "split-second" capture of your data and then perform the slow, heavy lifting of uploading to GCP in the background without affecting your application. 

[Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-data-mover)

[Blog Post](https://www.redhat.com/en/blog/openshift-apis-data-protection-13-data-mover)

*Note* - The first backup will take the longest as it uploads the full dataset. Subsequent backups are significantly quicker because Kopia only uploads the blocks that have changed since the last run.

The Backup Workflow
1. Instant Capture: OADP triggers a CSI Snapshot. Because this is a metadata operation at the storage level, it completes in a split second. Your MySQL database only needs to be frozen for this tiny window.

2. Snapshot Unlocking: Immediately after the snapshot is taken, the database is "thawed" and resumes normal operations.

3. The DataUpload Engine: OADP creates a DataUpload object and spins up DataMover pods (via the node-agent).

4. The Temp PV: These pods mount a temporary PVC which is a clone of the snapshot.

5. Durable Upload: The DataMover pod uses Kopia to deduplicate and upload the data from the temporary PVC to your GCP bucket.

---

### Backup Resource

How we define what objects to backup

[Velero Documentation](https://velero.io/docs/main/api-types/backup/)

[OADP Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-creating-backup-cr-doc)

Important spec components in the example `backup.yaml` and `schedule.yaml`:
- `ttl` - 'Time to live' - how long the backup will exist in the GCP bucket, this does not apply to the data backed up 
- `storage location ` - Which `BSL` to back up to
- `snapshotmovedata ` - This is how we enable `Data mover` 
- `excludedResources` - No secrets should be backed up, even if it is encrypted, anything required should be an external secret anyways
- `orderedResources` - Necessary to avoid split brain issue on restore. We want to backup the secondary mysql instance first so that it will be behind the primary mysql instance. If we do the opposite, the secondary could have newer data that the primary does not and when restoring, the secondary will have data the priamry does not. Only need to list `pods` because velero will also backup any mounted `persistent volumes` before backing up the next in the list.
- `hooks` - Necessary to freeze the database to avoid corruption, in my testing the time between hooks was about 15 seconds 
    - Pre-hook - Apply a read lock on the database, keep the process in the background, and save the pid. The `DO SLEEP` is necessary otherwise the the read lock will only run for that exact second and then release
    - Post-hook - kill the process started by the pre-hook
---
### Schedule Resource

To create backups on a schedule - we need to create a [schedule resource](https://velero.io/docs/v1.3.0/api-types/schedule/) - essentially a backup resource with cron. 

When a backup is created via the `schedule` resource, the backup will follow the template specified in the yaml and will be named `<schedule-resource-name>-<date time created>



*Process of what happens when a scheduled backup is triggered*
---
**Reminder** - Because Kopia uses incremental backups the initial backup of a volume will take the longest amount of time (can up more than an hour) as all the data is new. Subsequent backups will take significantly less time

1. A new backup resource is created
2. CSI Snapshot: OADP requests a local CSI snapshot of the volume. 
    1. For every volume being moved, OADP creates a `DataUpload` Custom Resource. 
    2.  Data Mover Pods: The node-agent (a DaemonSet running on your worker nodes) detects the DataUpload. It spins up or utilizes Data Mover pods (often referred to as the node-agent's worker process) to do the heavy lifting. These pods mount the CSI snapshot as a volume

3. Once the backup is complete you will see new directories in your GCP bucket
    - `/backups/<backup-name>`: Contains the metadata tarballs and logs.
    - `/kopia`: This is the "Repository." This is where the deduplicated volume data lives.

Phases: 
- **InProgress**: Metadata is being collected and snapshots are being triggered.
- **WaitingForPluginOperations**: The DataMover phase. OADP is waiting for the DataUpload pods to finish moving data to GCP.
- **Completed**
- **PartiallyFailed** - YAMLs are safe; some Data or Objects failed.
- **Failed**
---
### Restore 

How we define what backup we want to restore from and what objects should be restored 

[Velero Documentation](https://velero.io/docs/main/api-types/restore/)

[OADP Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-restoring)

Important spec components in the example `restore.yaml`:
- `backupName` - Must specify the backup or else restore will not happen
- `includedNamespaces` - Which namespaces to restore from the backup, can be multiple but for simplicity we only list the one from the backup
- `excludedResources` - Upon restore that included `pods`, there were no connections going in/out of the `mysql` pods. *Reason unknown for now.* So in order to avoid this issue, exclude restoring the pods and let the `statefulset` spin up a fresh pod.

*Process of what happens when a restore is triggered*
---
1. Before initiating a restore, it is highly recommended to completely delete the target project (namespace).
    - Avoid Conflicts: OADP may fail to overwrite existing resources (Services, Routes, or ConfigMaps), leading to a PartiallyFailed restore status.

2. When using DataMover (powered by VolSync and Kopia in OADP 4.19), the restore follows a sophisticated "Helper Pod" sequence to move data from offsite storage back into your cluster volumes.

    Step 1: API Object Injection
    OADP instantly injects the "logic" of your application (YAMLs for Deployments, Services, Routes, etc.) into the OpenShift API. Your application pods will appear but stay in a Pending or ContainerCreating state.

    Step 2: DataDownload Execution - The heavy lifting occurs through the DataDownload Custom Resource:

    1. Helper Pods: OADP spins up temporary restore pods in the openshift-adp namespace.

    2. Temporary PVC: These pods mount a temporary PVC.

    3. Data Transfer: Data is pulled from the object store (GCS/S3) and written into this temporary volume.

    4. Volume Binding: Once the download is 100% complete, the temporary PVC is detached, and the "Real" PVC for your application is bound to the populated Persistent Volume (PV).

    5. Termination: Once the data is successfully placed, the helper pods and DataDownload objects are cleaned up.

    Step 3: Application pods will reach `running` state. 
    - Note - if the secondary mysql instance starts before the primary, it will have 10 attempts to connect to the primary, once every 10 seconds. If all 10 attempts fail the secondary will remain `running` but is not actively connected to the primary. If this is the case restart the secondary pod.

Phases: 
- **InProgress**: OADP is recreating your Namespaces, Services, and Secrets.
- **WaitingForPluginOperations**: The YAMLs are in, but OADP is now waiting for the DataDownload (DataMover) pods to finish pulling the MySQL blocks from GCP.
- **Completed**
- **PartiallyFailed** - Some objects already existed or had conflicts. 
- **Failed**
---


### What if we did want to only backup with snapshots?

- Make sure your storage csi driver is capable of volume snapshots.
- Retain the pre/post hooks to apply the `READ LOCK` to the database.
- Current version of OADP requires us to delete the `VolumeSnapshot` and `VolumeSnapshotContent` objects that are created after a restore is run. Snapshots created during a backup and deleted automatically. 