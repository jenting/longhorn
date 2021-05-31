# Title

Final consistent backup volumes with historical volume backups from the remote backup store.

## Summary

Currently, Longhorn uses a blocking way for communication with the remote backup store, so there will be some potential voluntary or 
involuntary factors (ex: network latency) impacting the functions relying on remote backup stores like listing volume backups or even 
causing further [cascading problems](###related-issues) after the backup store operation.

This enhancement is to propose an asynchronous way to pull backup volumes and historical volume backups from the remote backup store (S3/NFS) 
then persistently saved via cluster custom resources.

This can resolve the problems above mentioned by asynchronously querying the list of backup volumes and historical volume backups from the remote 
backup store for final consistent available results. It's also scalable for the costly resources created by the original blocking query operations.

### Related Issues

- https://github.com/longhorn/longhorn/issues/1761
- https://github.com/longhorn/longhorn/issues/1955
- https://github.com/longhorn/longhorn/issues/2536
- https://github.com/longhorn/longhorn/issues/2543

## Motivation

### Goals

Decrease the query latency when listing volume backups _or_ historical volume backups in the circumstances like lots of volume backups, lots of 
historical volume backups, or the network latency between the Longhorn cluster and the remote backup store.

### Non-goals [optional]

Automatically adjust the remote backup store pull period.

## Proposal

Currently, we have a setting `backupstore-poll-interval` to periodically pull the remote backup store lastest volume backups within a setting controller.

We want to leverage the same concept but pull the remote backup volumes and historical volume backups from the remote backup store and saves them into the
cluster custom resource (CR). Therefore, we'll
1. create a new Custom Resource Definition (CRD) called _backupvolumes_ to save the backup volumes and historical volume backups.
2. removes the code of the backup store monitor in the setting controller.
3. create a new controller called `backup_volume_controller` that is responsible for synchronous the remote backup store backup volumes to the cluster 
   CR _backupvolumes_.
   1. Pull the backup volumes or historical volume backups to cluster CR (the backup volumes or historical volume backups that **are in** the remote backup 
      store and **aren't in** the cluster CR).
   2. Delete the backup volumes or historical volume backups from the cluster CR (the backup volumes or historical volume backups that **are in** the cluster 
      CR and **aren't in** the remote backup store).
4. Change the [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior.
   - The `backup list` command includes listing all backup volumes and the historical volume backups and read these metadata.
     We'd like to change the `backup list` behavior to perform list only, but not read the metadata.
   - The `backup inspect` command supports read historical volume backup metadata only.
     We'd like to extend it to support read historical volume backup metadata and also read backup volume metadata.

### User Stories

Before this enhancement, when the user's environment under the circumstances that the remote backup store has lots of backup volumes _or_ 
historical volume backups, and the latency between the longhorn manager to the remote backup store is high.
Then if the user clicks the `Backup` on the GUI, the user might hit list backup volumes _or_ list historical volume backups timeout issue 
(the default timeout is 1 min).

We choose to not create a new setting for the user to increase the list timeout value is because the browser has its timeout value also.
Let's say the list backups needs 5 minutes to finish. Even we allow the user to increase the longhorn manager list timeout value, we can't change 
the browser default timeout value. Furthermore, some browser doesn't allow the user to change default timeout value like Google Chrome.

After this enhancement, when the user's environment under the circumstances that the remote backup store has lots of backup volumes _or_ 
historical volume backups, and the latency between the longhorn manager to the remote backup store is high.
Then if the user clicks the `Backup` on the GUI, the user can eventually list backup volumes _or_ list historical volume backups without timeout issue.

#### Story 1

The user environment is under the circumstances that the remote backup store has lots of backup volumes and the latency between the longhorn manager 
to the remote backup store is high. Then, the user can list all backup volumes on the GUI.

#### Story 2

The user environment is under the circumstances that the remote backup store has lots of historical volume backups and the latency between the longhorn 
manager to the remote backup store is high. Then, the user can list all historical volume backups on the GUI.

### User Experience In Detail

None.

### API changes

1. For [longhorn/backupstore](https://github.com/longhorn/backupstore)

   The current [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior are:
   - `backup ls --volume-only`: List all backup volumes and read it's metadata (`volume.cfg`). For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume-only
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
         "Name": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
         "Size": "2147483648",
         "Labels": {},
         "Created": "2021-05-12T00:52:01Z",
         "LastBackupName": "backup-c5f548b7e86b4b56",
         "LastBackupAt": "2021-05-17T05:31:01Z",
         "DataStored": "121634816",
         "Messages": {}
       },
       "pvc-7a8ded68-862d-4abb-a08c-8cf9664dab10": {
         "Name": "pvc-7a8ded68-862d-4abb-a08c-8cf9664dab10",
         "Size": "10737418240",
         "Labels": {},
         "Created": "2021-05-10T02:43:02Z",
         "LastBackupName": "backup-432f4d6afa31481f",
         "LastBackupAt": "2021-05-10T06:04:02Z",
         "DataStored": "140509184",
         "Messages": {}
       }
     }
     ```
   - `backup ls --volume <volume-name>`: List all historical volume backups and read it's metadata (`backup_backup_<backup-hash>.cfg`). For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
         "Name": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
         "Size": "2147483648",
         "Labels": {}
         "Created": "2021-05-12T00:52:01Z",
         "LastBackupName": "backup-c5f548b7e86b4b56",
         "LastBackupAt": "2021-05-17T05:31:01Z",
         "DataStored": "121634816",
         "Messages": {},
         "Backups": {
           "s3://backupbucket@minio/?backup=backup-02224cb26b794e73\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
             "Name": "backup-02224cb26b794e73",
             "URL": "s3://backupbucket@minio/?backup=backup-02224cb26b794e73\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
             "SnapshotName": "backup-23c4fd9a",
             "SnapshotCreated": "2021-05-17T05:23:01Z",
             "Created": "2021-05-17T05:23:04Z",
             "Size": "115343360",
             "Labels": {},
             "IsIncremental": true,
             "Messages": null
            },
           ...
           "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
             "Name": "backup-fa78d89827664840",
             "URL": "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
             "SnapshotName": "backup-ac364071",
             "SnapshotCreated": "2021-05-17T04:42:01Z",
             "Created": "2021-05-17T04:42:03Z",
             "Size": "115343360",
             "Labels": {},
             "IsIncremental": true,
             "Messages": null
           }
         }
       }
     }
     ```
   - `backup inspect <backup>`: Read a single volume's backup metadata (`backup_backup_<backup-hash>.cfg`). For example:
     ```shell
     $ backup inspect s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "Name": "backup-fa78d89827664840",
       "URL": "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
       "SnapshotName": "backup-ac364071",
       "SnapshotCreated": "2021-05-17T04:42:01Z",
       "Created": "2021-05-17T04:42:03Z",
       "Size": "115343360",
       "Labels": {},
       "IsIncremental": true,
       "VolumeName": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
       "VolumeSize": "2147483648",
       "VolumeCreated": "2021-05-12T00:52:01Z",
       "Messages": null
     }
     ```

   After this enhancement, the [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior are:
   - `backup ls --volume-only`: List all backup volume names. For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume-only
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {},
       "pvc-7a8ded68-862d-4abb-a08c-8cf9664dab10": {}
     }
     ```
   - `backup ls --volume <volume-name>`: List all historical volume backup names. For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
         "Backups": {
           "s3://backupbucket@minio/?backup=backup-02224cb26b794e73\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {},
           ...
           "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {}
         }
       }
     }
     ```
   - `backup inspect <metadata-path>`: Read a single backup volume metadata (`volume.cfg`) or read a single volume's backup metadata (`backup_backup_<backup-hash>.cfg`).
     - Read a single backup volume metdata (`volume.cfg`). For example:
       ```shell
       $ backup inspect s3://backupbucket@minio/?volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
       {
         "Name": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
         "Size": "2147483648",
         "Labels": {},
         "Created": "2021-05-12T00:52:01Z",
         "LastBackupName": "backup-c5f548b7e86b4b56",
         "LastBackupAt": "2021-05-17T05:31:01Z",
         "DataStored": "121634816",
         "Messages": {}
       }
       ```
     - Read a single volume's backup metadata (`backup_backup_<backup-hash>.cfg`). For example:
       ```shell
       $ backup inspect s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
       {
         "Name": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
         "Size": "2147483648",
         "Labels": {},
         "Created": "2021-05-12T00:52:01Z",
         "LastBackupName": "backup-c5f548b7e86b4b56",
         "LastBackupAt": "2021-05-17T05:31:01Z",
         "DataStored": "121634816",
         "Messages": {},
         "Backups": {
           "s3://backupbucket@minio/?backup=backup-02224cb26b794e73\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
             "Name": "backup-02224cb26b794e73",
             "URL": "s3://backupbucket@minio/?backup=backup-02224cb26b794e73\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
             "SnapshotName": "backup-23c4fd9a",
             "SnapshotCreated": "2021-05-17T05:23:01Z",
             "Created": "2021-05-17T05:23:04Z",
             "Size": "115343360",
             "Labels": {},
             "IsIncremental": true,
             "Messages": null
           }
         }
       }
       ```

   Generally speaking, we want to separate the **list** and **read** command.

2. The Longhorn manager HTTP endpoints.

   The below HTTP endpoints current behavior are:
   - **GET** `/v1/backupvolumes`: read all backup volumes from the remote cluster store.
   - **GET** `/v1/backupvolumes/{volName}`: read a single backup volume from the remote cluster store.
   - **DELETE** `/v1/backupvolumes/{volName}`: delete a single backup volume from the remote cluster store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a single volume backups from the remote cluster store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a single volume backup from the remote cluster store.
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete a single volume backup from the remote cluster store.
  
   After this enhancement, below HTTP endpoints behavior are:
   - **GET** `/v1/backupvolumes`: read all backup volumes from the cluster custom resource (CR).
   - **GET** `/v1/backupvolumes/{volName}`: read a single backup volume from the cluster custom resource (CR).
   - **DELETE** `/v1/backupvolumes/{volName}`: delete a single backup volume from the cluster custom resource (CR) and also from the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a single volume backups from the cluster custom resource (CR).
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a single volume backup from the cluster custom resource (CR).
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete a single volume backup from the cluster custom resource (CR) and also from the remote backup store.

## Design

### Implementation Overview

1. Create a new CRD called `backupvolumes.longhorn.io`.

   We'll create a new Custom Resource Definition (CRD) called `backupvolumes.longhorn.io` to save pulled backup volumes metadata and its historical volume backups metadata as the Custom Resource (CR). For each backup volume, we'll create a new CR.
   - `metadata.name`: the backup volume name.
   - `spec.backupStoreURL`: the backup store URL.
   - `spec.pollInterval`: the backup store poll interval.
   - `status.size`: the backup volume size.
   - `status.labels`: the backup volume labels.
   - `status.createTimestamp`: the backup volume creation timestamp.
   - `status.lastBackupName`: the latest volume backup name.
   - `status.lastBackupTimestamp`: the latest volume backup timestamp.
   - `status.dataStored`: the backup volume block count.
   - `status.messages`: the error messages when list backup volumes or volume backups, or the error message when inspect backup volume metadata or volume backup metadata.
   - `status.backups`: a key-value map of volume backups (key: volume backup name, value: the volume backup metadata).
     - `status.backups[volumeBackupName].url`: the volume backup url.
     - `status.backups[volumeBackupName].snapshotName`: the snapshot name.
     - `status.backups[volumeBackupName].snapshotCreateTimestamp`: the snapshot creation timestamp.
     - `status.backups[volumeBackupName].backupCreateTimestamp`: the backup creation timestamp.
     - `status.backups[volumeBackupName].size`: the volume backup size.
     - `status.backups[volumeBackupName].labels`: the volume backup labels.
     - `status.backups[volumeBackupName].volumeName`: the backup volume name.
     - `status.backups[volumeBackupName].volumeSize`: the backup volume size.
     - `status.backups[volumeBackupName].volumeCreateTimestamp`: the backup volume creation timestamp.
     - `status.backups[volumeBackupName].messages`: the error messages when list volume backup or inspect volume backup metadata.
   - `status.lastSyncedTimestamp`: records the last time the backup store contents were synced into the cluster.

2. At the existed `setting_controller`.
   
   We'll remove the existed code on watches the change of CR `settings.longorn.io` field `backup-target`, and `backup-target-credential-secret`, and `backupstore-poll-interval`. Also, we'll remove the backup store monitor in the setting controller. The backup store monitor run as a goroutine to 
   periodically pull the latest volume backup name and volume backup timestamp, and updates the `status.LastBackup` and `status.LastBackupAt` to the 
   CR `volumes.longhorn.io` if it's a DR volume.
   
3. Create a new controller called `backup_volume_controller`.
   
   It watches the change of the settings `backup-target`, and `backup-target-credential-secret`, and `backupstore-poll-interval`. Within the controller, it runs:
   1. List backup volumes in the cluster CR `backupvolumes.longhorn.io`.
   2. List backup volumes from the remote backup store.
   3. Find the different backup volumes `backupVolumesToPull` that are in the remote backup store and aren't in the cluster CR `backupvolumes.longhorn.io`.
   4. Find the different backup volumes `backupVolumesToDelete` that are in the cluster CR `backupvolumes.longhorn.io` and aren't in the remote backup store.
   5. Loop the `backupVolumesToPull`, read the backup volume metadata from the remote backup store, and create a new CR in `backupvolumes.longhorn.io`.
   6. Loop the `backupVolumesToDelete`, delete the CR in `backupvolumes.longhorn.io`.
   7. Loop all backup volumes in the cluster CR `backupvolumes.longhorn.io`, for each backup volume (BV):
      1. Get the backup volume in the cluster CR `backupvolumes.longhorn.io`.
      2. List historical volume backups in the backup volume CR `backupvolumes.longhorn.io` field `status.backups`.
      3. List historical volume backups under the backup volume (BV)from the remote backup store.
      4. Find the different volume backups `volumeBackupsToPull` that are in the remote backup store and aren't in the backup volume CR field `status.backups`.
      5. Find the different volume backups `volumeBackupsToDelete` that are in the backup volume CR `status.backups` and aren't in the remote backup store.
      6. Loop the `volumeBackupsToPull`, read the historical volume backup metadata from the remote backup store, and add to the backup volume CR field `status.backups`.
      7. Loop the `volumeBackupsToDelete`, delete the backup volume CR `status.backups`.
      8. Update the backup volume (BV) CR `status.backups`.

4. For the Longhorn manager HTTP endpoints:
   - **GET** `/v1/backupvolumes`: read all backup volumes from the cluster custom resource (CR).
   - **GET** `/v1/backupvolumes/{volName}`: read a single backup volume from the cluster custom resource (CR).
   - **DELETE** `/v1/backupvolumes/{volName}`: delete a single backup volume from the cluster custom resource (CR) and also from the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a single volume backups from the cluster custom resource (CR).
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a single volume backup from the cluster custom resource (CR).
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete a single volume backup from the cluster custom resource (CR) and also from the remote backup store.

### Test plan

With over 1k backup volumes and over 1k historical volume backups under pretty high network latency (700-800ms per operation)
from longhorn manager to the remote backup store:
1. The user can list backup volumes and list historical volume backups on the Longhorn GUI.
2. When the user deletes a backup volume on the Longhorn GUI:
   1. list backup volumes on the Longhorn GUI, the deleted one does not exist immediately.
   2. check the remote backup store, the backup volume will be deleted after a while.
3. When the user deletes a historical volume backup on the Longhorn GUI:
   1. list historical volume backups on the Longhorn GUI, the deleted one does not exist immediately.
   2. check the remote backup store, the historical backup will be deleted after a while.
4. When the user deletes a backup volume or deletes a historical volume backup on the remote backup store manually.
   After `backupstore-poll-interval` seconds, list backup volumes or list historical volume backups on the Longhorn GUI, the deleted one does not exist anymore.
5. Create two cluster (clusterA and clusterB) both points to the same remote backup store.
   1. At cluster A, create a volume and run a recurring backup to the remote backup store.
   2. At cluster B, after `backupstore-poll-interval` seconds, the user can list backup volumes or list volume backups on the Longhorn GUI.
   3. At cluster B, create a DR volume from the backup volume.
   4. At cluster B, check the DR volume `status.LastBackup` and `status.LastBackupAt` be updated periodically.
   5. At cluster A, delete the backup volume on the GUI.
   6. At cluster B, after `backupstore-poll-interval` seconds, the deleted backup volume does not exist on the Longhorn GUI.
   7. At cluster B, the DR volume `status.LastBackup` and `status.LastBackupAt` won't be updated anymore.

### Upgrade strategy

Before this enhancement, the user might set `backupstore-poll-interval` to `0` to reduce the request to the remote backup store.
This is majorly for the user who wants to reduce the cost on access AWS S3.

After this enhancement, if the user ever configured the setting `backupstore-poll-interval` to `0`, then the user can't see the backup volumes
on the Longhorn GUI. We'll add a warning message on the Longhorn GUI to notify the user to configure the `backupstore-poll-interval` greater 
than `0`, so the backup volume controller can synchronous the backup volumes from the remote backup store.

## Note

With this enhancement, the CR _backupvolumes.longhorn.io_ is updated when the asynchronous timer be triggered.
However, the user might want to trigger it immediately. But currently, we haven't found a good way to let the user force trigger it now.
