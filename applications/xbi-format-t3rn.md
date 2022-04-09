# W3F Grant Proposal: XBI - XCM-based high-level standard and interface (ABI) for smart contracts
> This document will be part of the terms and conditions of your agreement and therefore needs to contain all the required information about the project. Don't remove any of the mandatory parts presented in bold letters or as headlines! Lines starting with a `>` (such as this one) can be removed.
>
> See the [Grants Program Process](https://github.com/w3f/Grants-Program/#pencil-process) on how to submit a proposal.

- **Project Name:** XBI - xcm-based high-level standard and interface (ABI) for smart contracts
- **Team Name:** t3rn
- **Payment Address:** 0x343f822207f65fba7cc5325fd76d879528e706f4
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 1, 2 or 3

> ⚠️ *The combination of your GitHub account submitting the application and the payment address above will be your unique identifier during the program. Please keep them safe.*

## Project Overview :page_facing_up:

### Overview
XCM-based high-level interface for cross-chain smart contract execution.

#### Brief Description
XBI will be an XCM-based Binary Interface that extends the XCM-protocol to enable smart contracts calls, while receiving execution results back to the source Parachain. The same interface will be used to connect Smart Contract VMs installed on other Parachains, as well as to communicate with remote-to-Polkadot blockchains using the XCM protocol, which will be compatible with the bridges of the most active blockchain ecosystems today (i.e. Ethereum, Solana, Avalanche, Cosmos).

#### Rationale
We propose to implement XBI as part of t3rn, a composable smart contracts platform, alongside selected Substrate-based blockchains, such as Moonbeam and Astar. This will be done in order to enable mutual cross-chain smart contract communication between internal-to-Polkadot projects using the same Interface for trust-free communication with remote-to-Polkadot ecosystems.

The XBI interface used by Parchains will also offer a contingencies against runtime upgrades, while allowing Parachains to define and expose their functionalties.

### Project Details
The XBI cross-chain binary interface for smart contracts is a format extension to XCM that allows Parachain to mutually call and retrieve results from:
- smart contracts VMs,
- pallets
- state queries: on Parachains as well as remote-to-Polkadot ecosystems that have an adapter, for a Pallet XBI Executor, to a Parachain's Runtime. Pallet XBI Executor adapts to a bridge linking a Parachain with selected remote-to-Polkadot ecosystem, defining the necessary interface, while configuring the XCM Executor Pallet to provide status responses on sent queries.

Parachains that use XBI can expect the following functionalities:
- a) Trigger smart contract execution on internal-to-Polkadot Parachains:
  - Pallet Contracts WASM smart contracts
  - Pallet EVM smart contracts
  - Other Pallets

- b) Trigger smart contract execution on external-to-Polkadot Ecosystems:
  - EVM-like smart contracts
  - Generic smart contracts

- c) Reveive responses for both successful and unsuccessful executions on both internal and remote-to-Polkadot ecosystems

- d) Expose customized APIs, specific to a Parachain, decodable via XBI Format.

##### Proposed XCM Format Extension
We propose for the XCM format to be extended; standardizing how XCM::Transact is used.

We further propose to introduce two format XCM extensions:
- `XCM::Transact("magicbyte", XCMFE#1, <Scale-encoded-native-call>)` - native runtime dispatch (in case of FRAME - Scale encoded call)
- `XCM::Transact(XCMFE#2, <palletname>::<methodname>, <scale-encoded-args>)` - public Scale-RPC, in case of FRAME - Method name is `<palletname>::<methodname>`.
- `XCM::Transact(XCMFE#3, XBI(<XBI-instance>, XBI-payload))`

## XBI-payload specification
- `call(instance_id/bridge_id)`: `modifications`
  - `call_native`: `trigger Scale encoded native call`
    - `payload: Bytes`
  - `call_evm`:  `trigger smart contract call`
    - `caller: AccountId`
    - `dest: AccountId`
    - `value: Balance`
    - `input: Bytes`
    - `gas_limit: Balance`
    - `max_fee_per_gas: Option<Balance>`
    - `max_priority_fee_per_gas: Option<Balance>`
    - `nonce: Option<u32>`
    - `access_list: Option<Bytes>`
  - `call_wasm`: `trigger smart contract call`
    - `caller: AccountId`
    - `dest: MultiAddress<AccountId, ()>`
    - `value: Balance`
    - `input: Bytes`
    - `additional_params: Option<Vec<ABIType>>`
  - `call_custom`
    - `caller: AccountId`
    - `dest: MultiAddress<AccountId, ()>`
    - `value: Balance`
    - `input: Bytes`
    - `additional_params: Option<Vec<ABIType>>`
- `query`: `access state / read-only` // worth making a batch/related call.
  - `query_evm`:
    - `address: AccountId`
    - `storage_key: Bytes`
  - `query_wasm`:
    - `address: AccountId`
    - `storage_key: Bytes`
- `result`: `(success|failure, <output|failruedetails>, <dest_parachain_witness>)`
- `metadata`: `Lifecycle status notifications`
  - `Sent (action timeout, notification timeout)`
  - `Delivered (action timeout, notification timeout)`
  - `Executed (action timeout, notification timeout)`
  - `Destination / Bridge security guarantees (e.g. in confirmation no for PoW, finality proofs)`
  - `max_exec_cost`: `Balance` : `Maximal cost / fees for execution of delivery`
  - `max_notification_cost`: `Balance` : `Maximal cost / fees per delivering notification`

Each XBI Executor's instance will need to implement the XCM Format for the underlying bridge it connects with.

## XBI Executor
Executers will be responsible for tracking the lifecycle of sent XBI payloads.

Getting the result should trigger a XCM-message back to the original sender of the XBI payload (if the sender subscribed to execution lifecycle status notification). 

The XCM-message will look like this `XCM::Extended|Transact(XCMFE#3, XBI::result(...))`.

### General introduction to proving with XBI Executor
Upon receiving an XBI request, an XBI Executor will generate the associated ID and stores in the state map. This entry to the state map gets updated as soon as the Executor receives the response from one of the Runtime's VM (Default Executor) or an installed Runtime Bridge. 
The state entry is updated with the output response to the requested XBI Payload. As such, it is available for trust-free validation on the requesting Parachain side by sending back the Witness that includes the dispatched call alongside accompanying bytes, which can be decoded to derive the status of the call after the inclusion has already been confirmed. We propose a form of Witness that should work with most external-to-Polkadot ecosystems; suitability will be assessed as part of the first Development Milestone.
```rust
struct Witness {
    encoded_message: Vec<u8>, // Encoded message containing the call dispatch   
    trie_pointer: TriePointer, // Enum pointer, to a merkle tree in that block: state, transaction or logs   
    block_hash: Vec<u8>, // Pointer to a block including the message   
    merkle_path_proof: Vec<Vec<u8>> // Proof - a merkle path including message into block 
}
```

## XBI Executor Instances
### XBI Default Executor
XCM Executor - Default Instance implements the dispatch of local-to-Parachain execution. Default Instance assumes that the received call will be executed on a local pallet: `call_native`, `call_wasm` or `call_evm`.

##### Proving for `Internal|Default`
Since the XCM channel is already established and acts as a trusted method of exchange there is no need to verify the inclusion and provide a witness for dispatched messages on the destination Parachain.

### XBI Default Executor - WASM & EVM Contracts
#### Dispatch - Execute
Converts `XBI:call_wasm` payload into a dispatchable call to Pallet Contracts:
- `call_wasm`: `trigger smart contract call`
  - `caller: AccountId`
  - `dest: MultiAddress<AccountId, ()>`
  - `value: Balance`
  - `input: Bytes`
  - `additional_params:  Option<Vec<ABIType>>`.
    Similarly, `XBI:call_evm` gets converted into dispatchable call to Pallet EVM:
- `call_evm`:  `trigger smart contract call`
  - `caller: AccountId`
  - `dest: AccountId`
  - `value: Balance`
  - `input: Bytes`
  - `gas_limit: Balance`
  - `max_fee_per_gas: Option<Balance>`
  - `max_priority_fee_per_gas: Option<Balance>`
  - `nonce: Option<u32>`
  - `access_list: Option<Bytes>`

#### Result - Source Witness
Converts the output of a Pallet Contract call into `XBI:result`: `(success|failure, <output|failuredetails>)`. Furthermore, it associates the state entry with the call ID, with the output result being `encoded_message`, so that it's possible to source for the Witness arguments attached to response: `block_header`, `encoded_message` and derive the `merkle_path` to the associated call ID state entry.

### XBI Remote Executor
XBI Remote Executor assumes the `XBI::call_remote` will be dispatched to a remote-to-Polkadot Consensus System. XBI leaves the interpretation of remote Consensus Systems relatively broad and shifts the responsibility of providing the trust layer to the XBI Remote Executor implementer. 
In our assumption the most common XBI Remote Executor implementers will be Bridge Providers, such as the Snowfork Bridge to Ethereum. 
In such cases, XBI Executors will be tightly coupled with the kind of bridge they are operating with and will need to fetch the result of a call sent across the bridge.

#### Dispatch - Execute
XBI Remote Executor enforces an interface to implement the `trait DispatchExecutor`, that similarly to Default XBI Instance converts `XBI::call_custom`, `XBI::call_evm` or `XBI::call_wasm` payload to the dispatchable call via the triggers that execute on remote-to-Polkadot consensus system.

#### Result - Source Witness
XBI Remote Executor enforces interface to implement the `fn source_witness`, that retreives the proof of inclusion of the call on remote-to-Polkadot ecosystem. Each bridge implementing XBI Remote Executor needs to implement the source witness method
```rust
trait XBIRemoteExecutor<T: frame_system::Config> {
  
  fn execute_dispatch(xbi_payload: XBIPayload, exec_id: SystemHash<T>);
  
  fn source_witness(xbi_payload: XBIPayload, exec_id: SystemHash<T>) -> XBIWitness;
}
```

## Location of XBI in the stack

XBI Format is a standard over XCM, enabling Parachains with effective communication to use the same interface with various smart contract VMs, installed both at local-to-Polkadot as well remote-to-Polkadot Consensus Systems.

Communication using XCM Format traverses as follows:

- `(trigger) XCM -> (send) XCM>XBI> -> (receive) XBI>DispatchableCall ->  (execute) -> (send) Result->XBI::result -> (receive) XBI result`

The above examples readability could also be enhances with the following example:
``
(send XBIDefaultExecutor::call_custom) Moonbeam -> t3rn (send XBIRemoteExecutor::call_custom) -> Cosmos Bridge -> (native-to-Cosmos execution) Cosmos Chain  -> Cosmos Bridge -> (send XBIExecutor::result) t3rn -> (receive XBIExecutor::result) Moonbeam
``

### XBI payload lifecycle

XBI payload lifecycle can be directed by developers using metadata. XBI Executors implement the functionalities allowing to handle the lifecycle:
#### Metadata
- Lifecycle status notifications
  - Sent (action timeout, notification timeout)
  - Delivered (action timeout, notification timeout)
  - Executed (action timeout, notification timeout)
- Destination / Bridge security guarantees (e.g. in confirmation no )
- Timeout for every lifecycle step.
- Maximal cost / fees
- Notification payment / stipend

#### Lifecycle
- Successfully sent across the bridge (no execution yet)
- Delivery on the other side
- Execution status on the other side
- Execution result / Notification stream

#### Expectations
- Propose XBI Format to be used by t3rn and Parachains, factored in feedback and discussion with selected teams building smart contracts VMs.
- Implemented & Unit Tested pallet-xbi-executor that triggers and proofs execution on both internal and external to Polkadot Ecosystems:
  - EVM calls tested with Pallet EVM
  - WASM calls tested with Pallet Contracts
  - EVM calls tested with Ethereum test Kovan Network (pallet-xbi-remote-executor)
- Live-Tested in XCM environment with collaboration with selected Substrate builders
- Documentation of pallet-xbi-executor interfaces and example usages
- pallet-xbi-executor included in t3rn Runtime
- Docker image including setup for test development network with a template parachain that includes pallet-xi4cc

### Ecosystem Fit

t3rn is a cross-chain smart contracts registry that enable smart contracts execution on mulitple blockchians. The XCM-based Interface will come in a form of a FRAME pallet and will be used by t3rn and any other Substrate-based project that wishes to use it.

The XBI Format and XBI Executors for cross-chain smart contracts will be tested live in a XCM Environment, such as the Rococo network with other selected Substrate builders.
## Team :busts_in_silhouette:

### Team members

- Maciej Baj (team lead)
- t3rn team members: 7 developers

### Contact

- **Contact Name:** Jacob Kowalewski
- **Contact Email:** jacob@t3rn.io, maciej@t3rn.io (CC)
- **Website:** https://www.t3rn.io/

### Legal Structure

- **Registered Address:** Quijano Chambers, Road Town, Tortola, British Virgin Islands, BVI, BC No. 2062235
- **Registered Legal Entity:** t3rn Ltd.

### Team's experience

t3rn team - succesfully completed one Web3 Foundation grant to establish and implement the prototype of t3rn's cross-chain gateways and is now building as part of Substrate Builders Program.

### Team Code Repos

- https://github.com/t3rn

Please also provide the GitHub accounts of all team members. If they contain no activity, references to projects hosted elsewhere or live are also fine.

- https://github.com/MaciejBaj

### Team LinkedIn Profiles (if available)

- https://www.linkedin.com/in/maciej-baj/
- https://www.linkedin.com/in/pauletscheit/
- https://www.linkedin.com/in/jacobkowalewski/
- select members of the [t3rn team](https://www.linkedin.com/company/t3rn-io) - TBD


## Development Roadmap :nut_and_bolt:


### Milestone 1 — Collaborate with Selected partners and gather requirements for XBI Format and XBI Executors Interface

- **Estimated duration:** 1 month
- **FTE:**  2
- **Costs:** $10.000 in ETH

| Number | Deliverable | Specification                                                                                                                                                                                                                                                                                                                                                                                     |
|-------:| ----------- |---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|    0a. | License | Apache 2.0 / GPLv3 / MIT / Unlicense                                                                                                                                                                                                                                                                                                                                                              |
|    0b. | Documentation | Provide both **inline documentation** of the code and a basic **tutorial** that establishes XBI Format. This assumes a series of consulations and feedback loops enhancing the XBI Format usability with min. 2 selected partnered Parachain teams. Tutorials will be done to show how to access the XBI-Executor interface and interact with XBI Format with it as a Substrate-based blockchain. |                                                     |

### Milestone 2 — Implement XBI-Default Executor Module for internal-to-Polkadot smart contracts communication for both EVM and WASM contracts

- **Estimated duration:** 2 month
- **FTE:**  2
- **Costs:** $20.000 in ETH

| Number | Deliverable                                   | Specification                                                                                                                                                                                                                                                                                                                                                                                |
|-------:|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|    0a. | License                                       | Apache 2.0 / GPLv3 / MIT / Unlicense                                                                                                                                                                                                                                                                                                                                                         |
|    0b. | Documentation                                 | We will provide both **inline documentation** of the code and a basic **tutorial** goes over details of XCM Default Executor and covers each `XBI::call_*` operation in detail                                                                                                                                                                                                               |
|    0c. | Testing Guide                                 | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests.                                                                                                                                                                                                                                            |
|    0d. | Docker                                        | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. Dockerfile includes a small development network - Rocococ Relaye chain and 2 parachains connected - one trigerring execution using various `XBI::call_*` operations, the second one receving the XBI Payload, dispatchig it and returning results back to the source Parachain. |
|    0e. | Article                                       | We will publish an **article**/workshop that elaborates on the local-to-Parachian smart contracts execution using XBI within the Polkadot-like ecosystems.
|     1. | Substrate module: XBI Local Executor - WASM   | We will create a Substrate module receives `XBI::call_wasm` payload, producuces dispatchable Pallet Contracts call, triggers execution of WASM contract and source the witness containing execution results in order to send the `XBI::result` message back                                                                                                                                  |  
|     2. | Substrate module: XBI Local Executor - EVM    | Similar to above, create a functionality to handle dispatches to Pallet EVM installed on that Parachain pallets using `XBI::call_evm`                                                                                                                                                                                                                                                        |          
|     3. | Substrate module: XBI Local Executor - Native | Similar to above, create a functionality to handle dispatches to Native to that Parachains pallets using `XBI::call_native`                                                                                                                                                                                                                                                                  |     |  


### Milestone 3 — Implement XBI External Executor Module for external-to-Polkadot smart contracts communication for both EVM contracts tightly coupled with Snowfork Bridge

- **Estimated duration:** 2 month
- **FTE:**  2
- **Costs:** $20.000 in ETH

| Number | Deliverable                                      | Specification                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|-------:|--------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|    0a. | License                                          | Apache 2.0 / GPLv3 / MIT / Unlicense                                                                                                                                                                                                                                                                                                                                                                                                                            |
|    0b. | Documentation                                    | We will provide both **inline documentation** of the code and a basic **tutorial** goes over details of XCM External Executor and cover each `XBI::call_evm` operation send to test Ethereum network - Kovan with tightly coupled to XBI External Executor Snowfork Bridge                                                                                                                                                                                      |
|    0c. | Testing Guide                                    | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests.                                                                                                                                                                                                                                                                                                               |
|    0d. | Docker                                           | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. Dockerfile includes a small development network - Rococo Relaye chain and 2 parachains connected - one trigerring execution using `XBI::call_evm` that assumes External XBI Executor on destination, the second one receving the XBI Payload, coupling it with Snowfork Bridge, dispatchig it and returning results back to the source Parachain. |
|    0e. | Article                                          | We will publish an **article**/workshop that elaborates on the local-to-Parachian smart contracts execution using XBI within the Polkadot-like ecosystems.
|     1. | Substrate module: XBI External Executor - EVM    | We will create a Substrate module receives `XBI::call_evm` payload, producuces dispatchable to Snowork Bridge call to Ethereum, via the bridge triggers execution of EVM contract and sources the witness from the bridge containing execution results in order to send the `XBI::result` message back                                                                                                                                                          |  
|     2. | Substrate module: XBI External Executor - Custom | Similar to above, create mock up as a template for other Consensus Systems, for example Cosmos Bridge.                                                                                                                                                                                                                                                                                                                                                          |          
|


## Future Plans

t3rn is building a cross-chain smart contract hosting platform that enable smart contracts execution on mulitple blockchians. XBI will help contribute to the t3rn vision of a fully connected cross-chain ecosystem, rooted in Polkadot. 

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?** 
This is our second Web3 Foundation grant, having delivered on our first grant back in December 2020. We having been working tirelessly within the Polkadot ecosystem ever since, as part of the Substrate Builders Program and intend to launch as a Polkadot parachain in summer 2022. 
