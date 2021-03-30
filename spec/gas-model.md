# Gas Model

The gas interface into and out of Neutron is specified as being "x 100". Thus, 100 gas is the minimum amount that can be specified. This greatly increases the granularity available in the gas model. When outputing gas from Neutron, it should thus be divided by 100, rounding up, to get the actual gas cost on the blockchain.

The Neutron gas model will involve two separate metering points:

1. VM execution metering
2. Element execution metering

These will be controlled by a two tables of "gas schedules". The element gas schedule is as so:

```text
(element) -> (operation) -> (cost_parameter, Optional<computation_function>)
```

Due to the intense computational workload that can come from checking for optional functions within a VM loop, the VM gas metering is greatly simplified:

```text
(operation) -> (cost_parameter)
```

Operations cover shared costs between VMs, though it is expected that some operations could be specific to a particular VM type. It is expected that each VM will copy relevant costs into their own local memory upon initialization as appropriate to avoid more expensive table lookups in the middle of VM loops.

Standard VM operation costs:

* GAS\_ALGORITHM\_VERSION -- note: not an actual cost, used to easily include fork behaviors
* INSTRUCTION\_BASE\_COST
* VERY\_LOW\_COST\_OPCODE -- note: may be 0 cost
* LOW\_COST\_OPCODE
* MEDIUM\_COST\_OPCODE
* HIGH\_COST\_OPCODE
* VERY\_HIGH\_COST\_OPCODE
* PREDICTABLE\_BRANCH\_OPCODE
* UNPREDICTABLE\_BRANCH\_OPCODE
* COPY\_INTO\_VM\_MEMORY -- charge per byte copied into VM memory
* COPY\_FROM\_VM\_MEMORY -- charge per byte copied from VM memory
* MEMORY\_READ\_COST -- charge per byte of memory read within the VM
* MEMORY\_WRITE\_COST -- charge per byte of memory written within the VM
* VM\_MUTABLE\_MEMORY\_ADDED -- charge per byte of VM memory allocated which can not be optimized away easily
* VM\_READONLY\_MEMORY\_ADDED -- charge per byte of VM memory added which can potentially be optimized away due to being read-only
* VM\_MEMORY\_PRESSURE -- Neutron supplied function. Applies gas pressure to prevent too much memory allocation
* VM\_MEMORY\_PRESSURE\_THRESHOLD -- Neutron supplied function. Specifies when pressure should start to be applied to an allocation
* CODATA\_PUSH\_BASE\_COST -- base cost per push. Note this is not used by VMs directly, but rather only by hypervisors
* CODATA\_PRESSURE\_THRESHOLD -- determination of when the costack and/or comap should begin exerting "pressure" costs
* CODATA\_PRESSURE -- amount of gas pressure to exert \(could potentially represent a curve parameter etc

Additional costs per VM could be for things like complex Mod R/M decoding in x86, or using conditional execution in ARM.

### Communication of Gas

Element gas is controlled and updated through the holder of the Codata

