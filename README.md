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
The important thing. able to take advantage of the split second speed of a volume snapshot while still uploading the data to gcp. 

if we dont do this we rely solely on volumesnapshots or completel file system backup

snapshot only - only reside on the cluster so is vulnerable to cluster wide disasters 
fs backup - super slow to check and upload every file even with kopia. presents an unacceptable amount of downtime to the database every time a backup is taken 
    - takes snapshot of the volume first 
    copies it to a temp pv 
    takes all data in temp pv and uploads it via kopia 
https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-data-mover

https://www.redhat.com/en/blog/openshift-apis-data-protection-13-data-mover

---
### Backup Resource

How we define what objects to backup  
https://velero.io/docs/main/api-types/backup/

https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-creating-backup-cr-doc

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



*explain process of what happens when it runs*
---
  pods are created as part of snapshotmovedata 
    dataupload object is created 
  first backup will take longest time to upload to gcp 
  subsequent backups will be much quicker depending on how much has changed 
  backup object will show differe phases - and then wait til it says completed. 
    can see the stuff show up in gcp bucket 
---
### Restore 

how we define what backup we want to restore from and what objects should be restored 

https://velero.io/docs/main/api-types/restore/

https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-restoring

#### important to explain 
  must specify the backup 
  exclude pods 
    avoid connection error 
      let statefulsets spin up a new pod 

*explain process of what happens when it runs*
---
  might be best to completely delete the project before restore 
  restore can take up to an hour depending on how much data there is in the backup 
  datadownload object is created 
  restore pods are spun up 
    will mount a temp pvc and download the data 
    once all data is downloaded to pv the temp pvc will detach 
    real pvc for the app will then bind to the pv 
    once download is done restore pods will go away 
  objects will come back pretty quick, data takes longest 
    statefulset will spin up pods like normal 
      there may be some time between primary/secondary 
        secondary might not connect to primary (will update if retries is fixed) 
      do not anticipate any issues connecting to pods 
    external secret should generate the secret just fine 
  