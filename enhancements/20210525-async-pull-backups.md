# Async Pull/Push Backups From/To The Remote Backup Target

## Summary

Currently, Longhorn uses a blocking way for communication with the remote backup target, so there will be some potential voluntary or involuntary factors (ex: network latency) impacting the functions relying on remote backup target like listing backups or even causing further [cascading problems](###related-issues) after the backup target operation.

This enhancement is to propose an asynchronous way to pull backup volumes and backups from the remote backup target (S3/NFS) then persistently saved via cluster custom resources.

This can resolve the problems above mentioned by asynchronously querying the list of backup volumes and backups from the remote backup target for final consistent available results. It's also scalable for the costly resources created by the original blocking query operations.

### Related Issues

- https://github.com/longhorn/longhorn/issues/1761
- https://github.com/longhorn/longhorn/issues/1955
- https://github.com/longhorn/longhorn/issues/2536
- https://github.com/longhorn/longhorn/issues/2543

## Motivation

### Goals

Decrease the query latency when listing backup volumes _or_ backups in the circumstances like lots of backup volumes, lots of backups, or the network latency between the Longhorn cluster and the remote backup target.

### Non-goals

- Automatically adjust the backup target poll interval.
- Supports multiple backup targets.
- Supports API pagination for listing backup volumes and backups.

## Proposal

1. Change the [longhorn/backupstore](https://github.com/longhorn/backupstore) _list_ command behavior, and add _inspect-volume_ command.
   - The `backup list` command includes listing all backup volumes and the backups and read these config.
     We'll change the `backup list` behavior to perform list only, but not read the config.
   - Add a new `backup inspect-volume` command to support read backup volume config.
   - Add a new `backup head` command to support get config metadata.
2. Create a BackupTarget CRD _backuptargets.longhorn.io_ to save the backup target URL, credential secret, and poll interval.
3. Create a BackupVolume CRD _backupvolumes.longhorn.io_ to save to backup volume config.
4. Create a Backup CRD _backups.longhorn.io_ to save the backup config.
5. At the existed `setting_controller`, which is responsible creating/updating the default BackupTarget CR with settings `backup-target`, `backup-target-credential-secret`, and `backupstore-poll-interval`.
6. Create a new controller `backup_target_controller`, which is responsible for creating/deleting the BackupVolume CR.
7. Create a new controller `backup_volume_controller`, which is responsible:
   1. deleting Backup CR and deleting backup volume from remote backup target if delete BackupVolume CR event comes in.
   2. updating BackupVolume CR status.
   3. creating/deleting Backup CR.
8. Create a new controller `backup_controller`, which is responsible:
   1. deleting backup from remote backup target if delete Backup CR event comes in.
   2. calling longhorn engine/replica to perform snapshot backup to the remote backup target.
   3. updating the Backup CR status, and deleting backup from the remote backup target.
9.  The HTTP endpoints CRUD methods related to backup volume and backup will interact with BackupVolume CR and Backup CR instead interact with the remote backup target.

### User Stories

Before this enhancement, when the user's environment under the circumstances that the remote backup target has lots of backup volumes _or_ backups, and the latency between the longhorn manager to the remote backup target is high.
Then when the user clicks the `Backup` on the GUI, the user might hit list backup volumes _or_ list backups timeout issue (the default timeout is 1 minute).

We choose to not create a new setting for the user to increase the list timeout value is because the browser has its timeout value also.
Let's say the list backups needs 5 minutes to finish. Even we allow the user to increase the longhorn manager list timeout value, we can't change the browser default timeout value. Furthermore, some browser doesn't allow the user to change default timeout value like Google Chrome.

After this enhancement, when the user's environment under the circumstances that the remote backup target has lots of backup volumes _or_ backups, and the latency between the longhorn manager to the remote backup target is high.
Then when the user clicks the `Backup` on the GUI, the user can eventually list backup volumes _or_ list backups without a timeout issue.

#### Story 1

The user environment is under the circumstances that the remote backup target has lots of backup volumes and the latency between the longhorn manager to the remote backup target is high. Then, the user can list all backup volumes on the GUI.

#### Story 2

The user environment is under the circumstances that the remote backup target has lots of backups and the latency between the longhorn manager to the remote backup target is high. Then, the user can list all backups on the GUI.

#### Story 3

The user creates a backup on the Longhorn GUI. Now the backup will create a Backup CR, then the backup_controller reconciles it to call Longhorn engine/replica to perform a backup to the remote backup target.

### User Experience In Detail

None.

### API changes

1. For [longhorn/backupstore](https://github.com/longhorn/backupstore)

   The current [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior are:
   - `backup ls --volume-only`: List all backup volumes and read it's config (`volume.cfg`). For example:
     ```json
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
   - `backup ls --volume <volume-name>`: List all backups and read it's config (`backup_backup_<backup-hash>.cfg`). For example:
     ```json
     $ backup ls s3://backupbucket@minio/ --volume pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
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
   - `backup inspect <backup>`: Read a single backup config (`backup_backup_<backup-hash>.cfg`). For example:
     ```json
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
     ```json
     $ backup ls s3://backupbucket@minio/ --volume-only
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {},
       "pvc-7a8ded68-862d-4abb-a08c-8cf9664dab10": {}
     }
     ```
   - `backup ls --volume <volume-name>`: List all backup names. For example:
     ```json
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
   - `backup inspect-volume <volume>`: Read a single backup volume config (`volume.cfg`). For example:
     ```json
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
   - `backup inspect <backup>`: Read a single backup config (`backup_backup_<backup-hash>.cfg`). For example:
     ```json
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
   - `backup head <config`: Get the config metadata. For example:
     ```json
     {
       "FileTime": "2021-05-17T04:42:03Z",
     }
     ```

   Generally speaking, we want to separate the **list**, **read**, and **head** commands.

2. The Longhorn manager HTTP endpoints.

   | HTTP Endpoint                                                | Before                                                | After                                                                                                                                                |
   | ------------------------------------------------------------ | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
   | **GET** `/v1/backupvolumes`                                  | read all backup volumes from the remote backup target | read all the BackupVolume CRs                                                                                                                        |
   | **GET** `/v1/backupvolumes/{volName}`                        | read a backup volume from the remote backup target    | read a BackupVolume CR with the given volume name                                                                                                    |
   | **DELETE** `/v1/backupvolumes/{volName}`                     | delete a backup volume from the remote backup target  | delete the BackupVolume CR with the given volume name, `backup_volume_controller` reconciles to delete a backup volume from the remote backup target |
   | **POST** `/v1/volumes/{volName}?action=snapshotBackup`       | create a backup to the remote backup target           | create a new Backup, `backup_controller` reconciles to create a backup to the remote backup target                                                   |
   | **GET** `/v1/backupvolumes/{volName}?action=backupList`      | read a list of  backups from the remote backup target | read a list of Backup CRs with the label filter `volume=<backup-volume-name>`                                                                        |
   | **GET** `/v1/backupvolumes/{volName}?action=backupGet`       | read a  backup from the remote backup target          | read a Backup CR with the given backup name                                                                                                          |
   | **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete` | delete a backup from the remote backup target         | delete the Backup CR, `backup_controller` reconciles to delete a backup from reomte backup target                                                    |

## Design

### Implementation Overview

1. Create a new BackupTarget CRD `backuptargets.longhorn.io`.

   ```yaml
   metadata:
     name: the backup target name (`default`), since currently we only support one backup target.
   spec:
     backupTargetURL: the backup target URL. (string)
     credentialSecret: the backup target credential secret. (string)
     pollInterval: the backup target poll interval. (metav1.Duration)
     forceSync: to force sync the backup volume or not. (bool)
   status:
     available: records if the remote backup target is available or not. (bool)
     lastSyncedAt: records the last time the backup target was running the reconcile process. (*metav1.Time)
   ```

2. Create a new BackupVolume CRD `backupvolumes.longhorn.io`.

   ```yaml
   metadata:
     name: the backup volume name.
   spec:
     forceSync: to force sync the backup volume or not. (bool)
   status:
     lastModificationTime: the backup volume config last modification time. (Time)
     size: the backup volume size. (string)
     labels: the backup volume labels. (map[string]string)
     createAt: the backup volume creation time. (string)
     lastBackupName: the latest volume backup name. (string)
     lastBackupAt: the latest volume backup time. (string)
     dataStored: the backup volume block count. (string)
     messages: the error messages when call longhorn engine on list or inspect backup volumes. (map[string]string)
     lastSyncedAt: records the last time the backup volume was synced into the cluster. (*metav1.Time)
   ```

3. Create a new Backup CRD `backups.longhorn.io`.

   ```yaml
   metadata:
     name: the backup name.
     labels:
       longhornvolume=<backup-volume-name>`: this label indicates which backup volume the backup belongs to.
   spec:
     snapshotName: the snapshot name. (string)
     labels: the labels of snapshot backup. (map[string]string)
     backingImage: the backing image. (string)
     backingImageURL: the backing image URL. (string)
     forceSync: to force sync the backup volume or not. (bool)
   status:
     lastModificationTime: the backup config last modification time. (Time)
     backupCreate: to indicate the backup been created or not. (bool)
     url: the snapshot backup URL. (string)
     snapshotName: the snapshot name. (string)
     snapshotCreateAt: the snapshot creation time. (string)
     backupCreateAt: the snapshot backup creation time. (string)
     size: the snapshot size. (string)
     labels: the labels of snapshot backup. (map[string]string)
     messages: the error messages when calling longhorn engine on listing or inspecting backups. (map[string]string)
     lastSyncedAt: records the last time the backup was synced into the cluster. (*metav1.Time)
   ```

4. At the existed `setting_controller`.

   Watches the changes of Setting CR `settings.longorn.io` field `backup-target`, `backup-target-credential-secret`, and `backupstore-poll-interval`. The setting controller is responsible to create/update the default BackupTarget CR.

5. Create a new `backup_target_controller`.

   Watches the change of BackupTarget CR. The backup target controller is responsible for creating/deleting BackupVolume CR metadata+spec. The reconcile loop steps are:
   1. Get the default BackupTarget CR.
   2. Check if the default BackupTarget CR `spec.PollInterval == 0` _or_ `spec.forceSync == false`. If yes, abort the current reconcile process. It no, continue the reconcile process.
   3. Check if `time.Now() - the default BackupTarget CR status.lastSyncedAt >= spec.pollInterval`.
     - If no, skip the current reconcile process.
     - If yes:
       1. Call the longhorn engine to list all the backup volumes `backup ls --volume-only` from the remote backup target `backupStoreBackupVolumes`. If the remote backup target is not available, updates the BackupTarget CR `status.available=false` and `status.lastSyncedAt=time.Now()`, then skip the current reconcile process.
       2. List in cluster BackupVolume CRs `clusterBackupVolumes`.
       3. Find the difference backup volumes `backupVolumesToPull = backupStoreBackupVolumes - clusterBackupVolumes` and create BackupVolume CR `metadata.name` + `spec.forceSync = true`.
       4. Find the difference backup volumes `backupVolumesToDelete = clusterBackupVolumes - backupStoreBackupVolumes` and delete BackupVolume CR.
       5. Updates the BackupTarget CR `spec.forceSync = false` if `spec.forceSync = true`.
       6. Updates the BackupTarget CR `status.available=true` and `status.lastSyncedAt = time.Now()`.

6. For the Longhorn manager HTTP endpoints:

   - **DELETE** `/v1/backupvolumes/{volName}`:
     1. Add the finalizer to the BackupVolume CR with the given volume name.
     2. Delete a BackupVolume CR with the given volume name.

7. Create a new controller `backup_volume_controller`.

   Watches the change of BackupVolume CR. The backup volume controller is responsible for deleting Backup CR and deleting backup volume from remote backup target if delete BackupVolume CR event comes in and updating BackupVolume CR status field, and creating/deleting Backup CR. The reconcile loop steps are:
   1. Get the default BackupTarget CR.
   2. If the delete BackupVolume CR event comes in, delete Backup CR with the given volume name. And delete the backup volume from the remote backup target `backup rm --volume <volume-name> <url>` if the finalizer configured, after that, remove the finalizer.
   3. Check if BackupVolume CR `spec.forceSync == false` and if the default BackupTarget CR `spec.PollInterval == 0` _or_ `time.Now() - the default BackupTarget CR status.lastSyncedAt < spec.pollInterval`, abort the current reconcile process.
   4. Call the longhorn engine to list all the backups `backup ls --volume <volume-name>` from the remote backup target `backupStoreBackups`.
   5.  List in cluster Backup CRs `clusterBackups`.
   6.  Find the difference backups `backupsToPull = backupStoreBackups - clusterBackups` and create Backup CR `metadata.name` + `metadata.labels["longhornvolume"]=<backup-volume-name>`.
   7.  Find the difference backups `backupsToDelete = clusterBackups - backupStoreBackups` and delete Backup CR.
   8.  Call the longhorn engine to get the backup volume config's last modification time `backup head <volume-config>` and compares to `status.lastModificationTime`. If the config last modification time not changed, abort the current reconcile process.
   9. Call the longhorn engine to read the backup volumes' config `backup inspect-volume <volume-name>`.
   10. Updates the BackupVolume CR status field according to the backup volumes' config, and also updates the BackupVolume CR `status.lastModificationTime`, `status.lastSyncedAt`.
   11. Updates the BackupVolume CR spec field `spec.forceSync`.
   12. Updates the Volume CR `status.lastBackup` and `status.lastBackupAt`.

8.  For the Longhorn manager HTTP endpoints:

   - **POST** `/v1/volumes/{volName}?action=snapshotBackup`:
     1. Generate the backup name <backup-name>.
     2. Create a new Backup CR with
        ```yaml
        metadata:
          name: <backup-name>
          labels: 
            longhornvolume: <backup-volume-name>
        spec:
          snapshotName: <snapshot-name>
          labels: <snapshot-backup-labels>
          backingImage: <backing-image>
          backingImageURL: <backing-image-URL>
        ```
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`:
     1. Add the finalizer to the Backup CR with the given backup name.
     2. Delete a Backup CR with the given backup name.

9.  Create a new controller `backup_controller`.

    Watches the change of Backup CR. The backup controller is responsible to update the Backup CR status field and create/delete backup to/from the remote backup target. The reconcile loop steps are:
    1.  Get the default BackupTarget CR.
    2.  If the delete Backup CR event comes in and the finalizer configured, delete the backup volume from the remote backup target `backup rm --volume <volume-name> <url>`.Then, updates BackupVolume CR `spec.forceSync=true` to request backup_volume_controller to immediately reconcile the backup volume. After that, remove the finalizer.
    3.  Check if the Backup CR `spec.snapshotName != ""` and `status.backupCreate == false`. If yes, call longhorn engine/replica for backup creation. Then fork a go routine to monitor the backup creation progress. After that, updates Backup CR `status.backupCreate = true`.
    4.  Check if Backup CR `spec.forceSync == false` and if the default BackupTarget CR `spec.PollInterval == 0` _or_ `time.Now() - the default BackupTarget CR status.lastSyncedAt < spec.pollInterval`, abort the current reconcile process.
    5.  Call the longhorn engine to get the backup config's last modification time `backup head <backup-config>` and compares to `status.lastModificationTime`. If the config last modification time not changed, abort the current reconcile process.
    6.  Call the longhorn engine to read the backup config `backup inspect <backup-url>`.
    7.  Updates the Backup CR status field according to the backup config, and also updates the Backup CR `status.lastModificationTime` and `status.lastSyncedAt`.
    8.  Updates the Backup CR spec field `spec.forceSync`.

10. For the Longhorn manager HTTP endpoints:

   - **GET** `/v1/backupvolumes`: read all the BackupVolume CRs.
   - **GET** `/v1/backupvolumes/{volName}`: read a BackupVolume CR with the given volume name.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a list of Backup CRs with the label filter `volume=<backup-volume-name>`.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a Backup CR with the given backup name.

### Test plan

With over 1k backup volumes and over 1k backups under pretty high network latency (700-800ms per operation)
from longhorn manager to the remote backup target:
- Create a single cluster with 2 volumes (vol-A and vol-B).
   1. The user configures the remote backup target S3 (s3://backupbucket@us-east-1/).
   2. The user creates two backups on vol-A and vol-B.
   3. The user can see the backup volume for vol-A and vol-B in Backup GUI.
   4. The user can see the two backups under vol-A and vol-B in Backup GUI.
   5. When the user deletes one of the backups of vol-A on the Longhorn GUI, the deleted one will be deleted after the remote backup target backup be deleted.
   6. When the user deletes backup volume vol-A on the Longhorn GUI, the backup volume will be deleted after the remote backup target backup volume is deleted, and the backup of vol-A will be deleted also.
   7. The user can see the backup volume for vol-B in Backup GUI.
   8. The user can see two backups under vol-B in Backup GUI.
   9. The user changes the remote backup target to another S3 (s3://backupbucket@us-east-2/), the user can't see the backup volume and backup of vol-B in Backup GUI.
   10. The user configures the `backstore-poll-interval` to 1 minute.
   11. The user changes the remote backup target to the original one S3 (s3://backupbucket@us-east-1/), after 1 minute later, the user can see the backup volume and backup of vol-B.
- Create two clusters (cluster-A and cluster-B) both points to the same remote backup target.
   1. At cluster A, create a volume and run a recurring backup to the remote backup target.
   2. At cluster B, after `backupstore-poll-interval` seconds, the user can list backup volumes or list volume backups on the Longhorn GUI.
   3. At cluster B, create a DR volume from the backup volume.
   4. At cluster B, check the DR volume `status.LastBackup` and `status.LastBackupAt` is updated periodically.
   5. At cluster A, delete the backup volume on the GUI.
   6. At cluster B, after `backupstore-poll-interval` seconds, the deleted backup volume does not exist on the Longhorn GUI.
   7. At cluster B, the DR volume `status.LastBackup` and `status.LastBackupAt` won't be updated anymore.

### Upgrade strategy

None.

## Note

With this enhancement, the user might want to trigger run synchronization immediately. The ways are:
1. Updates the default BackupTarget CR `spec.forceSync = true` to synchronize the backup volumes.
2. For each BackupVolume CR, update the `spec.forceSync = true` to synchronize the backup volumes status and backups.
3. For each Backup CR, update the `spec.forceSync = true` to synchronize the backups status.
