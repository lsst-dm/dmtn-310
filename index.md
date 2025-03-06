# Reducing Butler database contention in Prompt Processing

```{abstract}
During Operations Rehearsal 5, the scalability of Prompt Processing was limited by the Butler Postgres database.  We propose creating a service to batch the outputs from the individual Prompt Processing pods, reducing contention for the Butler database.  Instead of writing directly to the Butler database, pods would send an event to the service using Kafka.
```

## Add content here

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.
