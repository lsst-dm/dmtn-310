# Reducing Butler database contention in Prompt Processing

```{abstract}
During Operations Rehearsal 5, the scalability of Prompt Processing was limited by the Butler Postgres database.  We propose creating a service to batch the outputs from the individual Prompt Processing pods, reducing contention for the Butler database.  Instead of writing directly to the Butler database, pods would send an event to the service using Kafka.
```

## Butler writes

At the end of execution, each Prompt Processing pod does the following writes
to the central Butler database:

1. Inserts dimension records (metadata) associated with the processing that occurred
2. Inserts dataset records for the data products created by Prompt Processing
3. Updates a chained collection to include the collection containing the output datasets

### Inserting dimension records
Transferring and inserting dimension records is straightforward.  Prompt
Processing already generates a list of the records, and Butler already provides
a JSON serialization for these records that can be used to transfer them to the
writer service.

### Inserting dataset records
Currently, the insertion of dataset records in Prompt Processing uses the
method `Butler.transfer_from()`. `transfer_from` is a compound operation that
both copies the artifact files and does updates the database.

In the new scheme, the Prompt Processing pod is still required to copy the
artifact files to the central S3 repository, but it does not write the
associated dataset records to the Butler Postgres database.

Instead of calling `transfer_from()`, we can use `Datastore.export()`.
`Datastore.export()` generates a `FileDataset` object containing all the
information necessary to insert a dataset record into the database.
`FileDataset` does not currently have a convenient JSON serialization, but one
could be added easily.  The writer service can call `Butler.ingest()` to insert
the dataset records into the database.

In addition, `Datastore.export()` copies the files to a specified location.
Prompt Processing pods could write the files to a staging area separate from
the central Butler datastore directory, and the writer service could move them
to their final location.  Alternatively, the pods could write the files
directly to their final destination in the datastore, but this increases the
chances of consistency problems between the database and the files on disk.

### Chained collection update
The name of the collection can be transferred in JSON as a plain string, and
the writer service can use the same function that the pods are currently
using to update the database.

## Fringe Benefits

As a side-effect, Kafka orders the events in a way that makes it easy to create
a log of the data products that were generated.  This may make it
easier to incrementally populate downstream databases, when data is moved out
of the embargo rack and made available to end-users.

## Disadvantages to this approach
Adding an external service will complicate build and deployment.  It also makes
it harder for unit tests to fully cover the process of writing the outputs,
since it is split between services and adds additional infrastructure
requirements.

## The other half of the problem
Removing the writes to the database for the outputs will not fully remove the
need for Prompt Processing pods to communicate with the central Butler
database.  The database is also queried for inputs needed by the pipelines
executed on the pods.

However, with writes eliminated, we no longer need to connect to the primary
database.  We can set up a dedicated read replica for Prompt Processing,
insulating time-critical processing from other users/services that may
consume resources unpredictably on the primary database.

Testing in Operations Rehearsal 5 showed that with only the input queries, the
database access from the pods was able to scale acceptably.  We may not need
to do anything to address the input queries until we start seeing issues from
growth in the database or increased complexity of queries.