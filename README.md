# mysql-oadp
How to backup and restore MySQL replication instances running in OpenShift 4.x using OADP



oadp backup for mysql 
**somethign about what oadp is** 
**something about mysql backup procedure**

install oadp via rhacm policy 
prerequisites 
    have service account for gcp with bucket access 
        put creds in an external secret 
            gcp-installer-sa.json 
    have a bucket created in gcp already 

once adp is installed 
  create dataprotectionapplication 


  this will create a backupStorageLocation as well 
    this verifies if we can connect to the bucket 
      phase: available is good 


create backup schedule
  this will be in the helm charts as a template 
    new values 
      backup - true/false 
      cron schedule 
    templatize 
      all names 
      namespace 
      maybe ttl 


important things to explain  
  created backup objects and naming convention 
  ttl
    how long the backup exists in gcp bucket 
    does not apply to the data stored in volumes 
  storage location 
    which location to use if multiple - but wont 
  snapshotmovedata 
    takes snapshot of the volume first 
    copies it to a temp pv 
    takes all data in temp pv and uploads it via kopia 
  excludedResources
    no secrets because that data should be stored elsewhere, even if encrypted. anything required should be an external secret anyways
  orderedResources
    necessary to avoid split brain issue on restore 
    only doing pods because when you backup a pod velero will also backup any mounted pvc before doing the next one 
  hooks 
    necessary to freeze the db to avoid corruption 
    time between hooks usuall ~15s 
      pre - do the read lock and keep the process in background because otherwise the the read lock will only run tfor that exact second and release. so the sleep is necessary 
      post - kill the process and resume normal functionality 

explain process 
  pods are created as part of snapshotmovedata 
    dataupload object is created 
  first backup will take longest time to upload to gcp 
  subsequent backups will be much quicker depending on how much has changed 

restore 
  will want to make it a workflow 
    user inputs 
      what namespace/app
      what backup to use
    might want to find a way to get existing backups?
  also templatize it as helm file 

important to explain 
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
  