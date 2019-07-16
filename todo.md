todo
====
## refactor items
#### proto / messaging
- [x] refactor the proto buf message layout. Must be top-level requests with responses. This will allow users of the library to more easily use the protobuf definitions with RPC systems, and allows them to wrap them in more unified enums if needed.
- [x] get rid of protobuf and just use standard Rust types with serde. Protobuf, capnproto, flatbuffers and any other data scheme integrations can be easily managed with Rust's standard `From/Into` traits.
- [x] top level requests should also include the ID of the node which the RPC is targeting. This will more easily allow applications to make routing decisions.
- [x] top level proto requests should have actix::Message impls. The Result::Ok should be the corresponding response type and the Result::Err should just be `()`, as raft derives no meaning from application level or network level failures. Errors will simply causes RPCs to be sent again.
- refactor client requests to indicate they they are entirely application specific.
    - [x] should be updated to require `Vec<proto::EntryNormal>` as these are the only entry types allowed for client requests.
    - [x]x ClientRpcIn should be simply renamed to `ClientRequest`.
    - [x] The old raft rpc in & out variants can be removed, as they will no longer be needed.
- [x] update the way the Raft actor handles these message types and responses.
- [x] ~~for users of this crate, it would be logical to copy the `raft.proto` to their own app, as the protobuf definition is strictly versioned and any changes to it will coincide with a version release of this crate. Perhaps in the future we can add some tooling patterns for being able to sync this repo's raft.proto at a specific version and then have it built with prost in a `build.rs` file ... should be pretty simple. Fetch the proto file from github at the specific version. Maybe bundle the proto file in the crate. Then point to the file and build it like normal.~~
- [x] finish up work on the replication stream actor now that everything else has been refactored.

#### network
- [x] Update everything to use the RaftNetwork trait. This is a much more clear and concise pattern.

#### raft
**Arc<_> Arc<_> Arc<_>!!!**
- [x] update the replication protocol & the client request handling protocol to simply `Arc` the vector of entries which need to be applied and such. The storage layer will no longer need to return anything as a response when appending entries to the log or applying entries to the state machine.
- [x] replication streams can be sent an arc of the entries as well. When entries need to be buffered, the arcs themselves can be buffered and then the entries can be transformed into a larger payload when ready to be sent.
- ~~create an `EntriesPayload` type which wraps the `Arc<Vec<Entries>>` so that we have a more concise interface into knowing the first & last index of the entries, the term of the last entry &c.~~
- [x] `storage::AppendLogEntries` should have a field indicating if the request is from replication requests (node is follower) or from client requests (node is leader). Update docs to indicate when it is acceptable to return a custom error.
- [x] `storage::AppendLogEntries` docs need to indicate that appending a batch of entries when the node is the leader must be atomic. The whole batch must either succeed or all must fail. For replication, failure is not allowed.

##### algorithm optimizations
- [ ] update the AppendEntries RPC receiver to pipeline all requests. This will potentially remove the need for having to use boolean flags to mark if the Raft is currently appending logs or not.
- [ ] in the client request handling logic, after logs have been appended locally and have been queued for replication on the streams, the pipeline can immediately begin processing the next payload of entries. **This optimization will greatly increse the throughput of this Raft implementation.**
    - this will mean that the process of waiting for the logs to be committed to the cluster and responding to the client will happen outside of the main pipeline.
    - multiple payloads of enntries will be queued and delivered at the same time from replication streams to followers, so batches of requests may become ready for further processing/response when under heavy write load.
    - applying the logs to the state machine will also happen outside of the pipeline. It must be ensured that entries are not sent to the state machine to be applied out-of-order. It is absolutely critical that they are sent to be applied in index order.
    - the pipeline for applying logs to the statemachine should also perform batching whenever possible.

----

## impl
- [x] actix messaging errors from messaging the RaftStorage should cause Raft to stop. This functionality is fundamentally required for the system to work.
- [x] finish up handlers of the replication stream messages.

#### snapshots
- [ ] get the system in place for periodic snapshot creation.

#### admin commands
- [ ] get AdminCommands setup and implemented.

#### observability
- [ ] ensure that internal state transitions and updates are emitted for host application use. Such as RaftState changes, membership changes, errors from async ops.
- [ ] add mechanism for custom metrics gathering. Should be generic enough that applications should be able to expose the metrics for any metrics gathering platform (prometheus, influx, graphite &c).

#### testing
- [ ] finish implement MemoryStroage for testing (and general demo usage).
- setup testing framework to assert accurate behavior of Raft implementation and adherence to Raft's safety protocols.
- all actor based. Transport layer can be a simple message passing mechanism.

----

## docs
- put together a networking guide on how to get started with building a networking layer.
- put together a storage guide on how to implement an application's RaftStorage layer.
- add docs examples on how to instantiate and start the various actix actor components (Raft, RaftStorage impl &c).