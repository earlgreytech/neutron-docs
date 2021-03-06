# 1 - Blockchain Info Element

This element provides basic generic information about the current blockchain the smart contract is executing on.

ABI Information

* Element ID: 1 
* Type: Neutron Standard 
* Provision: Recommended

Standard Functions:

* 1, block\_creator\(\):const -&gt; NeutronShortAddress
* 2, block\_creator\_long\(\):const -&gt; NeutronFullAddress
* 3, last\_difficulty\(\):const -&gt; u64
* 4, block\_gas\_limit\(\):const -&gt; u64
* 5, block\_height\(\):const -&gt; u64
* 6, block\_time\(\):const -&gt; u64
* 7, blockchain\_info\(\):const -&gt; struct
* 8, blockchain\_vendor\(\):pure -&gt; u32
* 9, blockchain\_id\(\):pure -&gt; u32

### Details

#### Block Creator

This is the principle address which can be said to be responsible for creating the current block. If there is no direct creator of a block, or if there are multiple co-creators, then this may return a null address.

#### Difficulty

The exact meaning of this is completely blockchain independent. The only constraint is that a higher number returned should mean a higher difficulty level or higher amount of competition in block creation

#### Gas Limit

#### Block Height

#### Block Time

#### Blockchain Info

The blockchain info returned is a structure which conveys the following information \(TBD\)

* Consensus mechanism -- PoW, PoS, DPoS, PoA, other
* Expected block time, in seconds \(0 for sub-second or irregular blocks\)
* UTXO or Accounting based
* Uses rent based state

#### Blockchain Vendor and ID

The blockchain vendor and ID are methods of uniquely identifying a blockchain within the smart contract environment. Vendor in this case should be more like the organization or type of chain. For instance, Ethereum and a private version of Ethereum should share the same vendor ID, but have different blockchain IDs.

These are self selected IDs and there is no standard other than the pre-established ones within this document and that a best effort should be applied to prevent collisions.

Vendor:

* 1, Qtum
* 2, Ethereum
* 3, Generic Bitcoin Based

Blockchain IDs for Qtum

* 1, Qtum Mainnet
* 2, Qtum Primary Testnet
* 3, Qtum Regtest
* 4, Qtum Neutron Testnet

