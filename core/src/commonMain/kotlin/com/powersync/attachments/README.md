# PowerSync Attachment Helpers

A [PowerSync](https://powersync.com) library to manage attachments (such as images or files) in Kotlin Multiplatform apps.

This package is included in the PowerSync Core module.

### Alpha Release

Attachment helpers are currently in an alpha state, intended strictly for testing. Expect breaking changes and instability as development continues.

Do not rely on this package for production use.

## Usage

An `AttachmentQueue` is used to manage and sync attachments in your app. The attachments' state is
stored in a local-only attachments table.

### Key Assumptions

- Each attachment is identified by a unique ID
- Attachments are immutable once created
- Relational data should reference attachments using a foreign key column
- Relational data should reflect the holistic state of attachments at any given time. An existing local attachment will deleted locally if no relational data references it.

### Example Implementation

See the [Android Supabase Demo](/demos/android-supabase-todolist/README.md) for a basic example of attachment syncing.

In the example below, the user captures photos when checklist items are completed as part of an
inspection workflow.

1. First, define your schema including the `checklist` table and the local-only attachments table:

```kotlin
val checklists = Table(
    name = "checklists",
    columns =
        listOf(
            Column.text("description"),
            Column.integer("completed"),
            Column.text("photo_id"),
        ),
)

val schema = Schema(
    UserRow.table,
    // Add the local-only table which stores attachment states
    // Learn more about this function below
    createAttachmentsTable("attachments")
)
```

2. Create an `AttachmentQueue` instance. This class provides
default syncing utilities and implements a default sync strategy. This class is open and can be overridden for custom functionality:

```kotlin
val queue = AttachmentQueue(
    db = db,
    attachmentsDirectory = attachmentsDirectory,
    remoteStorage = SupabaseRemoteStorage(supabase),
    watchAttachments = { 
        db.watch(
            sql = """
                SELECT photo_id
                FROM checklists
                WHERE photo_id IS NOT NULL
            """,
        ) {
            WatchedAttachmentItem(id = it.getString("photo_id"), fileExtension = "jpg")
        }
    }
)
```

* The `attachmentsDirectory`, specifies where local attachment files should be stored. This directory needs to be provided to the constructor. On Android
  `"${applicationContext.filesDir.canonicalPath}/attachments"` is a good choice.
* The `remoteStorage` is responsible for connecting to the attachments backend. See the `RemoteStorageAdapter` interface
  definition [here](https://github.com/powersync-ja/powersync-kotlin/blob/main/core/src/commonMain/kotlin/com.powersync/attachments/RemoteStorageAdapter.ts).
* `watchAttachments` is a `Flow` of `WatchedAttachmentItem`. The `WatchedAttachmentItem`s represent the attachments which should be present in the
  application. We recommend using `PowerSync`'s `watch` query as shown above. In this example we provide the `fileExtension` for all photos. This information could also be
  obtained from the query if necessary.

3. Implement a `RemoteStorageAdapter` which interfaces with a remote storage provider. This will be
   used for downloading, uploading and deleting attachments:

```kotlin
val remote = object : RemoteStorage() {
    override suspend fun uploadFile(fileData: Flow<ByteArray>, attachment: Attachment) {
        TODO("Implement upload to your backend")
    }

    override suspend fun downloadFile(attachment: Attachment): Flow<ByteArray> {
        TODO("Implement download from your backend")
    }

    override suspend fun deleteFile(attachment: Attachment) {
        TODO("Implement delete in your backend")
    }
}
```

4. Start the sync process:

```kotlin
queue.startSync()
```

5. Create and save attachments using `saveFile()`. This method will
   save the file to the local storage, create an attachment record which queues the file for upload
   to the remote storage and allows assigning the newly created attachment ID to a checklist item:

```kotlin
queue.saveFile(
    data = flowOf(ByteArray(1)), // Your attachment data
    mediaType = "image/jpg",
    fileExtension = "jpg",
) { tx, attachment ->
    /**
     * This lambda is invoked in the same transaction which creates the attachment record.
     * Assignments of the newly created photo_id should be done in the same transaction for maximum efficiency.
     */
    tx.execute(
        """
            UPDATE checklists 
            SET photo_id = ?
            WHERE id = ?
        """,
        listOf(attachment.id, checklistId),
    )
}
```

## Implementation Details

### Attachment Table Structure

The `createAttachmentsTable` function creates a local-only table for tracking attachment states. 

An attachments table definition can be created with the following options:

| Option | Description           | Default       |
|--------|-----------------------|---------------|
| `name` | The name of the table | `attachments` |

The default columns are:

| Column Name  | Type      | Description                                                                                                        |
|--------------|-----------|--------------------------------------------------------------------------------------------------------------------|
| `id`         | `TEXT`    | Unique identifier for the attachment                                                                               |
| `filename`   | `TEXT`    | The filename of the attachment                                                                                        |
| `media_type` | `TEXT`    | The media type of the attachment                                                                                        |
| `state`      | `INTEGER` | Current state of the attachment (see `AttachmentState` enum)                                                         |
| `timestamp`  | `INTEGER` | The timestamp of last update to the attachment                                                                                              |
| `size`       | `INTEGER` | File size in bytes                                                                                                 |
| `has_synced` | `INTEGER` | Internal flag tracking if the attachment has ever been synced (used for caching)                                    |
| `meta_data`  | `TEXT`    | Additional metadata in JSON format                                                                                 |

### Attachment States

Attachments are managed through the following states:

| State             | Description                                                                   |
|-------------------|-------------------------------------------------------------------------------|
| `QUEUED_UPLOAD`   | Attachment is queued for upload to cloud storage                              |
| `QUEUED_DELETE`   | Attachment is queued for deletion from cloud storage and local storage        |
| `QUEUED_DOWNLOAD` | Attachment is queued for download from cloud storage                          |
| `SYNCED`          | Attachment is fully synced                                              |
| `ARCHIVED`        | Attachment is orphaned - i.e. no longer referenced by any data                                |

### Sync Process

The `AttachmentQueue` implements a sync process with these components:

1. **State Monitoring**: The queue watches the attachments table for records in `QUEUED_UPLOAD`, `QUEUED_DELETE`, and `QUEUED_DOWNLOAD` states. An event loop triggers calls to the remote storage for these operations.

2. **Periodic Sync**: By default, the queue triggers a sync every 30 seconds to retry failed uploads/downloads, in particular after the app was offline. This interval can be configured by setting `syncInterval` in the `AttachmentQueue` constructor options, or disabled by setting the interval to `0`.

3. **Watching State**: The `watchAttachments` flow generator in the `AttachmentQueue` constructor is used to maintain consistency between local and remote states:
   - New items trigger downloads - see the Download Process below.
   - Missing items trigger archiving - see Cache Management below.

### Upload Process

The `saveFile` method handles attachment creation and upload:

1. The attachment is saved to local storage 
2. An `AttachmentRecord` is created with `QUEUED_UPLOAD` state, linked to the local file using  `localURI` 
3. The attachment must be assigned to relational data in the same transaction, since this data is constantly watched and should always represent the attachment queue state
4. The `RemoteStorage` `uploadFile` function is called
5. On successful upload, the state changes to `SYNCED`
6. If upload fails, the record stays in `QUEUED_UPLOAD` state for retry

### Download Process

Attachments are scheduled for download when the flow from `watchAttachments` emits a new item that is not present locally:

1. An `AttachmentRecord` is created with `QUEUED_DOWNLOAD` state
2. The `RemoteStorage` `downloadFile` function is called
3. The received data is saved to local storage
4. On successful download, the state changes to `SYNCED`
5. If download fails, the operation is retried in the next sync cycle

### Delete Process

The `deleteFile` method deletes attachments from both local and remote storage:

1. The attachment record moves to `QUEUED_DELETE` state
2. The attachment must be unassigned from relational data in the same transaction, since this data is constantly watched and should always represent the attachment queue state
3. On successful deletion, the record is removed
4. If deletion fails, the operation is retried in the next sync cycle

### Cache Management

The `AttachmentQueue` implements a caching system for archived attachments:

1. Local attachments are marked as `ARCHIVED` if the flow from `watchAttachments` no longer references them
2. Archived attachments are kept in the cache for potential future restoration
3. The cache size is controlled by the `archivedCacheLimit` parameter in the `AttachmentQueue` constructor
4. By default, the queue keeps the last 100 archived attachment records
5. When the cache limit is reached, the oldest archived attachments are permanently deleted
6. If an archived attachment is referenced again while still in the cache, it can be restored
7. The cache limit can be configured in the `AttachmentQueue` constructor

### Error Handling

1. **Automatic Retries**:
   - Failed uploads/downloads/deletes are automatically retried
   - The sync interval (default 30 seconds) ensures periodic retry attempts
   - Retries continue indefinitely until successful

2. **Custom Error Handling**:
   - A `SyncErrorHandler` can be implemented to customize retry behavior (see example below)
   - The handler can decide whether to retry or archive failed operations
   - Different handlers can be provided for upload, download, and delete operations


Example of a custom `SyncErrorHandler`:

```kotlin
val errorHandler = object : SyncErrorHandler {
    override suspend fun onDownloadError(
        attachment: Attachment,
        exception: Exception
    ): Boolean {
        TODO("Return if the attachment sync should be retried")
    }

    override suspend fun onUploadError(
        attachment: Attachment,
        exception: Exception
    ): Boolean {
        TODO("Return if the attachment sync should be retried")
    }

    override suspend fun onDeleteError(
        attachment: Attachment,
        exception: Exception
    ): Boolean {
        TODO("Return if the attachment sync should be retried")
    }
}

val queue = AttachmentQueue(
    // ... other parameters ...
    errorHandler = errorHandler
)
```