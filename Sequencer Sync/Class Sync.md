> 🚨 As of now, Deoxys **does not implement class sync** from the Starknet sequencer. This is necessary so as not to have to resort to later, individually costly calls to retrieve class data.

> ℹ️ Class data can be retrieved from the state diff during synchronization with the Starknet sequencer.

---
# Proposed solution:

1. Extract class data from state diff during L2 sync in [`client::deoxys::src::l2::sync`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/l2.rs#L89).
2. Similar to how this is done for blocks and state diff, synchronize class data with pre-runtime digest provider in [`node::src::service::run_manual_seal_authorship`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L641C3-L641C3).
3. This will allow to access class data in a `no-std` environment in when creating a new block in [`on_initialize`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/pallets/starknet/src/lib.rs#L194) and [`on_finalize`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/pallets/starknet/src/lib.rs#L185).

---
# Further information

Bellow is a summary of how Pathfinder implements it's own class sync system:

> ℹ️ Syncing logic begins in `pathfinder/crates/pathfinder/src/state/sync.rs` in `sync` function.

Sync handled through an **event buss** (`event_sender` / `event_receiver`). These are handled by [`l1Sync`](https://github.com/eqlabs/pathfinder/blob/main/crates/pathfinder/src/state/sync.rs#L125) and [`l2Sync`](https://github.com/eqlabs/pathfinder/blob/main/crates/pathfinder/src/state/sync.rs#L126) lamdas which act as event providers to then be read by a [`consumer`](https://github.com/eqlabs/pathfinder/blob/main/crates/pathfinder/src/state/sync.rs#L331).

> 💡 It appears l1Sync is responsible for sending information about l1 synchronization status (this is where pending blocks are determined), while l2Sync is responsible for sending l2 block data, such as *block hashes* and *contract classes*.

---

# L2 sync

> ℹ️ *L2 [`sync`](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/pathfinder/src/state/sync/l2.rs#L92) implementation is in [`pathfinder/crates/pathfinder/src/state/sync.rs`](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/pathfinder/src/state/sync/l2.rs)*

1. Block retrieval occurs [here](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/pathfinder/src/state/sync/l2.rs#L425).
	1. Block transaction hash mismatch calculations occur [here](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/pathfinder/src/state/sync/l2.rs#L523).
	
2. Next state update calculate [here](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/pathfinder/src/state/sync/l2.rs#L147).
	1.  Pending blocks are retrieved at the same time as the [`StateUpdate`](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/gateway-types/src/reply.rs#L1692) which is deserialized using serde_json. This includes a [`StateDiff`](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/gateway-types/src/reply.rs#L1783) which store changes between blocks. It is then converted into [`pathdinder_common::StateUpdate`](https://github.com/eqlabs/pathfinder/blob/main/crates/common/src/state_update.rs#L15) which is derived from the `StateDiff`.

3. New classes are retrieved [here](https://github.com/eqlabs/pathfinder/blob/bb08596e2bd6b05fee90d85ea63845789935a3fa/crates/pathfinder/src/state/sync/l2.rs#L240) based on the `StateDiff` and are individually downloaded if they are not already part of the local database.
	1. Missing classes are downloaded using a call to the sequencer.
	2. Classes using Sierra (Cairo v1) are then compiled to casm from the resulting JSON.
	3. If casm compilation for a Sierra contract fails, a call to the sequencer for the existing casm is made (hence step *2* can be seen as an optimization avoiding repeated calls to the sequencer).

> ℹ️ Class hashes can then be retrieved from a local database without any further calls to the sequencer.

---
# Dissecting [`StateUpdate`](https://github.com/eqlabs/pathfinder/blob/main/crates/common/src/state_update.rs#L15)

```rust
#[derive(Default, Debug, Clone, PartialEq, Dummy)]
pub struct StateUpdate {
    pub block_hash: BlockHash,
    pub parent_state_commitment: StateCommitment,
    pub state_commitment: StateCommitment,
    // aggregates non-reserved contract storage diffs, new contract deployments
    // and replaced contract classes. This essentially stores all non-system
    // contract state mutations compared to the previous block.
    pub contract_updates: HashMap<ContractAddress, ContractUpdate>,
    // stores storage diffs for Starknet System contract at address 0x1
    // This is a reserved contract address, used to to store values for smart 
    // contracts used during syscalls. 
    pub system_contract_updates: HashMap<ContractAddress, SystemContractUpdate>,
    // stores newly declared cairo v0 classes
    pub declared_cairo_classes: HashSet<ClassHash>,
    // stores newly declared cairo v1 classes 
    pub declared_sierra_classes: HashMap<SierraHash, CasmHash>,
}
```
