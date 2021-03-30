# Neutron ARM VM

## NARM Subset of ARM

Due to various toolchain problems as well as extra complexities that come with x86, an ARM VM is proposed as an alternative. An ARMv6-M VM has already been implemented as the "narm" project, but ARMv6-M is missing several key opcodes necessary for efficient smart contracts. Thus, ideally narm would be expanded into ARMv7 or ARMv8.

The memory map and interface for narm is similar. Instead of interrupts being used for hypervisor communication, narm uses the `SVC` opcode. The subset would have all privileged opcodes removed, use a similar "split" of readable/writeable memory, and share a similar memory map.

Further details on the exact subset are TBD

## The NARM hypervisor

The ARM Hypervisor ties into the the narm VM and allows for communication to and from it into Neutron. It accomplishes this by implementing a series of system calls that are exposed to smart contract programs. These system calls are simple interrupts using a consistent ABI for passing arguments etc. The hypervisor also manages the state used for a smart contract's bytecode and non-mutable data, as well as interpreting data from Neutron which can result in either a smart contract creation or call.

### ARM Service Call ABI

This ABI will follow the "new" Linux eabi protocol.

The 32-bit registers used for passing in data to a system call are, in order:

* r0
* r1
* r2
* r3

The registers which can be used for returning data from a system call are:

* r0
* r1 \(64 bit results only\)

If more data is needed, or if dynamic length data is used, then the CoStack should be used.

For reference, given an interrupt of the format `do_stuff(foo, bar, baz, bim, fam) -> zam:u64` the register usage would be as so for input:

* r0 = foo
* r1 = bar
* r2 = baz
* r3 = bim
* first item popped from costack = fam

And as so for output:

* r0 = lower 32 bits of zam
* r1 = upper 32 bits of zam

The list of operations supported by ARM Service Calls are:

Note all operations here unless specified are classified as "pure". "variable" means that the type of operation may be pure or another type depending on exact arguments etc.

For the mixing of registers and the CoStack in hypervisor system calls, consider this example:

`function(arg1, arg2, stack arg3, stack arg4) -> (result1, result2, stack result3, stack result4)`

This function would be used from the smart contract like so:

* r0 = arg1
* r1 = arg2
* push arg3
* push arg4
* call\_function\(\)
* r0 = result1
* r1 = result2
* pop result3
* pop result4

Misc:

* SVC 0x00: nop -- Always considered a no-operation with no modifications to CPU state

CoStack operations: --note: CoStack functions are limited to 4 u32 register parameters

* SVC 0x10: push\_costack \(buffer: pointer, size: u32\)
* SVC 0x11: pop\_costack \(buffer: pointer, max\_size: u32\) -&gt; actual\_size: u32 -- note: if buffer and max\_size is 0, then the item will be popped without copying the item to memory and only the actual\_size will be returned
* SVC 0x12: peek\_costack \(buffer: pointer, max\_size: u32, index: u32\) -&gt; actual\_size: u32 -- note: if buffer and max\_size is 0, then this function can be used solely to read the length of the item. 
* SVC 0x13: dup\_costack\(\) -- will duplicate the top item on the stack
* SVC 0x14: costack\_clear\(\) -- Will clear the stack completely, without giving any information about what was held on the stack
* SVC 0x15: peek\_partial\_costack\(buffer: pointer, begin: u32, max\_size: u32\) -&gt; actual\_amount\_read: u32 -- will read only a partial amount of data from an SCCS item in the middle of the item's data \(starting at 'begin'\)

Call System Functions:

* SVC 0x20: system\_call\(feature, function\):variable -&gt; error:u32 -- will call into the NeutronCallSystem
* SVC 0x21: system\_call\_with\_comap\(feature, function\):variable -&gt; error:u32 -- will call into the NeutronCallSystem

CoMap operations:

Note: abi\_data actual size can be determined by reading the top 2 bits:

* 00 -- single byte \(top byte only, others will be 0\)
* 01 -- two bytes \(top byte and upper middle byte, others will be 0\)
* 10 -- four bytes \(all four bytes\)
* 11 -- reserved/unknown

abi\_data in "raw" calls will be variable sized, but in using the parsing functions, will always be treated as a u32

* SVC 0x30: push\_comap\(key: stack \[u8\], abi\_data: u32, value: stack \[u8\]\)
* SVC 0x31: push\_raw\_comap\(key: stack \[u8\], raw\_value: stack \[u8\]\)
* SVC 0x32: peek\_comap\(key: stack \[u8\], begin: u32, max\_length: u32\) -&gt; \(abi\_data: u32, value: stack \[u8\]\) --note max\_length of 0 is treated as "peek size only" and will not write any data. The begin parameter can be used to only read a subset of the data. In the case of a key not existing, abi\_data is 0 and value will be an empty stack item
* SVC 0x33: peek\_raw\_comap\(key: stack \[u8\], begin: u32, max\_length: u32\) -&gt; \(raw\_value: stack \[u8\]\)
* SVC 0x34: peek\_result\_comap\(key: stack \[u8\], begin: u32, max\_length: u32\) -&gt; \(abi\_data: u32, value: stack \[u8\]\)
* SVC 0x35: peek\_raw\_result\_comap\(key: stack \[u8\], begin: u32, max\_length: u32\) -&gt; \(raw\_value: stack \[u8\]\)
* SVC 0x36: clear\_comap\_key\(key: stack \[u8\]\)
* SVC 0x37: clear\_comap\_outputs\(\)
* SVC 0x38: clear\_comap\_inputs\(\)
* SVC 0x39: clear\_comap\_results\(\)
* SVC 0x3A: copy\_input\_to\_output\(key: stack \[u8\]\)
* SVC 0x3B: copy\_result\_to\_output\(key: stack \[u8\]\)

  --todo: specific key copying operations

CoMap Transfer Operations:

* SVC 0x3C: get\_incoming\_transfer\_value\(token\_contract: stack NeutronAddress, token\_id: stack u64\) -&gt; value: stack u64 -- if no transfer has been sent, then the value returned is 0, rather than an error

It is expected that for a more general purpose contract capable of receiving multiple tokens, that the ABI data will include a list of \(token\_contract, token\_id\) pairs so that the receiving smart contract knows what tokens are being sent to it. For more specialized smart contracts which know it will only receive one type of token, this can be hard coded.

Hypervisor Functions:

* SVC 0x80: alloc\_memory TBD

Context Functions:

* SVC 0x90: gas\_remaining\(\) -&gt; limit:u64 -- Will get the total amount of gas available for the current execution
* SVC 0x91: self\_address\(\) -- result on stack as NeutronAddress -- Will return the current address for the execution. For a "one-time" execution, this will return a null address
* SVC 0x92: origin\(\) -- result on stack as NeutronAddress -- Will return the original address which caused the current chain of executions
* SVC 0x93: origin\_long\(\) -- result on stack as array of bytes
* SVC 0x94: sender\(\) -- result on stack as NeutronAddress -- Will return the address which caused the current execution \(and not the entire chain\)
* SVC 0x95: sender\_long\(\) -- result on stack as array of bytes
* SVC 0x96: execution\_type\(\) -&gt; type:u32 -- The type of the current execution \(see built-in types\)
* SVC 0x97: execution\_permissions\(\) -&gt; permissions:u32 -- The current permissions of the execution \(see built-in types\)

Contract Management Functions:

* SVC 0xA0: upgrade\_code\_section\(id: u8, bytecode: \[u8\], position: u32\):mutable
* SVC 0xA1: upgrade\_data\_section\(id: u8, data: \[u8\], position: u32\):mutable
* SVC 0xA2: upgrades\_allowed\(\): static -&gt; bool
* SVC 0xA4: get\_data\_section\(id: u8, begin, max\_size\) -&gt; data: \[u8\] --there is no code counter type provided because it can be read directly from memory. Data can as well, but may have been modified during execution

System Functions:

* SVC 0xFE: revert\_execution\(status\) -&gt; noreturn -- Will revert the current execution, moving up the chain of execution to return to the previous contract, and reverting all state changes which occured within the current execution
* SVC 0xFF: exit\_execution\(status\) -&gt; noreturn -- Will exit the current execution, moving up the chain of execution to return to the previous contract. State changes will only be committed if the entire above chain of execution also exits without any reverting operations. 

### ARM Memory Map

The ARM memory map is as follows:

* 0x10000, immutable, first code memory
* 0x20000, immutable, second code memory
* ... up to 16 code memories
* 0x80010000, mutable, first data memory
* 0x80020000, mutable, second data memory
* ... up to 16 data memories
* 0x81000000, 8Kb, mutable, stack memory \(for the ARM stack\)
* 0x82000000, ??? size, mutable, aux memory, loaded always as 0 and can be used as an extra RAM area

### Internal State

The data and code of a smart contract are stored separately within NeutronDB. The keys are as so:

* 0x02 00 -- first code section
* 0x02 01 -- second code section
* ... up to 16 code sections
* 0x02 10 -- first data section
* 0x02 11 -- second data section
* ... up to 16 data sections

These memory sections can be conveniently be split up and used by ELF sections in smart contract compilation which can then be parsed by a Neutron tool to remove the complexities of the ELF format, leaving only a list of memory sections with corresponding target addresses.

### Contract Creation

The contract creation ABI is defined using the standard NeutronABI contract creation method using the CoMap to transfer relevant data into the hypervisor. The following fields are defined:

* `!.v` -&gt; Neutron Standard -- ExecutorInfo structure
* `!.c` -&gt; Neutron Standard -- primary code section
* `!.d` -&gt; Neutron Standard -- primary data section
* `!.#extra_code` -&gt; Extra code section count
* `!.@extra_code` -&gt; Extra code sections...
* `!.#extra_data` -&gt; Extra data section count
* `!.@extra_data` -&gt; Extra data sections...

If \#extra\_code and/or \#extra\_data is missing, then it is assumed that there is no such sections

### Initial CPU State

The expected initial state when VM execution begins is all registers and flag values set to 0, excluding EIP being set to 0x10000, where execution will begin, and the following memory areas will be loaded:

* 1st code section
* stack memory accessible
* aux memory accessible

### Loading Memory Areas

Each code section and data section which is attempted to be accessed by the VM will result in loading that section from NeutronDB without any smart contract visible error \(unless the section does not exist\). This will incur a memory size gas cost as well as the gas cost for loading the state from NeutronDB. There is no explicit operation to load or unload memory and it is instead done implicitly by trying to access that memory. The entire memory section will be loaded \(and thus gas fees paid\) upon accessing a single memory location within that section.

Note that in the case of one-time executions, no state will be stored in NeutronDB for the contract, nor will any state be loaded \(except by external contract calls and external state loads\) from NeutronDB. Instead, the entire set of code and data memory data will be stored within the transaction data and loaded into the Hypervisor via the CoMap.

### Constants and Structures

The following constants are defined for different types of execution:

```text
EXECUTION_TYPE_CALL     = 0
EXECUTION_TYPE_DEPLOY   = 1
EXECUTION_TYPE_ONE_TIME = 2
```

* CALL -- A call from a transaction or from an external contract into an existing smart contract
* DEPLOY -- A new smart contract is being deployed
* ONE\_TIME -- A piece of smart contract code is being executed which has not and will not be saved to the blockchain permanently and will not be assigned an account/address 

Execution Permissions can be one or more of the following flags:

* Mutable call -- standard mutable call with no restrictions
* static call -- A static call which can read external contract and otherwise mutable internal data, but can not modify any data or make mutable calls
* Pure call -- a restricted pure call which can only read immutable internal data and can not otherwise access any external data and can not modify any data or make any mutable or static calls.

Defined as so:

```text
EXECUTION_PERMS_MUTABLE     = 1
EXECUTION_PERMS_STATIC      = 2
EXECUTION_PERMS_PURE        = 4
```

All smart contract APIs for checking this, should be a bitwise comparison:

```text
if (permissions & EXECUTION_PERMS_MUTABLE) > 0
    //capable of doing everything a mutable call can
if (permissions & EXECUTION_PERMS_STATIC) > 0
    //capable of doing everything a static call can
if (permissions & EXECUTION_PERMS_PURE) > 0
    //capable of doing everything a pure call can
```

Furthermore, in order to express that an execution is mutable \(which would incldue the permissions for static and pure\) it should be written as so:

```text
permissions = EXECUTION_PERMS_MUTABLE + EXECUTION_PERMS_STATIC + EXECUTION_PERMS_PURE
```

All unspecified bits are reserved for future additions to these permissions. In the case of a permission being created that is even more restrictive than "pure", then none of these flags should be set. In the case of a permission being created which gives more power than "mutable", then all of these flags should be set, along with a new flag conveying this new permission.

### Hypervisor Internal State

All narm internal state has a prefix of `02`. It specifically stores the following state:

* `0200` - `020F` -- code section states
* `0210` - `021F` -- data section states

Note that this state is affected by state rent and is restored via the typical methods and persisted by actually using the code/data sections within a smart contract execution.

