@tags datastore

# Datastore

The datastore has a simple intent:

- you store data and get a URI to uniquely describe that bit of data
- later you can use that URI to retrieve your data

[^.resources/datastore-run.mp4]

## Where is the data?

The goal of datastore is that someone using it doesn't necessarily need to care where the data ends up. You can configure this using datastore routes where you indicate per context where data should end up. This can be different per environment as well.

A datastore route can configure a provider that should handle the data. The default provider uses a virtual file system that already supports a number of protocols out of the box. Apart from that you can easily implement a provider that might save your data in a database for example.

