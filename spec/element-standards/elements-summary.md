# Elements Summary

In this folder all of the various element standards will be defined. Element Standards shall be prefixed with a number for defined standards. Other standards can be prefixed with `DRAFT` if it is not currently a finalized standard, but strongly favored to be included in this specification. There will later be a separate location for publishing and working on draft Element Standards and including community proposals into this spec as appropriate.

Element Function Modifiers:

These are only used in these specifications and are not necessarily strict.

* :const -- Conveys that it will express data which may change in later blocks or transactions, but will be constant across the entire current chain of execution. Introduces no current execution data dependencies, but does introduce whole-blockchain data dependencies. Thus these can not be used by pure execution contexts. These types of functions are effectively less dynamic than static, but more dynamic than pure. 
* :pure -- Conveys that the function is purely computation and thus given the same input should give the same output at any point in the blockchain history. Can be used by all execution contexts \(short of forks etc for adding new algorithms or bug fixes\)
* :static -- Conveys that the function will not modify or alter potentially upstream data dependencies, but also that this function's results may change through the current chain of execution. In other words, it may read execution-mutable state, but not modify it. This can not be used by pure execution contexts
* :mutable -- Conveys that the function will modify data which may be relied upon in other places in the execution history. In other words, it may both read and write to execution-mutable state. This can not be used by pure nor static execution contexts. 
* :comap -- Conveys that the comap must be used for this function
* :comap\_optional -- Conveys that additional features are available by using the comap for this function

Every Element API must implement the following function:

* 0, `function_exists(function_id: u32) -> version: u32`

If version in this case is 0, then the function is not implemented. Any other value means it is implemented. Numbers other than 1 may be used to indicate additional information that is implementation-defined. This function shall be used rather than speculative execution \(ie, pushing inputs and then checking if the error indicates the function did not exist\) to save on gas costs.

Calling any `function_exists` Element API function shall be a free function in terms of gas costs. 

Calling function\_exists on an element which is not supported will not return a 0 version, but rather give an error which will indicate the element is not supported. Usage of the API for this purpose to detect supported Elements shall also be gas-free. Platforms may choose to have a `function_exists` that returns 0 result in an UnrecoverableError. This would allow for functions to be more easily added in forks, but reduces the flexibility of smart contract code. 

Hypervisors may choose to provide a "native" call to use the `function_exists` function which would allow for complete avoidance of using the CoStack. 

Standard Tracts:

* Neutron Standard -- These are standard Neutron functions which should not rely on any particular blockchain or platform design, though most are not mandatory
* Neutron Account Tract -- These are Neutron functions which rely on an account based blockchain model, or an account abstracted model being available
* Neutron UTXO Tract -- These are Neutron functions which rely on a UTXO based blockchain model

Provisions:

* Mandatory -- All implementations of Neutron must implement these functions if the specified standard ElementAPI is supported
* Recommended -- This is recommended functionality to support if possible and typically involves minimal additional requirements over the mandatory set of functions
* Optional -- These tend to depend on platform design or may be more involved or with less usefulness, and thus may be excluded from some platform integrations of Neutron

