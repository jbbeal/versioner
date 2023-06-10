# Versioner

This repository creates a simple API and related infrastructure for creating an external data versioning system.

Data versioning is a data management best practice where data objects (e.g. rows in a relational database) are never updated, but new
versions of the data are created. Versioning allows for things like audit trails and data history that are important for compliance, and
treating data as immutable can also have benefits for cacheability of data. However, versioning complicates relational data models
substantially, and creating versions of data records in a primary database creates orders of magnitude more data to store and manage. Moving
the data history to an external system offloads all of that work to a separate data store

## Versioner Data Model

The central data model object in Versioner is the `DataObjectVersion`. The `DataObjectVersion` includes the following fields.  (In some
field names, we use the prefix `do` -- for `DataObject` -- to help disambiguate concepts.)

### Identifiers

* `doDomain`: A string representing a grouping of related data types. If your primary database is relational, you may want to use the
  database schema or tablespace name as the domain.
* `dataType`: String; name of the type of data object. This should correspond to a table or collection name in your primary database.
* `doId`: This is the primary identifier for the `DataObject`
* `version`: Numeric value representing a specific version of an object.

The above four fields uniquely represent a `DataObjectVersion`. For convenience, Versioner APIs also return a field that concatentates all
four values:

* `dovId`: String that independently identifies a full `DataObjectVersion`. This field is allowed, but ignored, as part of the body of a
  `DataObjectVersion` in any request to create a new version. (Specifically, if your application generates a `dovId` when creating a
  `DataObjectVersion` your `dovId` does not match the `dovId` that Versioner's internal logic would create, Versioner will ignore the
  provided `dovId`.)
  
### Timestamps

* `vsRecordedTs`: The time at which Versioner records the version in its database. If the application provides a version for this timestamp
  when creating a version, Versioner will ignore it.
* `vsReceivedTs`: The time at which Versioner receives notification of the version. When called asynchronously, Versioner will use the
  `ApproximateFirstReceiveTimestamp` from SQS; when called synchronously, this time will be the same as `vsRecordedTs`.
* `appRecordedTs`: The time at which the application recorded the version in its database. If not provided, Versioner will use the `vsRecordedTs`.
  Note that, if provided by the application, Versioner will not validate that the provided value is in ISO-8601 format; any string will be
  recorded (but truncated to 30 characters)


### Audit field

The `audit` field is used to store additional metadata to help associate a `DataObjectVersion` with the originating user or system behavior
generating the change. Applications are free to use this field as they choose, but we suggest the following fields that cover most common
use cases.

* `user_id`: Identifier or email address for a human user who initiated a change.
* `user_ip`: The IP address where user activity originated
* `service`: A name or other identifier for the service or system responsible for saving the data into the primary database
* `service_ip`: The IP address of the specific server that writes data to the database
* `request_id`: A request ID for tracing the request in application logs

Versioner records the following fields as a `versioner` object within the `audit` field:

* `channel`: either `https` or `sqs`
* `request_id`: For `https` only
* `message_id`: For `sqs` only
* `sender_id`: For `sqs` only

### Data Object Value

Finally, the `DataObjectVersion` needs a copy of the data object at the point in time represented by this record.

* `doValue`: The value of the data object itself at the time this version is created.

Versioner does performa any validation on this field, other than enforcing a maximum length (which is configurable when Versioner is
deployed). Applications may choose to copy the entire value of the `DataObject` (generally recommended) or store only changed fields if data
storage costs are a concern.  (The entire `DataObjectVersion` must be JSON-serializable. The `doValue` can either be a JSON object, or a 
stringified version of an object. If stringified, it could even be XML escaped in a JSON string if applications choose.)

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
