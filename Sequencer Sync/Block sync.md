Synchronization happens 

- *Using deprecated functions* `get_block` *on* `SequencerGatewayProvider` [here](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/l2.rs#L142).
- *Using deprecated functions* `get_state_update` on `SequencerGatewayProvider` [here](https://github.com/KasarLabs/deoxys/blob/8f746a950de942b74e4d62f40027acecf85e2154/crates/client/deoxys/src/l2.rs#L162).

> ℹ️ Use [`Provider`](https://docs.rs/starknet-providers/latest/starknet_providers/trait.Provider.html#tymethod.get_state_update) trait instead.

