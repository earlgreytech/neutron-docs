# Spec Components

## Component Detail

* CoMap -- a set of communication maps for smart contracts talking primarily to external smart contracts and vice versa, an integral piece of NeutronABI specifications
* CoStack -- a set of communication stacks for smart contracts talking to Neutron Elements and vice versa, an integral piece of ElementABI specifications
* CoData -- The CoMap and CoStack data structures, named as a whole together.
* VMManager -- This components keeps track of the various VMs and their hypervisors, allowing for Neutron to use multiple virtual machines
* Neutron Hypervisor -- a hypervisor is what mediates between the smart contract executing within a VM and the Neutron Call System and CoData.
* Neutron ARM Hypervisor -- The specific working mechanisms of the ARM VM hypervisor will be included
* NARM VM -- The specifications for the ARM VM itself will be included here for posterity, even though it may not exactly belong here. 
* Neutron Call System -- The core interface by which all other pieces of Neutron can communicate with each other
* Element ABI -- An ABI used for communicating with Neutron Elements. It primarily uses the CoStack data structures for communication. All Neutron Element APIs must use the Element ABI.
* Neutron ABI -- The ABI used for communicating with smart contracts. Smart contracts are not forced to use it, but it is expected that the NeutronABI will be extended in backwards compatible ways, rather than a complete break from the groundwork laid down here. Using an alternative ABI would require some tooling changes, but does not require any consensus-critical changes within Neutron \(ie, a hard-fork\) 
* Neutron Element API Proposal \(NEAP\) -- Neutron will include standard proposals for various Element concepts. These include things like a standard interface for UTXO based blockchains, wallet management, cryptography standards, etc. The list described here will be non-exhaustive and is expected to be expanded as time goes on.
* NeutronDB -- The consensus-critical database for smart contract state to be implemented along with Neutron. Note that Neutron could run without any database, or with an alternative one. 

### CoMap

The CoMap is the central piece of communication for almost all things within Neutron. It is used for passing call data into smart contracts, for smart contracts to pass parameter data to Element APIs, and various other uses. It is expected to also be a central point of abuse for smart contracts, and so must be carefully designed to avoid exploits forming here. Each contract execution has access to three CoMap structures, titled Inputs, Results, and Outputs. The Inputs and Results CoMap is read-only. The Outputs CoMap is write-only. Is it possible to copy the entire inputs/results map into outputs, as well as to copy individual keys without involving smart contract code explicitly loading the key into memory and then storing the data into the outputs array.

Both CoMaps are quite volatile in how they operate in order to minimize memory usage and the potential for abuse. Specifically, the following happens upon entering an Element/Contract:

* The outputs CoMap from the caller is made accessible as the inputs CoMap to the callee
* A new CoMap is made for the outputs CoMap for the callee
* A new CoMap is made for the resutls CoMap for the callee

When the Element/contract returns control to the caller, the following happens:

* The outputs CoMap from the callee will overwrite the results CoMap for the caller
* The outputs CoMap from the caller is cleared and made empty
* The callee's inputs are destroyed \(since they are aliased as the outputs of the caller\)

Notice that over the entire contract execution, the Inputs map remains accessible and preserved, the Outputs map becomes a method by which to both return data to the calling contract, as well as to communicate with Elements or sub-contracts, and finally the Results map allows accessing sub-call results without interfering with Inputs, nor keeping every map of each sub-call accessible. In some contract call flows, it may be necessary to copy some data from a map temporarily into memory to preserve it, but this shouldn't be a common case. Ideally, as little data copying needs to be done as possible, while also putting an upper bound on the number of maps which must be preserved during contract execution.

Note that for gas purposes, there are various pressure variables on the amount of CoMap data being tracked, thus there should be pressure to prevent using the CoMap as a generalized data store

Functions:

* `push_output (key, type, data)`
* `push_output_with_type(key, data_with_type)`
* `peek_input(key) -> data`
* `peek_input_with_type(key) -> (type, data)`
* `peek_result(key) -> data`
* `peek_result_with_type(key) -> (type, data)`
* `copy_input_to_output(key)`
* `copy_result_to_output(key)`
* TBD `count_map_swaps() -> count` -- This can be used for smart data structures which can be made easily aware of invalidations to output data. This value is incremented every time the outputs map is invalidated
* `get_incoming_transfer_value(address: NeutronAddress, id: u64) -> value: u64`
* `get_incoming_transfer_info(index: u32) -> (address: NeutronAddress, id: u64)` -- note, both output parameters use the costack

Note: most hypervisors are expected to include max length, beginning index, etc parameters to allow for easier subset access to each piece of data within the comaps without needing to copy more data than necessary into the VM memory. Hypervisors may allow direct access to comap functionality, or may require the usage of the CoStack in order to load the data indirectly into the VM

### CoStack

The CoStack is used for passing data in and out of ElementAPI calls. It is used rather than CoMap to avoid overhead with key names and generally unneeded complexity. There are two CoStacks available for smart contracts, read-only inputs, and write-only outputs. Smart contracts can not receive data from external contracts via the inputs stack, this can only be done by CoMaps. CoStack is only used for ElementAPI communication. Some ElementAPI functions may optionally, or by a requirement of the element, use the CoMap concept as well.

Unlike CoMaps, there is only one instance of the two CoStacks. In other words, CoStack data is not preserved across external contract calls. The input stack will contain only the result of the external contract call and the output stack will be cleared.

Functions:

* `push_stack_output(data)`
* `pop_stack_input() -> data`
* `peek_stack_input() -> data`
* `clear_stacks()` -- clears both input and output stacks. Might later be rewarded with a gas refund for doing this
* `pop_stack_into_output_comap()`
* `pop_stack_into_output_stack()`

### CoStack and CoMap meta info

CoStack and CoMap data structures may be handled by the same overall interface/structure. It shall be possible to get some info about the CoStack and CoMap structures

* `remaining_memory() -> size`
* `remaining_stack_items() -> count`
* `reentrancy_count() -> count` -- counts the number of times the current smart contract has been called within the current call stack \(1 = only once\)

Along with contract accessible data, there is also hidden various functions which should not be exposed to contracts directly:

* `push_context(context) -> void` -- this will push a new context onto the Context Tracking Stack
* `peek_context(index) -> context` -- this will get the context at the specified index in the stack
* `pop_context() -> context` -- this will destroy the current context and return it's content
* `context_count() -> count` -- this will return the total number of contexts currently held
* `comap_released() -> bool` -- \(ElementAPI use only, TBD\) indicates if the caller has released access to the comaps

Each context is an item in the call stack and represents an execution of a smart contract.

Constants used:

* `CODATA_MAX_ELEMENT_SIZE` -- proposed, 64Kb. This is the maximum size of a single stack item on the CoData. This limit has broad effects on ABI design, hypervisor implementation, and internal communications. Thus it would be extremely difficult to change after the fact.
* `CODATA_MAX_TOTAL_MEMORY` -- proposed, 2Mb. This is the maximum size of all elements on the stack put together, excluding context stack info. This determines the overall maximum amount of memory that can be consumed by the CoData. This would be hard coded into various pieces of infrastructure, and so reducing it after the fact will be extremely difficult. However, it can easily be made larger without breaking compatibility
* `CODATA_MAX_ELEMENT_COUNT` -- proposed, 256. This is the maximum number of elements that can be held on the stack. As with total memory, this is hard to make smaller after being set, but easy to make larger
* `CONTEXT_STACK_MAX_COUNT` -- proposed, 128. This is the total number of different contexts which can be held. It is expected that gas costs will prevent exceeding the actual count here, as each context added equates to a new VM instance and new call operation.

### Neutron Call System

The Neutron Call System is what works together with CoStack to facilitate all intercommunication between smart contracts and different ElementAPIs and thus the final underlying blockchain.

The Call System can be called from other Elements or from smart contract code. Calling it from an external interface uses a simple interface consisting of 2 inputs \(arguments\) and 2 outputs \(results\).

Inputs:

* `ElementID, u32` -- The Element to contact regarding this call
* `FunctionID, u32` -- The actual function to utilize within the Element

Outputs:

* `Result, u32` -- The result code. In Rust implementations, errors are handled so that it appears to give 2 outputs, one with a non-error result and one with an error result
* `Unrecoverable, bool` -- If this is set, then an error has occurred which means that the entire smart contract execution \(including any further up contexts\) must immediately terminate, reverting all state. This can happen as a result of reading out of state rent, or encountering an "impossible" error within some Neutron Infrastructure.

From this, the interface would appear quite limited, however, additional arguments, results, and data can be passed using the CoStack, which are shared between both the caller and callee.

There is a standard for ElementIDs and FunctionIDs. Specifically, if the top bit is set on either an ElementID or FunctionID \(ie, greater than `0x8000_0000`\) then the Element or Function is considered to be a platform specific one. This would mean that these elements/functions are not expected to be shared or implemented in any other blockchain. In addition the FunctionID of 0 is reserved. This is used to test if a particular ElementID is available. The element should return a result of 0 or greater if it exists. It should otherwise return an element does not exist error code.

There is no standard for result codes, but anything greater than or equal to `0x8000_0000` is regarded as an error. This could include errors such as reading a stack item that doesn't exist, running out of gas, etc. In some cases this may cause the current smart contract execution to terminate, but will not cause a chain of terminations up the context stack like an unrecoverable error would.

The other interfaces beyond actual Element calls include:

* Logging interface, for logging errors, info, debug messages, etc.
* Tracking of block height or another mechanism which is used to determine what Elements and features are currently available to smart contracts \(ie, to handle forks\)
* Initial loading of state data for smart contract bytecode
* Platform-specific methods of beginning the execution of a smart contract VM

The Call System is regarded as a "platform provided component". This means that each blockchain implementation will provide its own version of the Call System. In actual implementation terms, there may be a lot of borrowed code through interfaces, traits, classes, etc. However, the Call System needs to be aware of several pieces of platform specific information.

The specific responsibilities of the Call System includes:

* Holding a list of ElementIDs and their mapping to Element components when called upon
* Tracking the block height and which Elements are enabled at any one time within the blockchain
* How to load smart contract bytecode from the consensus database
* How to write smart contract bytecode into the consensus database, optionally \(ie, if the blockchain provides a proper writeable database\)
* Initiates the top level execution into a smart contract \(ie, to translate from a received transaction on a blockchain into a smart contract execution\)
* Tracks the different VM hypervisors and interprets smart contract external calls to call the appropriate VM
* Implements the logging system used by other Element components for informative messages which are not tracked for consensus purposes
* Handles checkpoints \(if implementing a database\) within the state database to properly allow for reverting state in the case of errors

The CallSystem is implemented in the Rust reference implementation in such a way that reentrant calls into the same ElementAPI are impossible. For instance, if a custom method was implemented on GlobalState called "store\_bytecode\_upgrade\_result" and this ended up calling a custom BytecodeUpgrade element which then further called GlobalState again to use store\_state, then the this would result in an error. This is not a strictly defined behavior, but given the complexities that could be introduced by such reentrancy, it is not recommended for other implementations of Neutron to allow such behavior.

### Neutron Hypervisor

A Neutron Hypervisor is a component which exposes an interface for Neutron to a VM. Because each VM could be radically different, there is no one size fits all approach that is appropriate. However, the specific key pieces that should be exposed in a hypervisor includes:

* CoData operations, including placing data on the stack and getting data off of the stack
* Current context information, including gas used, gas limits, self-address information, etc
* An interface to call Neutron Element APIs

There is additional "typical" responsibilities such as entry point handling, memory allocation, etc. However, these are highly variable between the different types of potential smart contract VMs. Neutron is built to allow for many different types of VMs and so there are minimal demands on this specific component to accommodate this goal.

### 

