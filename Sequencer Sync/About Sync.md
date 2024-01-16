> ⚠️ Syncing from the Starknet sequencer in Deoxys occurs in multiple steps. It should *not* be seen as a single event but rather as the combination of **multiple action communicating together**.

---
# [`new_full`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L269)

> 📚 `new_full` can be seen as responsible for setting up the substrate environment by spawning multiple worker threads, including L2 synchronization.

Synchronization is implemented via a separate worker thread which runs [`client::deoxys::src::sync`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/lib.rs#L16). Data is then relayed to a separate thread to store the synchronized information in the [pre-runtime digest](https://releases.parity.io/substrate-rustdoc/sp_runtime/enum.DigestItem.html#variant.PreRuntime).

---
# Block sync

Block sync in Deoxys takes place in [`client::deoxys::src::l2::sync`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/l2.rs#L89), which is spawned in [`new_full`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L269). This part of Deoxys is responsible for retrieving block data and state differences between blocks from the Starknet sequencer. The resulting blocks and state differences are then emitted via a [tokio thread synchronization buss (`mpsc`)](https://docs.rs/tokio/latest/tokio/sync/#mpsc-channel). 

> ℹ️ Note that as of this step, blocks and state differences have been computed *but have not been stored*: this will be handled on the consuming of messages by the mpsc thread synchronization bus.

---
# Saving to the pre-runtime digest

Storing to the pre-runtime digest is handled in [`new_full`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L269) by a [`QueryBlockConsensusDataProvider`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L641C12-L641C43). This structure is responsible for generating a digest when called and is used to provide a pre-digest source. This is done as part of a custom handler attached to [manual sealing](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L693) operations (ie: it will be called every time a manual seal is conducted).

> ℹ️ Since the goal of Deoxys is full synchronization as fast as possible, it does not make sense to be creating blocks at a fixed interval as would be the case on a L1 full node (an L1 full node acts as a combine Starknet node and sequencer on L2). Since this is a core assumption of substrate, it is currently necessary to use manual sealing mode to generate new blocks.

> ⚠️ This generates further issues by providing no consensus mechanism for adding blocks to the local merkle tree db. This is fine with the assumption of a single source of truth Starknet sequencer, but falls apart once decentralization and/or p2p have been implemented.

---

# Creating new blocks

> ⚠️ [`QueryBlockConsensusDataProvider`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L641C12-L641C43) should [`client::deoxys::src::l2::sync`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/l2.rs#L89) be viewed as different services running in parallel but with strong interactions. [`client::deoxys::src::l2::sync`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/l2.rs#L89) serves as a *block and state difference producer*, which [`QueryBlockConsensusDataProvider`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L641C12-L641C43) then consumes.

New blocks are created as part of L2 synchronization:
1. Once a new block and state diff have been retrieved from the Starknet sequencer, they are sent via a thread synchronization bus to be later read by [`QueryBlockConsensusDataProvider`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/node/src/service.rs#L641C12-L641C43).
2. If both state diff and new block have been retrieved successfully, the L2 synchronization worker thread will then [initialize a call to write then new block](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/l2.rs#L113) to the local merkle tree db.
3. This will call the Substrate event [`on_initialize`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/pallets/starknet/src/lib.rs#L194), which is called at the start of block creation.

> ℹ️ Because Starknet blocks do not adhere strictly to substrate's storage policy, it is necessary to find alternative ways to store the required information as part of the substrate db. This is done in [`on_initialize`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/pallets/starknet/src/lib.rs#L194) by [manually storing](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/pallets/starknet/src/lib.rs#L211) state diff items from the [pre-runtime digest](https://releases.parity.io/substrate-rustdoc/sp_runtime/enum.DigestItem.html#variant.PreRuntime) into the local db.

4. Further events will trigger to set up the substrate block, and once these have resolved, a call to [`on_finalize`](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/pallets/starknet/src/lib.rs#L185) will be issues. This is used to finally [store the Starknet block](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/pallets/starknet/src/lib.rs#L991) as part of a substrate block.