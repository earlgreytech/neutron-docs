# NeutronAddress and Deployment State

### NeutronAddress Structure

A NeutronAddress is a structure used throughout Neutron as a way of uniquely identifying smart contracts and platform assets. It is of the following structure:

```text
//total size: 36 bytes
pub struct NeutronAddress{
    pub type_info: u32,
    pub data: [u8; 32]
}
```

The `type_info` field is broken into the following structure \(from top bit to bottom\) :

* Reserved: 3 bits \(must be 0\)
* Platform Defined: 5 bits
* Address Target: 8 bits
* Platform Type: 16 bits

This version number can be represented like so:

```text
pub struct AddressType{
    pub reserved: u8,
    pub platform_defined: u8,
    pub target: u8,
    pub platform_type: u16
}
fn unpack_address_type(type_info: u32) -> AddressType{
    AddressType{
        reserved: (info & 0b1110_0000_0000_0000_0000_0000_0000_0000) >> 29,
        platform_defined: (info & 0b0001_1111_0000_0000_0000_0000_0000_0000) >> 24,
        target: (info & 0b0000_0000_1111_1111_0000_0000_0000_0000) >> 16,
        platform_type: (info & 0b0000_0000_0000_0000_1111_1111_1111_1111
    }
}
```

The `reserved` field shall always be 0. It may be given additional meaning in the future. 

The `target` value has meaning inside it as well, the top 2 bits are used to mean the following:

* 00 -- Neutron standard VM
* 01 -- Platform specific VM
* 11 -- Platform specific target without VM functionality
* 10 -- reserved

`target` shall be one of the following values which has the top 2 bits cleared, indicating a standard Neutron VM:

* 0 -- Reserved
* 1 -- Reserved
* 2 -- ARM
* 3 -- Neutron EVM \(reserved\)
* 4 -- WASM \(reserved\)
* 5 -- x86 \(reserved\)
* 6 -- RISC-V \(reserved\)

Currently, there is the following platform specific non-VM targets proposed as part of the standard:

* 192 -- Reserved
* 193 -- pay-to-pubkeyhash
* 194 -- pay-to-scripthash
* 195 -- pay-to-withness-pubkeyhash
* 196 -- pay-to-witness-scripthash

Currently, there is the following platform specific VM targets proposed as part of the standard:

* 64 -- Reserved 
* 65 -- Native EVM \(for interfacing with non-Neutron EVMs\)

Note that proposed platform specific standards and constants here are recommended for use when needed, but are not compulsory. 

The `platform_type` field has no explicit meaning within Neutron, and is completely platform-defined, however it is recommended that the top bit of the value is set for addresses that are intended for a testnet platforms, and the top bit be cleared for production platforms. When applicable, it is preferable for a platform to match Ethereum's CHAINID registry and/or avoid conflicts within it. 

In the case of the `type_info` field being 0, the address is considered a null address. Attempting to use such an address must result in a recoverable error.

### NeutronAddress Generation

NeutronAddresses can be made by four mechanisms:

* Platform native translation \(for native platform address\)
* Deployed directly from the platform \(such as a user sending a contract creating transaction to the blockchain\)
* Deployed from an existing smart contract
* Deployed as a clone of an existing smart contract

The first mechanism is outside of Neutron's control and thus not covered here. This is for the usage of, for instance, pay-to-scripthash bitcoin addresses to be given to a smart contract in some way.

Platform native addresses shall be unique and Neutron Smart contract addresses generated must be unique. In some cases it may be desirable for an address to be predictable before actual deployment however. It must not be possible for two different users deploying a Neutron Smart Contract from a platform to predict the same address, however. 

It is possible for platforms to fully control the address generation for platform deployed smart contracts. However, there may also be methods for Neutron to assist in unique address generation. The following two address generations are defined for platform deployed smart contracts. Each method involves the concatenating the pieces of data together and then using the SHA3 algorithm to form a 256bit cryptographic hash.

User Predictable / Public Non-Predictable Method:

* Deployer address \(as a NeutronAddress\)
* Unique Identifier
* Hash of contract deployment state \(constructor arguments in sorted order, including bytecode\)

The following algorithms are used for deploying from existing smart contracts:

New Deployment From Existing Method:

* Origin address \(address which is causing execution of the current smart contract chain\)
* Deployer contract address
* Hash of new contract deployment state
* Unique Identifier \(as a 256 bit value, like in the user predictable method\)

New deployment from clone method:

* Origin address
* Deployer contract address
* Unique Identifier

The unique identifier must be completely unique to each contract execution and preferably not possible to predict without the transaction signer's private key. An algorithm for a bitcoin-style blockchain like Qtum could look like so:

```text
counter = 0
foreach contract_execution
    uniqueid = hash(txid + vout_number + counter)
    execute_contract()
    counter += 1
end foreach
```

In all of these algorithms it must always be possible to use the platform provided Unique Identifier value. However, there may also be functionality exposed for a user generated "salt" value to change the unique and fairly unpredictable ID into something which can be predicted by users sharing this salt value as desired. Attempting to deploy a smart contract to an address which has already been established must always be an unrecoverable error. Attempting to make use of an address which has not been established must always be a recoverable or unrecoverable error error. 

Address uniqueness is determined solely by the data of the address, and not the type info. All database entries etc must ignore the type info for database key generation, and in certain cases it may be reasonable for smart contract code to only keep track of an address's data and discard the type info, however this is not recommended. This type info is required for many operations including token transfers and contract execution, and platform native addresses do not hold the same uniqueness guarantees that internal NeutronAddress generation gives. 

### Contract Deployment And Status State

Whatever method by which Global Storage is implemented, must allow for permanent \(ie, not affected by rent\) fields to be stored and accessed internally which indicate the overall characteristics and status of each smart contract deployed to the blockchain. This is standardized for the entire NeutronSystem and each hypervisor/VM must follow this standard. Specifically, any state with the prefix of `00` must be Neutron standardized data. Hypervisors should use a different prefix for internal data, bytecode, etc.

The state at key `00 00` will contain the ContractInfo structure.

The ContractInfo field will contain the following information:

* `ExecutorInfo: u64` -- Consists of info for which VM to use, execution flags, etc.
* `InitialDeploymentHash: u256` -- this is a hash of the original constructor arguments and other data used to deploy the contract. This value is not changed by bytecode upgrades. Immutable
* `Creator: NeutronAddress` -- This is the address which originally caused the contract to be created. Can be either a platform native address or Neutron smart contract address. Immutable
* `UpgradeCount: u32` -- tracks the number of time bytecode/data upgrades have occured within the contract.

The `ExecutionInfo` field is composed as a structure like so:

```text
pub struct ExecutorInfo{
    pub vm_target: u8, //same as the target field in AddressType with top bit set to 0
    pub vm_version: u8,
    pub vm_extra: u16
    pub flags: u16
    pub reserved: u16
}
```

The `vm_version` field specifies which VM version was used when this contract was deployed. If the top bit is set to 1, then this specific VM version will be used for future executions \(if supported by the platform, otherwise an error must be generated preventing deployment\). Otherwise, the latest version of the VM will be used. When this is stored into the database, if the top bit is not 1, then Neutron may update the `vm_version` to match the latest version of the specified `vm_target`

The `vm_extra` field is dependent on each VM/Hypervisor to define and is currently reserved with no immediate use.

The `flags` field has the current proposed meanings, but at this point is only a reserved field:

* Upgradeable -- Can update it's own deployed bytecode
* Stateless -- The contract can store no global storage state, aside from it's own bytecode. Notably it can read external smart contract state and cause mutable side effects in other smart contracts by calling them
* PureContract -- Every execution of this smart contract should be assumed to be pure, with no side effects. \(requires Upgradeable, Stateless, and NonPayable flag\)
* NonPayable -- This smart contract should never be capable of holding coins

The `UpgradeCount` field tracks the number of times that the immutable data of the smart contract has been modified. This can include immutable data, or to change internal metadata such as to change the flags in the ContractInfo field. The UpgradeCount must not be possible to modify directly by smart contract code. 

Note it is possible to modify `ExecutorInfo` field after a smart contract has been deployed by an upgrade. This allows for a single address to be used even when a smart contract undertakes a radical upgrade such as using a different VM for bytecode execution.

The InitialDeploymentHash uses the NeutronABI "flat" variant for constructing the hash. Specifically, it takes the entire CoMap \(sorted byte-wise from 0 as first to 255 as highest\) at the time of smart contract construction, converts it into NeutronABI Flat variant data, and then hashes the resulting data. Thus, it is essential for consistency to ensure that extra items are not on the CoMap when a smart contract is created, as this will also be included into this hash.

