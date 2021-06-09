# Async Pull/Push Backups From/To The Remote Backup Store

## Summary

Currently, Longhorn uses a blocking way for communication with the remote backup store, so there will be some potential voluntary or 
involuntary factors (ex: network latency) impacting the functions relying on remote backup stores like listing volume backups or even 
causing further [cascading problems](###related-issues) after the backup store operation.

This enhancement is to propose an asynchronous way to pull backup volumes and volume snapshot backups from the remote backup store (S3/NFS) 
then persistently saved via cluster custom resources.

This can resolve the problems above mentioned by asynchronously querying the list of backup volumes and volume snapshot backups from the remote 
backup store for final consistent available results. It's also scalable for the costly resources created by the original blocking query operations.

### Related Issues

- https://github.com/longhorn/longhorn/issues/1761
- https://github.com/longhorn/longhorn/issues/1955
- https://github.com/longhorn/longhorn/issues/2536
- https://github.com/longhorn/longhorn/issues/2543

## Motivation

### Goals

Decrease the query latency when listing volume backups _or_ volume snapshot backups in the circumstances like lots of volume backups, lots of 
volume snapshot backups, or the network latency between the Longhorn cluster and the remote backup store.

### Non-goals

Automatically adjust the remote backup store pull period.

## Proposal

1. Change the [longhorn/backupstore](https://github.com/longhorn/backupstore) _list_ command behavior, and add two _inspect_ and _head_ commands.
   - The `backup list` command includes listing all backup volumes and the volume snapshot backups and read these metadata.
     We'll change the `backup list` behavior to perform list only, but not read the metadata.
   - Add a new `backup inspect-volume` command to support read backup volume metadata.
   - Add a new `backup head` command to support get the file's last modified time.
2. Create a BackupTarget CRD _backuptargets.longhorn.io_ to save the backup target URL, credential secret, and poll interval.
3. Create a BackupVolume CRD _backupvolumes.longhorn.io_ to save to backup volume metadata.
4. Create a Backup CRD _backups.longhorn.io_ to save the volume snapshot backup metadata.
5. At the existed `setting_controller`, which is responsible creating/deleting BackupTarget CR metadata+spec according to the settings `backup-target`, `backup-target-credential-secret`, and `backupstore-poll-interval`.
6. Create a new controller `backup_target_controller`, which is responsible creating/deleting the BackupVolume CR metadata+spec, and updating the BackupTarget CR status. 
7. Create a new controller `backup_volume_controller`, which is responsible creating/deleting Backup CR metadata+spec, updating BackupVolume CR status, and deleting backup volume from the remote backup store.
8. Create a new controller `backup_controller`, which is responsible calling longhorn engine/replica to perform snapshot backup to the remote backup store, updating the Backup CR status, and deleting backup from the remote backup store.
9. The HTTP endpoints CRUD methods related to backup volume and volume snapshot backups will interact with BackupVolume CR and Backup CR instead interact with the remote backup store.

### User Stories

Before this enhancement, when the user's environment under the circumstances that the remote backup store has lots of backup volumes _or_ 
volume snapshot backups, and the latency between the longhorn manager to the remote backup store is high.
Then if the user clicks the `Backup` on the GUI, the user might hit list backup volumes _or_ list volume snapshot backups timeout issue 
(the default timeout is 1 min).

We choose to not create a new setting for the user to increase the list timeout value is because the browser has its timeout value also.
Let's say the list backups needs 5 minutes to finish. Even we allow the user to increase the longhorn manager list timeout value, we can't change 
the browser default timeout value. Furthermore, some browser doesn't allow the user to change default timeout value like Google Chrome.

After this enhancement, when the user's environment under the circumstances that the remote backup store has lots of backup volumes _or_ 
volume snapshot backups, and the latency between the longhorn manager to the remote backup store is high.
Then if the user clicks the `Backup` on the GUI, the user can eventually list backup volumes _or_ list volume snapshot backups without timeout issue.

#### Story 1

The user environment is under the circumstances that the remote backup store has lots of backup volumes and the latency between the longhorn manager 
to the remote backup store is high. Then, the user can list all backup volumes on the GUI.

#### Story 2

The user environment is under the circumstances that the remote backup store has lots of volume snapshot backups and the latency between the longhorn 
manager to the remote backup store is high. Then, the user can list all volume snapshot backups on the GUI.

#### Story 3

The user creates snapshot backup on the Longhorn GUI. Now the snapshot backup will create a Backup CR, then the backup_controller reconcile it to call Longhorn engine/replica to perform snapshot backup to the remote backup store.

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
   - `backup ls --volume <volume-name>`: List all volume snapshot backups and read it's metadata (`backup_backup_<backup-hash>.cfg`). For example:
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
   - `backup ls --volume <volume-name>`: List all volume snapshot backup names. For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
         "Backups": {
           "backup-02224cb26b794e73": {},
           ...
           "backup-fa78d89827664840": {}
         }
       }
     }
     ```
   - `backup inspect-volume <volume>`: Read a single backup volume metadata (`volume.cfg`). For example:
     ```shell
     $ backup inspect-volume s3://backupbucket@minio/?volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
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
   - `backup head <filepath>`: Get the filepath filesystem metadata. For example:
     ```shell
     $ backup head s3://backupbucket@minio/?volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "ModifiedTime": "2021-05-12T00:52:01Z",
     }
     $ backup head s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "ModifiedTime": "2021-05-17T04:42:01Z"
     }
     ```

   Generally speaking, we want to separate the **list** and **read** command and add the **head** command.

2. The Longhorn manager HTTP endpoints.

   The below HTTP endpoints current behavior are:
   - **GET** `/v1/backupvolumes`: read all backup volumes from the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}`: read a backup volume from the remote backup store.
   - **DELETE** `/v1/backupvolumes/{volName}`: delete a backup volume from the remote backup store.
   - **POST** `/v1/volumes/{volName}?action=snapshotBackup`: create a volume snapshot backup to the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a list of volume snapshot backups from the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a volume snapshot backup from the remote backup store.
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete a volume snapshot backup from the remote backup store.
  
   After this enhancement, below HTTP endpoints behavior are:
   - **GET** `/v1/backupvolumes`: read all the BackupVolume CRs with label filter `synced=true`.
   - **GET** `/v1/backupvolumes/{volName}`: read a BackupVolume CR with the given volume name and with label filter `synced=true`.
   - **DELETE** `/v1/backupvolumes/{volName}`: delete the BackupVolume CR, the backup_volume_controller reconcile it to delete the backup volume from the remote backup store.
   - **POST** `/v1/volumes/{volName}?action=snapshotBackup`: create a new Backup CR, the backup_controller reconcile it to create a volume snapshot backup to the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a list of Backup CRs with the label filter `volume=<backup-volume-name>` and `synced=true`.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a Backup CR with the label filter `volume=<backup-volume-name>`. and `synced=true`.
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete the Backup CR, the backup_controller reconcile it to delete the backup from the remote backup store.

## Design

### Implementation Overview

1. Create a new BackupTarget CRD `backuptargets.longhorn.io`.
   
   - `metadata.name`: the hashed name of backup target URL.
   - `spec.backupTargetURL`: the backup target URL.
   - `spec.credentialSecret`: the backup target credential secret.
   - `spec.pollInterval`: the backup target poll interval.
   - `status.lastSyncedTimestamp`: records the last time the backup target was running the reconcile process.

2. Create a new BackupVolume CRD `backupvolumes.longhorn.io`.
   - `metadata.name`: the backup volume name.
   - `metadata.labels`:
     - `synced=<true|false>`: this label indicates the backup volume was synced with the remote backup store.   - 
   - `metadata.ownerReferences`: the owning object.
   - `status.size`: the backup volume size.
   - `status.labels`: the backup volume labels.
   - `status.createTimestamp`: the backup volume creation timestamp.
   - `status.lastBackupName`: the latest volume backup name.
   - `status.lastBackupTimestamp`: the latest volume backup timestamp.
   - `status.dataStored`: the backup volume block count.
   - `status.messages`: the error messages when call longhorn engine on list or inspect backup volumes.
   - `status.lastModifiedTimestamp`: records the last time the backup volume metadata was modified.
   - `status.lastSyncedTimestamp`: records the last time the backup volume was synced into the cluster.

3. Create a new Backup CRD `backups.longhorn.io`.

   - `metadata.name`: the volume snapshot backup name.
   - `metadata.labels`:
     - `volume=<backup-volume-name>`: this label indicates which backup volume the volume snapshot backup belongs to.
     - `synced=<true|false>`: this label indicates the backup was synced with the remote backup store.
   - `metadata.ownerReferences`: the owning object.
   - `spec.snapshotName`: the volume snapshot name.
   - `spec.labels`: the labels of volume snapshot backup.
   - `status.url`: the volume snapshot backup url.
   - `status.snapshotName`: the volume snapshot name.
   - `status.snapshotCreateTimestamp`: the volume snapshot creation timestamp.
   - `status.backupCreateTimestamp`: the volume snapshot backup creation timestamp.
   - `status.size`: the volume snapshot size.
   - `status.labels`: the labels of volume snapshot backup.
   - `status.messages`: the error messages when call longhorn engine on list or inspect volume snapshot backups.
   - `status.lastSyncedTimestamp`: records the last time the volume snapshot backup was synced into the cluster.

4. At the existed `setting_controller`.
   
   Watches the changes of Setting CR `settings.longorn.io` field `backup-target`, `backup-target-credential-secret`, and `backupstore-poll-interval`. The setting controller is responsible to create/update/delete BackupTarget CR metadata and spec.

5. Create a new `backup_target_controller`.
   
   Watches the change of BackupTarget CR `backuptargets.longhorn.io`. The backup target controller is responsible creating/deleting BackupVolume CR metadata. The reconcile loop steps are:
   - **AddFunc**/**UpdateFunc**: Check if `the current timestamp - BackupTarget CR status.lastSyncedTimestamp >= spec.pollInterval > 0`.
     - If no, abort the current reconcile process.
     - If yes:
        1. Call the longhorn engine to list all the backup volumes `backup ls --volume-only` from the remote backup store `backupStoreBackups`.
        2. List in cluster BackupVolume CRs `clusterBackupVolumes`.
        3. Find the difference backup volumes `backupVolumesToPull = backupStoreBackups - clusterBackupVolumes` and create BackupVolume CR `metadata.name` and `metadata.ownerReferences.Kind=BackupTarget`.
        4. Find the difference backup volumes `backupVolumesToDelete = clusterBackupVolumes - backupStoreBackups` and delete BackupVolume CR.
        5. Updates the BackupTarget CR `status.lastSyncedTimestamp`.
   - **DeleteFunc**: We don't need the DeleteFunc function because if the user changes/removes the backup target URL, the BackupTarget CR be deleted. Then all the related BackupVolume CR will be deleted because the BackupVolume CR owner reference is the BackupTarget CR.

6. For the Longhorn manager HTTP endpoints:
   - **DELETE** `/v1/backupvolumes/{volName}`:
     1. Add the finalizer to the BackupVolume CR with the given volume name.
     2. Delete a BackupVolume CR with the given volume name.

7. Create a new controller `backup_volume_controller`.
   
   Watches the change of BackupVolume CR `backupvolumes.longhorn.io`. The backup volume controller is responsible to update BackupVolume CR status field, and creating/deleting Backup CR metadata. The reconcile loop steps are:
   - **AddFunc**: 
     1. List in cluster BackupVolume CRs _with_ label filter `synced=false`. For each BackupVolume CR `metadata.name`:
        1. Call the longhorn engine to read backup volume's metadata `backup inspect-volume <volume-name>`
        2. Updates the BackupVolume CR status field according to the backup volumes' metadata
        3. Updates the BackupVolume CR `status.lastSyncedTimestamp`.
        4. Change the label from `synced=false` to `synced=true`.
     2. Check if `the current timestamp - BackupTarget CR status.lastSyncedTimestamp >= spec.pollInterval > 0`.
        - If no, abort this step.
        - If yes, list in cluster BackupVolume CRs _without_ the label `synced=false` to get the volume name `volume-name` from the `metadata.name` field. For each BackupVolume CR `metadata.name`:
          1. Call the longhorn engine to get the last modified time `backup head <volume-name>`.
             - If the last modified time == `status.lastModifiedTimestamp`, skip this volume.
             - Otherwise:
                1. Call the longhorn engine to read all backup volumes' metadata `backup inspect-volume <volume-name>`.
                2. Updates the BackupVolume CR status field according to the backup volumes' metadata, and also updates the BackupVolume CR `status.lastSyncedTimestamp`.
                3. Add label `synced=true`.
                4. Call the longhorn engine to list all the volume snapshot backups `backup ls --volume <volume-name>` from the remote backup store `backupStoreVolumeSnapshotBackups`.
                5. List in cluster Backup CRs `clusterVolumeSnapshotBackups`.
                6. Find the difference volume snapshot backups `volumeSnapshotBackupsToPull = backupStoreVolumeSnapshotBackups - clusterVolumeSnapshotBackups` and create Backup CR `metadata.name` + `metadata.ownerReferences.Kind=BackupVolume`.
                7. Find the difference volume snapshot backups `volumeSnapshotBackupsToDelete = clusterVolumeSnapshotBackups - backupStoreVolumeSnapshotBackups` and delete Backup CR.
   - **DeleteFunc**: If the finalizer has been set, delete the backup volume from the remote backup store `backup rm --volume <volume-name> <url>`. After that, remove the finalizer.

8. For the Longhorn manager HTTP endpoints:
   - **POST** `/v1/volumes/{volName}?action=snapshotBackup`:
     1. Generate the backup name.
     2. Create a new BackupVolume CR if not present with
        - `metadata.name`
        - `metadata.ownerReferences.Kind=BackupTarget`
        - `metadata.Labels["synced"]=false`
     3. Create a new Backup CR with
        - `metadata.name`
        - `metadata.ownerReferences.Kind=BackupVolume`
        - `metadata.Labels["synced"]=false`
        - `spec.snapshotName`
        - `spec.labels`
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`:
     1. Add the finalizer to the Backup CR with the given backup name.
     2. Delete a Backup CR with the given backup name.

9. Create a new controller `backup_controller`.

   Watches the change of Backup CR `backups.longhorn.io`. The backup controller is responsible to update the Backup CR status field and create/delete backup to/from the remote backup store. The reconcile loop steps are:
   - **AddFunc**:
     1. List in cluster Backup CRs _with_ label filter `synced=false`. For each Backup CR `metadata.name`:
        1. Call the longhorn engine to perform snapshot backup to the remote backup store.
        2. Call the longhorn engine to read the volume snapshot backup metadata `backup inspect <backup-url>`.
        3. Updates the Backup CR status field according to the volume snapshot backup metadata.
        4. Updates the Backup CR `status.lastSyncedTimestamp`.
        5. Change the label from `synced=false` to `synced=true`.
     2. Check if the `current timestamp - BackupVolume CR status.lastSyncedTimestamp >= spec.pollInterval > 0`.
        - If no, abort the current snapshot backup process.
        - If yes, list in cluster Backup CRs _without_ label filter `synced=false`. For each Backup CR `metadata.name`:
          - If the Backup CR `status.lastSyncedTimestamp` is not nil, skip it.
          - If the Backup CR `status.lastSyncedTimestamp` is nil.
             1. Call the longhorn engine to read the volume snapshot backup metadata `backup inspect <backup-url>`.
             2. Updates the Backup CR status field according to the volume snapshot backup metadata.
             3. Updates the Backup CR `status.lastSyncedTimestamp`.
             4. Add label `synced=true`.
   - **DeleteFunc**: If the finalizer has been set, delete the backup from the remote backup store `backup rm <backup-url>`. After that, remove the finalizer.

10. For the Longhorn manager HTTP endpoints:
   - **GET** `/v1/backupvolumes`: read all the BackupVolume CRs with label filter `synced=true`.
   - **GET** `/v1/backupvolumes/{volName}`: read a BackupVolume CR with the given volume name with label filter `synced=true`.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a list of Backup CRs with the label filter `volume=<backup-volume-name>` and `synced=true`.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a Backup CR with the label filter `volume=<backup-volume-name>` and `synced=true`

### Test plan

With over 1k backup volumes and over 1k volume snapshot backups under pretty high network latency (700-800ms per operation)
from longhorn manager to the remote backup store:
- Create a single cluster with 2 volumes (vol-A and vol-B).
   1. The user configures the remote backup store S3 (s3://backupbucket@us-east-1/).
   2. The user creates two volume snapshots on vol-A and vol-B.
   3. The user can see the backup volume for vol-A and vol-B in backups GUI.
   4. The user can see the two volume snapshot backups under vol-A and vol-B in backups GUI.
   5. When the user deletes one of the volume snapshot backup of vol-A on the Longhorn GUI, the delete one will be deleted after the remote backup store backup be deleted.
   6. When the user deletes backup volume vol-A on the Longhorn GUI, the backup volume will be deleted after the remote backup store backup volume be deleted, and the volume snaphost backup of vol-A will be delete also.
   7. The user can see the backup volume for vol-B in backups GUI.
   8. The user can see two volume snapshot backups under vol-B in backups GUI.
   9. The user changes the remote backup store to another S3 (s3://backupbucket@us-east-2/), the user can't see the backup volume and volume snapshot backup of vol-B in backups GUI.
   10. The user configures the backstore poll interval to 1 mins.
   11. The user changes the remote backup store to original one S3 (s3://backupbucket@us-east-1/), after 1 mins later, the user can see the backup volume and volume snapshot backup of vol-B.
- Create two clusters (cluster-A and cluster-B) both points to the same remote backup store.
   1. At cluster A, create a volume and run a recurring backup to the remote backup store.
   2. At cluster B, after `backupstore-poll-interval` seconds, the user can list backup volumes or list volume backups on the Longhorn GUI.
   3. At cluster B, create a DR volume from the backup volume.
   4. At cluster B, check the DR volume `status.LastBackup` and `status.LastBackupAt` be updated periodically.
   5. At cluster A, delete the backup volume on the GUI.
   6. At cluster B, after `backupstore-poll-interval` seconds, the deleted backup volume does not exist on the Longhorn GUI.
   7. At cluster B, the DR volume `status.LastBackup` and `status.LastBackupAt` won't be updated anymore.

### Upgrade strategy

None.

## Note

With this enhancement, if the user configures a new remote backup store that already contains Longhorn backups, then the BackupVolume CR status and Backup CR status are updated when `the current time - status.lastSyncedTime >= pollInterval`. The user might want to trigger it immediately. The only way now is to configure the value of `backupstore-poll-interval` smaller.
