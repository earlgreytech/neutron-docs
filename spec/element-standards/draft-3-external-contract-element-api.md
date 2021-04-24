# DRAFT - 3 - External Contract Element API

This element provides methods of interacting with external smart contracts stored on the platform.

ABI Information:

* Element ID: 3 
* Type: Neutron Standard 
* Provision: Recommended if underlying blockchain has the capability

Standard Functions:

* 1, call\_contract\(address: NeutronShortAddress, gas\_limit: u64, inputs: ...\):comap, mutable -&gt; \(error\_code:u32, outputs: ...\)
* 2, static\_call\_contract\(address: NeutronShortAddress, gas\_limit: u64, inputs: ...\):comap, static -&gt; \(error\_code:u32, outputs: ...\)
* 3, pure\_call\_contract\(address: NeutronShortAddress, gas\_limit: u64, inputs: ...\):comap, pure -&gt; \(error\_code:u32, outputs: ...\)
* 4, get\_self\_address\_of\_deployer\(\):pure -&gt; address: NeutronShortAddress
* 5, get\_address\_of\_deployer\(address: NeutronShortAddress\):static -&gt; address: NeutronShortAddress
* 6, get\_self\_initial\_deploy\_hash\(\):pure -&gt; hash: u256
* 7, get\_initial\_deploy\_hash\(address: UniversalShortAddress\):static -&gt; hash: u256
* 8, get\_self\_contract\_flags\(\):pure -&gt; flags: u8
* 9, get\_contract\_flags\(address: UniversalShortAddress\):static -&gt; flags: u8
* 10, get\_self\_upgrade\_count\(\):const -&gt; count:u32
* 11, get\_upgrade\_count\(address: UniversalShortAddress\):static -&gt; count:u32

Optional Functions:

* 16, call\_contract\_with\_value\(address: NeutronShortAddress, gas\_limit: u64, value: u64, inputs: ...\):comap, mutable -&gt; \(error\_code:u32, outputs: ...\)

### Details

#### Types of calls

A static call is one in which no state can be written to the blockchain. A pure call is one in which no additional state can be loaded beyond the contract's internal bytecode state needed for execution. A pure call could eventually be optimized in certain ways as it would have no state dependencies and thus may result in discounted gas fees.

A static call execution strictly comes with the following restrictions:

* Can only make use of static, pure, and const Element API Functions

A pure call execution comes with the following restrictions:

* Can only make use of pure Element API Functions

