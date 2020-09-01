@tags datastore

# Datastore

The datastore has a simple intent:

- you store data and get a URI to uniquely describe that bit of data
- later you can use that URI to retrieve your data

[^.resources/datastore-run.mp4]

## Where is the data?

The goal of datastore is that someone using it doesn't necessarily need to care where the data ends up. This means you don't need to keep a per-application configuration for data storage.
Instead, you can use datastore routes to indicate per context where data should end up. This can be different per environment as well.

By default datastore uses the Nabu VFS which already supports a number of protocols out of the box. You can however create your own datastore providers which might do entirely different things with the data.

If for example we want to configure all data for our "example" application to be stored in the tmp folder, we can configure this:

[^.resources/datastore-configure-route.mp4]

## URN


