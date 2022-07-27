﻿# Lokad.AzureEventStore

**Event sourcing backed by Azure Blob Storage, in a simple, low-maintenance 
.NET client library.** You can use Lokad.AzureEventStore for your prototypes and 
small-scale projects, then move on to other solutions (such as 
[Apache Kafka](https://kafka.apache.org/) or [Greg Young's Event Store](https://geteventstore.com))
once you confirm the need for additional performance or features. 

If you're curious about event sourcing, you can read [this introduction from Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing),
or [the documentation for Event Store](http://docs.geteventstore.com/introduction/4.0.2/event-sourcing-basics/).

## When should I use this ?

Lokad.AzureEventStore will fit your project comfortably if: 

 - Your events are serialized with [Json.NET](https://www.newtonsoft.com/json), and weigh less than 512KB each.
 - Each aggregate receives less than 10 events per second.
 - The materialized views for all your aggregates fit in memory.

NuGet package: [Lokad.AzureEventStore](https://www.nuget.org/packages/Lokad.AzureEventStore/).

What you get out of Lokad.AzureEventStore:

 - **Robust data storage**: your events are written to [Azure Append Blobs](https://docs.microsoft.com/en-us/rest/api/storageservices/understanding-block-blobs--append-blobs--and-page-blobs#about-append-blobs),
   which are append-only. This makes it impossible for your application to 
   overwrite, delete or alter the existing stream of events in any way. 

   The serialization format of the event stream includes checksums to detect
   data corruption (those are checked on every read), and is simple enough 
   that creating a backup of a stream can be done by copying the blobs
   byte-for-byte. 

   This library has been used as the primary persistence model at Lokad
   since 2014, and we have yet to encounter a case of data loss.

 - **No need for a server**: you do not need to install a database server, 
   keep it up-to-date with security fixes, monitor its performance, 
   or plan a fail-over solution. 

 - **Thread-safety within the application**: multiple threads can append
   events to the stream, or read the current state of a projection. The 
   library also reads events from the remote stream and updates the 
   projection state in the background.

   All operations which involve remote access to the Append Blobs use 
   are asynchronous (using `System.Threading.Tasks`).

 - **Consistency within each aggregate**: when deciding to appending a 
   new event to a stream, the application has access to _all_ events 
   present in the stream before that new event. 

   For example, if two applications want to append two mutually 
   exclusive events at the same time, one of the two applications is 
   guaranteed to see the event generated by the other application 
   before it attempts to write its own. This is handled transparently 
   by the library.

 - **Projection state snapshots**: as long as your application provides
   serialization methods for your projection state, the library will 
   automatically load a previous snapshot of the state whenever available,
   and only read back the events that were appended since this snapshot
   was taken.

   You are in control of when (and how often) those snapshots are created.

 - **Easy migration path**: the event stream is just a stream of events, 
   serialized as JSON. It can be easily moved to another event sourcing 
   solution. 
   
   Projection code and event contracts are not dependent on the library 
   and can be reused as-is. 

## How do I use this ? 

The [example project](./ExampleProject) can serve as a quick introduction
to the concepts behind Lokad.AzureEventStore. The project is a distributed
word counter: each instance is a console application, and all instances 
share a dictionary of words with their corresponding counts.

If text is typed on the command line, the application splits the text into
words and updates the shared count dictionary.

If the line starts with `--` then the words on the line are instead removed
from the dictionary.

### Event Contracts

An event represents "something that has happened", as part of an append-only
stream of events representing the full history of the application. In C#, 
these are represented as JSON-serializable plain old objects implementing a
single interface local to the project (such as `IEvent` in the example project).

We use two event types for this example: `ValueUpdated` happens whenever a
word is counted, it contains the word (`Key`) and the new tally for that word 
(`Value`) ; `ValueDeleted` happens when the word is removed from the dictionary,
and only references the word (`Key`).

```csharp
[DataContract]
public sealed class ValueUpdated : IEvent
{
    [DataMember]
    public string Key { get; private set; }

    [DataMember]
    public int Value { get; private set; }
}

[DataContract]
public sealed class ValueDeleted : IEvent
{
    [DataMember]
    public string Key { get; private set; }
}
```

### State

As expected of an event sourcing solution, the primary way of accessing the state
application is not to read back through the event stream (which would become very 
time-consuming as the stream grows) but instead to create one or more materialized
views of the state. Keeping these views up-to-date can be done efficiently by applying
new events as they appear.

Lokad.AzureEventStore imposes the additional constraint that state should be
represented by immutable data structures. This guarantees that multiple threads can
read the state while a background thread applies new events, without any locks being
required.

In the example project, the state is simply a dictionary that maps words to their
current tally.

```csharp
public sealed class State
{
    public State(ImmutableDictionary<string, int> bindings)
    {
        Bindings = bindings;
    }

    public ImmutableDictionary<string,int> Bindings { get; }
}
```

### Projection

In Lokad.AzureEventStore, a projection is the code that turns the stream of
events into a state object representing the materialized view. A projection
that creates a `State` from a stream of `IEvent` should implement the interface
`IProjection<IEvent, State>`, which itself implements the interface 
`IProjection<IEvent>` of projections that work with events of type `IEvent`
(since you may have several of these projections, with different state types, 
in a single application).

The following fields must be implemented: 

 - The initial state, to which the first event in the stream will be 
   applied.

   ```csharp
   public State Initial => 
       new State(ImmutableDictionary<string, int>.Empty);
   ```

 - The type of the state. This is used for reflection purposes, to identify
   which projection should be invoked for a given materialized view.

   ```csharp
   public Type State => typeof(State);
   ```

 - The `Apply` function which, based on a previous state and a new event,
   returns a new state. The `sequence` number uniquely identifies the event,
   being incremented by 1 for each event added to the stream (the first event
   has a sequence number of 1).

   Note that the projection should never rely on events being provided in any
   specific order. The library will routinely call the projection with the 
   same event more than once, or with events that do not even appear in the 
   event stream, or skip certain events entirely.
   
   It is however guaranteed that the `State` passed to function `Apply` on a 
   given sequence number will be the result of calling `Apply` on the previous
   sequence number (that is, the `State` is build by applying events in 
   chronological order, starting with `Initial`).
   
   In a real-life project, this function would be implemented with a visitor
   pattern, but in the example project it is simply:

   ```csharp
   public State Apply(uint sequence, IEvent e, State previous)
   {
       if (e is ValueDeleted d)
           return new State(previous.Bindings.Remove(d.Key));
     
       if (e is ValueUpdated u)
           return new State(previous.Bindings.SetItem(u.Key, u.Value));

       throw new ArgumentOutOfRangeException(nameof(e), "Unknown event type " + e.GetType());
   }
   ```

 - The `TryLoadAsync` and `TrySaveAsync`, used to add checkpoint support to a
   projection. `TrySaveAsync` should return `false` if checkpoints are not 
   supported, `TryLoadAsync` will only be called if a checkpoint was previously 
   generated (but should throw if the checkpoint cannot be parsed).

 - The `FullName` is used for logging, but also for checkpoint versioning if
   checkpoints are supported. Permitted characters are letters, numbers and the
   character `-`. The canonical name for version 18 of projection Foo is `Foo-18`.

   You should increase the version number whenever changes were applied to the
   projection that would alter the behaviour of existing events, so that the 
   library does not mistakenly load an incorrect checkpoint but instead replays
   the stream from the beginning.

### The storage configuration

The storage configuration describes the location of an event stream, which is 
then used by the application to connect to the stream either to read or to
read and write. 

To create a connection string:

 1. Use the `new StorageConfiguration("...")` constructor and pass an 
    Azure Storage connection string. This can be the typical read-write 
	connection string (starts with `DefaultEndpointsProtocol=...`) or a
	connection string with a Shared Access Signature that permits only 
	listing and reading blobs (starts with `BlobEndpoint=...`).

	By default, Lokad.AzureEventStore will store the event stream in the 
	root container of the storage account. To override this, you can 
	append `;Container=...` at the end of the connection string to specify
	another container.

 2. Use the `new StorageConfiguration("...")` constructor and pass an 
    absolute or relative path on your disk, such as `C:\MyStream` or 
	`.\MyStream`. **This method is not acceptable for production use !**
	It is provided so that applications can be run locally on a development
	machine without internet access (and without the Azure Storage 
	emulator), and does not support concurrent access by multiple 
	applications.

 3. Use `Testing.Initialize()` to create a non-persistent in-memory stream
    of events (either empty, or containing a specific list of initial events).

	This function is intended for unit tests, to quickly set up the initial
	state of the stream.

	You can use `Testing.GetEvents()` later on to retrieve all events in such
	a stream, again for unit testing purposes.

### The stream service

To represent an event stream and associated materialized views, the library 
provides the `EventStreamService<IEvent, State>` class. 

```csharp
var svc = EventStreamService<IEvent, State>.StartNew(        
    storage: new StorageConfiguration("..."),
    projections: new[] { new Projection() },
    projectionCache: null,
    log: null,
    cancel: cancellationToken);
```

When created, the service connects to the stream and starts retrieving events, 
building the materialized views on-the fly. If a checkpoint is found and can 
be loaded, then the service will only read events that were generated after 
the creation of the checkpoint. This phase is called the **catch-up**, and 
ends once the service reaches the end of the stream. During catch-up:

 - Property `service.IsReady` returns false.
 - Task `service.Ready` is running.
 - Attempting to access the state or to append new events throws a 
   `StreamNotReadyException`.

#### Accessing the materialized views

Once the catch-up is complete, you can access the materialized views
in two ways: 

 - `service.LocalState` immediately returns the last version of the 
   state. This will not take into account any events appended to the
   stream by other applications since the last time the service 
   interacted with the stream. It will, however, be very fast.
 - `service.CurrentState()` asynchronously returns the current version
   of the state, including any events written to the stream up until 
   the moment the call was made. It performs a round-trip to the 
   Azure Blob Storage, which might take a few milliseconds.

#### Appending new events directly

Call `service.TransactionAsync(builder, cancellationToken)` to perform 
a transaction that can add events to the stream. The builder is either an
`Action<Transaction<IEvent,TState>>` or a `Func<Transaction<IEvent,TState>, TReturn>` 
(in the latter case, the `TransactionAsync` method will include the value returned 
by the builder in the field `.More` of its return value).

```csharp
var email = "...";
await service.TransactionAsync(transaction =>
{
    // Throwing automatically rolls back the transaction
    if (transaction.State.Emails.ContainsKey(email))
        throw new ArgumentException("Mail already exists");

    // Events added to the transaction are appended to the stream
    // after it ends. 
    transaction.Add(new EmailAdded(email));

    // After each Add() the state of the transaction is updated.
    // This also means that if the event causes the projection to 
    // throw, the exception will interrupt the transaction.
    var mailCount = transaction.State.Emails.Count;

    // Transactions can be aborted. 
    transaction.Abort();

}, cancellationToken);
```

The events accumulated during the transaction are then appended to the stream.
If this fails because of a conflict (someone else appended to the stream while
the transaction was running), the events are discarded, the state is refreshed
to take the new remote events into account, and the transaction builder callback
is invoked again. 

A transaction that aborts (or emits no events) automatically succeeds. 

#### Appending new events directly

Note that the preferred way is to use transactions (documented above).

Call `service.AppendEventsAsync(builder, cancellationToken)`. The builder
is a function `Func<State, Append<IEvent>>` which takes the current state 
and returns the list of events to be appended. 

If the builder returns at least one event, the service will then attempt
to append to the event stream. If appending fails due to a conflict 
(because another application appended events and rendered the state 
obsolete), the service will fetch any new events, update the local state,
and call the builder again. This repeats until the builder throws, 
returns no events, or the appending succeeds. 

Function `service.Use(params IEvent[] events)` is a good helper function
for creating an `Append<IEvent>`:

```csharp
// Another occurence of 'word' has been found
await service.AppendEventsAsync(state =>
{
  state.Bindings.TryGetValue(word, out int currentCount);
  return service.Use(new ValueUpdated(word, currentCount + 1));
}, CancellationToken.None)
```

In practice, it is often useful to return some state from the builder back
to the calling context. For instance, if the builder creates a new item in
the state, it should be able to return the identifier of the new item. This
is possible by using an `Append<IEvent, TResult>` instead, which carries data out
of the builder: 

```csharp
// Another occurence of 'word' has been found
var result = await service.AppendEventsAsync(state =>
{
  state.Bindings.TryGetValue(word, out int currentCount);
  return service.With(
      currentCount + 1, // Return the new count
      new ValueUpdated(word, currentCount + 1));
}, CancellationToken.None);

var newCount = result.Result;
```

### Stream Migrations

The application should not be allowed to modify past events. However, as 
a matter of system administration, it is possible to change the event 
stream. This should always be a manually triggered operation, and will 
include the following steps: 

 1. Develop the migration concierge, an executable which reads from the 
    current stream (using `EventStream<T>`) and writes to a new stream 
    (using `MigrationStream<T>`). This will take anywhere from a few minutes
    to several days.
 2. Start the migration concierge, wait for it to catch up to the current
    stream. This will usually take tens of minutes. 
 3. Turn off the application (or re-configure it to be read-only).
 4. Wait for the migration concierge to process all events that were added
    to the stream before the application stopped writing. This should take
    a few seconds.
 5. Stop the migration concierge.
 6. Re-configure the application to use the new stream instead of the current
    stream, and start it. 

You can improve the restart time by pre-generating a state cache based on the
new stream (right after step 2) and uploading it so that the application can
use it to start up more easily.

Example of a migration concierge which drops all events of a defunct type 
`ObsoleteEvent`:

```csharp
var currentStream = new EventStream<IMyEvent>(currentConfig);
var newStream = new MigrationStream<IMyEvent>(newConfig);

await newStream.MigrateFromAsync(
    currentStream, 
    // Drop events of type 'ObsoletEvent'
    (IMyEvent ev, uint _seq) => ev is ObseleteEvent ? null : ev, 
    // When reaching the end of 'currentStream', refresh it every 2 seconds
    // to see if new events have appeared
    TimeSpan.FromSeconds(2),
    default(CancellationToken));
```

## Architecture

This project is split into multiple layers. Starting at the top level 
(the highest level of abstraction) and moving down: 

 - The `EventStreamService` (or ESS) acts as a helpful wrapper 
around a permanently connected stream. On the write side, it 
provides the "append event to stream" primitive. On the read side,
it keeps the current state up-to-date with the latest events in 
the stream. All its operations are thread-safe. Finally, combining 
both read and write access in the same object allows the "try commit, 
if failed update" pattern for transactions. 

 - The `EventStreamWrapper` (or ESW) does the heavy lifting for the ESS, 
in a single-threaded manner (the ESS just adds multi-thread support).

 - The `ReifiedProjection` keeps the state of a projection up-to-date
when it is passed events, and also supports saving and loading of the
projection state to a persistent cache. The ESW will keep one or more
`ReifiedProjection` instances to keep track of its state (or 
sub-components of its state).

 - The `EventStream` provides read-write access to the events in the
stream. It is optimized for sequential, beginning-to-end reads. The
ESW uses the `EventStream` directly when pushing events, and reads the
events to update the state. It is the `EventStream` that generates the 
`uint` sequence numbers for new events.

 - The `IStorageDriver` provides low-level access to persistent storage 
optimized for concurrent appends (several drivers are available, though
only one for production needs). It stores `byte[]` data along with `uint`
sequence numbers, and the `EventStream` is responsible for converting
events to and from the `byte[]` representation.

