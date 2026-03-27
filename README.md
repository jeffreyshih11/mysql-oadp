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

that led us to 
**somethign about what oadp is** 
https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore

- it comes with openshift 
- kubernetes native 
- has minimal downtime 
- can backup all objects in a project, not just the data 
## Installing OAPD

prerequisites 
- have service account for gcp with bucket access 
    - put creds in an external secret 
            - gcp-installer-sa.json 
- have a bucket created in gcp already 

https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#installing-oadp

https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#configuring-oadp-with-google-cloud

## once adp is installed 
### create dataprotectionapplication 
- this will create a backupStorageLocation as well 
    
    this verifies if we can connect to the bucket 
      phase: available is good 

### Data Mover 
The important thing. able to take advantage of the split second speed of a volume snapshot while still uploading the data to gcp. 

if we dont do this we rely solely on volumesnapshots or completel file system backup

snapshot only - only reside on the cluster so is vulnerable to cluster wide disasters 
fs backup - super slow to check and upload every file even with kopia. presents an unacceptable amount of downtime to the database every time a backup is taken 

https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-data-mover

https://www.redhat.com/en/blog/openshift-apis-data-protection-13-data-mover


### Backup Resource

How we define what objects to backup  
https://velero.io/docs/main/api-types/backup/

https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-creating-backup-cr-doc


- ttl
    - how long the backup exists in gcp bucket 
    - does not apply to the data stored in volumes 
- storage location 
    - which location to use if multiple - but wont 
- snapshotmovedata 
    - takes snapshot of the volume first 
    copies it to a temp pv 
    takes all data in temp pv and uploads it via kopia 
- excludedResources
    no secrets because that data should be stored elsewhere, even if encrypted. anything required should be an external secret anyways
- orderedResources
    necessary to avoid split brain issue on restore 
    only doing pods because when you backup a pod velero will also backup any mounted pvc before doing the next one 
- hooks 
    - necessary to freeze the db to avoid corruption 
    time between hooks usuall ~15s 
      pre - do the read lock and keep the process in background because otherwise the the read lock will only run tfor that exact second and release. so the sleep is necessary 
      post - kill the process and resume normal functionality 

### Schedule Resource

Create backups on a schedule - essentially the backup resource with cron 

https://velero.io/docs/v1.3.0/api-types/schedule/


- this will be in the helm charts as a template 
    new values 
      backup - true/false 
      cron schedule 
    templatize 
      all names 
      namespace 
      maybe ttl 


#### important things to explain  
- created backup objects and naming convention 

#### explain process 

  pods are created as part of snapshotmovedata 
    dataupload object is created 
  first backup will take longest time to upload to gcp 
  subsequent backups will be much quicker depending on how much has changed 

### Restore 

how we define what backup we want to restore from and what objects should be restored 

https://velero.io/docs/main/api-types/restore/

https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/backup_and_restore/oadp-application-backup-and-restore#oadp-restoring

  will want to make it a workflow 
    user inputs 
      what namespace/app
      what backup to use
    might want to find a way to get existing backups?
  also templatize it as helm file 

#### important to explain 
  must specify the backup 
  exclude pods 
    avoid connection error 
      let statefulsets spin up a new pod 

explain process 
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
  