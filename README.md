# Versioner

This repository creates a simple API and related infrastructure for creating an external data versioning system.

Data versioning is a data management best practice where data objects (e.g. rows in a relational database) are
never updated, but new versions of the data are created. Versioning allows for things like audit trails and data
history that are important for compliance, and treating data as immutable can also have benefits for cacheability
of data. However, versioning complicates relational data models substantially, and creating versions of data
records in a primary database creates orders of magnitude more data to store and manage. Moving the data history
to an external system offloads all of that work to a separate data store

## Versioner Data Model

The central data model object in Versioner is the `DataObjectVersion`. 

## Creating new DataObjectVersions

Versioner supports two primary modalities for adding new versions to the repository. The asynchronous model is
appropriate for most use cases, but for applications with extreme senstivity to missing version history, the 
Versioner also supports a synchronous model.

### Asynchronous Version Creation

The initial implementation of Versioner supports asynchronous ingestion of new data object versions via SQS. 
Future implementations may include support for additional input mechanisms. In the asynchronous model, your
application simply needs to create a `DataObjectVersion` instance representing the change, and publish it to 
the SQS queue associated with the Versioner instance.  When properly implemented, this approach has an *extremely low,
but non-zero* probability of losing some data object versions.

We recommend the following best practices to minimize the possibility of missing any data version updates:

* Use EventBridge as an intermediary for publishing to SQS, and [configure a message archive](
    https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-archive.html)
* Add an error handler around your publish code to log the full `DataObjectVersion` to disk if your request to 
  EventBridge fails (this allows you to resubmit data versions after EventBridge recovers from an outage, for example)

Outages in EventBridge or SQS may *delay* the propagation of versions to Versioner -- in extreme cases, requiring
operational action to replay from the EventBridge archive or from your application logs -- but true data loss would
be limited to cases where both EventBridge fails to archive the message **and** SQS fails to deliver the message to
Versioner, or if your application exits after writing to the database but before publishing to EventBridge.

### Synchronous Version Creation

As described above, we believe the asynchronous version creation model has an extremely low probability of data 
loss, and recommend it for most use cases, but Versioner also supports synchronous version creation via both
HTTP `POST` (version ID generation handled by Versioner) or `PUT` (version ID generation handled in your application).
Regardless of which of these methods you approach, carefully consider the following tradeoffs:

* If you create the `DataObjectVersion` after writing your primary object to the database, you are still subject
  to the possibility of missing a version record in the case where your application crashes (or otherwise errors)
  between writing to the database and creating the version
* If you create the `DataObjectVersion` record *before* writing your primary object to your database, the availability
  (and latency) of your application takes a hard dependency on Versioner, which is just an open source library some
  random dude threw together in a weekend or two