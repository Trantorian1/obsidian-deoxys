Deoxys uses a custom storage handler (referred to in the codebase as a `StorageOverride`) on top of Substrate's storage capabilities. This allows for more optimized storage handling by having the handler tailor-made to Starknet block storage.

**Storage overrides** are provided for each type of Cairo contract as follows:

- An `OverrideHandle` contains a binary tree hashmap of `StorageOverride` providing individual storage handling for each Cairo contract version. A default behavior is provided as a `fallback`

- A method `for_schema_version` allows to retrieve the specific `StorageOverride` from a `OverrideHandle` for further operations.

- Cairo v0 is represented as `StarknetStorageSchemaVersion::Undefined`

- Cairo v1 is represented as `StarknetStorageSchemaVersion::V1`

```rust
pub struct OverrideHandle<B: BlockT> {
	/// Cairo contract version to storage handler equivalence
    pub schemas: BTreeMap<StarknetStorageSchemaVersion, Box<dyn StorageOverride<B>>>,
    /// Default data retrieval in case a version-specific handler cannot be found
    pub fallback: Box<dyn StorageOverride<B>>,
}
```

> ⚠️ As of now, this is a particularly naive implementation of storage handling as **only 2 storage handles are ever stored**: Cairo v0 (labelled as `StarknetStorageSchemaVersion::Undefined` ???) and Cairo v1. Note that Cairo v0 is *not even stored in the* `schemas` *hashmap* but rather in the `fallback`.

