# Title

Asynchronous pull backup volumes and historical volume backups into cluster custom resource (CR).

## Summary

Asynchronous way to pull backup volumes and historical volume backups from the external backup store (S3/NFS) into cluster custom resource (CR).
This enhancement is to fix the timeout issue when listing backup volumes from the external backup store _or_ listing historical volume backups 
from the external backup store.

### Related Issues

- https://github.com/longhorn/longhorn/issues/1761
- https://github.com/longhorn/longhorn/issues/1955
- https://github.com/longhorn/longhorn/issues/2536
- https://github.com/longhorn/longhorn/issues/2543

## Motivation

### Goals

Decrease the time when the user clicks list backup _or_ list historical volume backups under the circumstances that lots of volume backups _or_ 
lots of historical volume backups in the external backup store **and** the latency from Longhorn cluster to the external backup store is high.

### Non-goals [optional]

Automatically adjust the external backup store pull period.

## Proposal

Currently, we have a setting `backupstore-poll-interval` to periodically pull the external backup store lastest volume backups within a controller.
We want to leverage it to pull the external backup volumes and historical volume backups from the external backup store and saves them into the cluster 
custom resource (CR). Therefore, we'll create a new Custom Resource Definition (CRD) called _backups_ to save the backup volumes and historical volume backups.

With this enhancement, we want to:
1. Pull the backup volumess and historical volume backups that **are in** the external backup store and **aren't in** the cluster CR only.
2. Change the [longhorn/backupstore](https://github.com/longhorn/backupstore) list command behavior. Currently, `backup list` command includes
   listing all backup volumes and the historical volume backups and read these metadata.
   We'd like to change the `backup list` behavior to perform list only, but not read the metadata.

### User Stories

Before this enhancement, when the user's environment under the circumstances that the external backup store has lots of backup volumes _or_ 
historical volume backups, and the latency between the longhorn manager to the external backup store is high.
Then if the user clicks the `Backup` on the GUI, the user might hit list backup volumes _or_ list historical volume backups timeout issue 
(the default timeout is 1 min).

We choose to not create a new setting for the user to increase the list timeout value is because the browser has its timeout value also.
Let's say the list backups needs 5 minutes to finish. Even we allow the user to increase the longhorn manager list timeout value, we can't change 
the browser default timeout value. Furthermore, some browser doesn't allow the user to change default timeout value like Google Chrome.

After this enhancement, when the user's environment under the circumstances that the external backup store has lots of backup volumes _or_ 
historical volume backups, and the latency between the longhorn manager to the external backup store is high.
Then if the user clicks the `Backup` on the GUI, the user can list backup volumes _or_ list historical volume backups without timeout issue.

#### Story 1

The user environment is under the circumstances that the external backup store has lots of backup volumes and the latency between the longhorn manager 
to the external backup store is high. Then, the user can list all backup volumes on the GUI.

#### Story 2

The user environment is under the circumstances that the external backup store has lots of historical volume backups and the latency between the longhorn 
manager to the external backup store is high. Then, the user can list all historical volume backups on the GUI.

### User Experience In Detail

None.

### API changes

The current [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior are:
- `backup ls --volume-only`: List all backup volumes and read it's metadata (`volume.cfg`).
- `backup ls --volume <volume-name>`: List all historical volume backups and read it's metadata (`backup_backup_<backup-hash>.cfg`).
- `backup ls --volume <volume-name> --volume-only`: Read a single backup volume metadata (`volume.cfg`).
- `backup inspect <backup>`: Read a single volume's backup metadata (`backup_backup_<backup-hash>.cfg`).

After this enhancement, the [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior are:
- `backup ls --volume-only`: List all backup volume names.
- `backup ls --volume <volume-name>`: List all historical volume backup names.
- `backup ls --volume <volume-name> --volume-only`: Read a single backup volume name.
- `backup inspect <metadata-path>`: Read a single backup volume metadata (`volume.cfg`) or read a single volume's backup metadata (`backup_backup_<backup-hash>.cfg`).

Generally speaking, we want to separate the **list** and **read** command.

## Design

### Implementation Overview

We'll create a new Custom Resource Definition (CRD) called `backupvolumes.longhorn.io` to save pulled backup volumes metadata 
and its historical volume backups metadata as the Custom Resource (CR). For each backup volume, we'll create a new CR.
- `metadata.name`: the backup volume name.
- `spec.name`: the name field inside the backup volume metadata.
- `spec.size`: the size field inside the backup volume metadata.
- `spec.labels`: the labels field inside the backup volume metadata.
- `spec.created`: the created field inside the backup volume metadata.
- `spec.lastBackupName`: the lastBackupName field inside the backup volume metadata.
- `spec.lastBackupAt`: the lastBackupAt field inside the backup volume metadata.
- `spec.dataStored`: the dataStored field inside the backup volume metadata.
- `spec.messages`: the messages field inside the backup volume metadata.
- `spec.backups`: a map of volume backups (key is volume backup name, value is the volume backup metadata).
  - `spec.backups[volumeBackupName].name`: the name field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].url`: the url field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].snapshotName`: the snapshotName field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].snapshotCreated`: the snapshotCreated field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].created`: the created field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].size`: the size field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].labels`: the labels field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].volumeName`: the volumeName field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].volumeSize`: the volumeSize field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].volumeCreated`: the volumeCreated field inside a historical volume backup metadata.
  - `spec.backups[volumeBackupName].messages`: the messages field inside a historical volume backup metadata.
- `status.lastSyncedTime`: records the last time the backup store contents were synced into the cluster.

There is already a backup store monitor that runs as a timer (inside setting controller), the timer period is the setting `backupstore-poll-interval`.
Within it, it runs:
1. List backup volumes in the cluster CR `backupvolumes.longhorn.io`.
2. List backup volumes from the external backup store.
3. Find the different backup volumes `backupVolumesToPull` that are in the external backup store and aren't in the cluster CR `backupvolumes.longhorn.io`.
4. Find the different backup volumes `backupVolumesToDelete` that are in the cluster CR `backupvolumes.longhorn.io` and aren't in the external backup store.
5. Loop the `backupVolumesToPull`, read the backup volume metadata from the external backup store, and create a new CR in `backupvolumes.longhorn.io`.
6. Loop the `backupVolumesToDelete`, delete the CR in `backupvolumes.longhorn.io`.
7. Loop all backup volumes in the cluster CR `backupvolumes.longhorn.io`, for each backup volume (BV):
   1. Get the backup volume in the cluster CR `backupvolumes.longhorn.io`.
   2. List historical volume backups in the backup volume CR `backupvolumes.longhorn.io` field `spec.backups`.
   3. List historical volume backups under the backup volume (BV)from the external backup store.
   4. Find the different volume backups `volumeBackupsToPull` that are in the external backup store and aren't in the backup volume CR field `spec.backups`.
   5. Find the different volume backups `volumeBackupsToDelete` that are in the backup volume CR `spec.backups` and aren't in the external backup store.
   6. Loop the `volumeBackupsToPull`, read the historical volume backup metadata from the external backup store, and add to the backup volume CR field `spec.backups`.
   7. Loop the `volumeBackupsToDelete`, delete the backup volume CR `spec.backups`.
   8. Update the backup volume (BV) CR `spec.backups`.

For the longhorn manager endpoints:
- **GET** `/v1/backupvolumes`: read all backup volumes from the cluster custom resource (CR).
- **GET** `/v1/backupvolumes/{volName}`: read a single backup volume from the cluster custom resource (CR).
- **DELETE** `/v1/backupvolumes/{volName}`: delete a single backup volume from the cluster custom resource (CR) and also from external backup store.
- **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a single volume backups from the cluster custom resource (CR).
- **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a single volume backup from the cluster custom resource (CR).
- **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete a single volume backup from the cluster custom resource (CR) and also from external backup store.

### Test plan

With over 1k backup volumes and over 1k historical volume backups under pretty high network latency (700-800ms per operation)
from longhorn manager to external backup store:
1. The user can list backup volumes.
2. The user can list historical volume backups.
3. When the user deletes a backup volume on the Longhorn GUI:
   1. list backup volumes, the deleted one does not exist immediately
   2. check the external backup store, the backup volume will be deleted after a while.
4. When the user deletes a historical volume backup on the Longhorn GUI:
   1. list historical volume backups, the deleted one does not exist immediately
   2. check the external backup store, the historical backup will be deleted after a while.
5. When the user deletes a backup volume on the external backup store manually.
   After the `backupstore-poll-interval`, list backup volumes, the deleted one does not exist.
6. When the user deletes a historical volume backup on the external backup store manually.
   After the `backupstore-poll-interval`, list historical volume backups, the deleted one does not exist immediately.

### Upgrade strategy

Before this enhancement, the user might set `backupstore-poll-interval` to `0` to reduce the request to the external backup store.
This is majorly for the user who wants to reduce the cost on access AWS S3.

After this enhancement, if the user ever configured the setting `backupstore-poll-interval` to `0`, then the user is unable to list backup volumes
_or_ historical volume backups. We should add this note to the documentation and the release note.

## Note

With this enhancement, the CR _backups.longhorn.io_ is updated when the asynchronous timer be triggered.
However, the user might want to trigger it immediately. But currently, we haven't found a good way to let the user force trigger it now.
