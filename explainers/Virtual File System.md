@explain vfs

# Virtual File System

We are all familiar with a regular filesystem: it is a set of files spread out over a hierarchic structure of directories. The virtual file system that comes with the Nabu Platform allows you to access any system that has the same file/directory setup or has a specific implementation that mimics this.

Some examples:

- :sftp: this [protocol](https://en.wikipedia.org/wiki/SSH_File_Transfer_Protocol) allows you to securily move files between systems. It is often the standard file delivery protocol in large enterprises.
- :smb: this [protocol](https://en.wikipedia.org/wiki/Server_Message_Block) is the standard used for file sharing in windows systems
- :zip: a read-only version that exposes a zip file as if it were a directory
- :memory: builds a file system in memory which is very fast but not persisted
- :http: there is an implementation that exposes a file system over http
- :file: obviously there is also an implementation of the virtual file system that exposes the actual file system

## Mix and match

You can create new virtual file systems that combine any of the existing providers into a hybrid structure. This means whoever is using the virtual file system sees one file system with files and directories as per usual. However, depending on where he reads and writes files, he might be accessing them over sftp on a remote server or reading straight from a zip file.

## Alias

You can create aliases that point to other parts of a virtual file system. You can then redefine these aliases per environment making it easy to use a seemingly fixed endpoint but actually differentiate it per environment.

One of the aliases we often use is ``data:``. You will often see a datastore route that simply points to ``data:/datastore`` or something like that. In development it might point to the local filesystem but in production it might point to a shared drive.

## Snapshot

The virtual file system comes with some advanced features that Nabu uses in some contexts. One of them is snapshots. This feature is best explained with an example.

When Nabu performs a deployment to a target repository, it actually has to write new files to the virtual file system that the target repository uses. But we want to limit downtime or file corruption.

Before we start deployment, we take a snapshot in memory of the vfs. We can then write do changes to the backing file systems without impacting the runtime.

[^.resources/vfs-snapshot.mp4]

Once the new files are written to the file systems, we have two options:

- :release snapshot: we drop the memory snapshot and relink the repository to the newly updated filesystem, we then reload the changed folders
- :restore snapshot: if something went wrong or we simply want to roll back, we can restore the snapshot to the file system and effectively undo the deployment
