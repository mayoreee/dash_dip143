<pre>
DIP: 143
Title: Transaction value signing analogous to BIP143 as implemented in Bitcoin Cash 
Authors: greatwolf, mayoree
Status: Draft
Layer: Consensus (hard fork)
Created: 2021-07-03
License: MIT License
</pre>

# Abstract

This DIP describes a digest algorithm that implements the signature covers value when signing Dash transactions. It opens the path for more efficient signing of Dash transactions on hardware wallets.

The proposed digest algorithm is adapted from BIP143[[1]](#bip143) as it minimizes redundant data hashing in verification, covers the input value by the signature and is already implemented in a wide variety of applications[[2]](#bip143Motivation).

# Specification

The proposed digest algorithm computes the double SHA256 of the serialization of:
1. nVersion of the transaction (2-byte uint16_t)
2. hashPrevouts (32-byte hash)
3. hashSequence (4-byte hash)
4. outpoint (32-byte hash + 4-byte index)
5. scriptCode of the input (serialized as pk_script inside CTxOuts)
6. value of the output spent by this input (8-byte int64_t)
7. nSequence of the input (8-byte int64_t)
8. hashOutputs (32-byte hash)
9. nLockTime of the transaction (4-byte uint32_t) 
10. sighash type of the signature (4-byte uint32_t) 

#### nVersion

* This is the transaction number; currently version `3`. 

#### hashPrevouts

* If the `ANYONECANPAY` flag is not set, `hashPrevouts` is the double SHA256 of the serialization of all input `outpoints`;
* Otherwise, `hashPrevouts` is a `uint256` of `0x0000......0000`.

#### hashSequence

* If none of the `ANYONECANPAY`, `SINGLE`, `NONE` sighash type is set, `hashSequence` is the double SHA256 of the serialization of `nSequence` of all inputs;
* Otherwise, `hashSequence` is a `uint256` of `0x0000......0000`.

#### outpoint

* Single transactions can include multiple outputs.
* The `outpoint` structure includes both a `TXID` and an output `index` number to refer to specific output.

#### scriptCode

* If the `script` does not contain any `OP_CODESEPARATOR`, the `scriptCode` is the `script` serialized as scripts inside `CTxOut`.
* If the `script` contains any `OP_CODESEPARATOR`, the `scriptCode` is the `script` but removing everything up to and including the last executed `OP_CODESEPARATOR` before the signature checking opcode being executed, serialized as scripts inside CTxOut.

#### value

* The 8-byte `value` of the `amount` of `duffs` the input contains.

#### nSequence

* This is the `sequence` number. 
* Default is `0xffffffff`.

#### hashOutputs

* If the sighash type is neither `SINGLE` nor `NONE`, `hashOutputs` is the double SHA256 of the serialization of all output `values` (8-byte int64_t) paired up with their `scriptPubKey` (serialized as scripts inside `CTxOuts`);
* If sighash type is `SINGLE` and the input `index` is smaller than the number of outputs, `hashOutputs` is the double SHA256 of the output `amount` with `scriptPubKey` of the same `index` as the input;
* Otherwise, `hashOutputs` is a `uint256` of `0x0000......0000`.

#### nLockTime

* Time (Unix epoch time) or block number.

#### sighash type

````cpp
  ss << nHashType;
````

# Implementation

Addition to `SignatureHash` :

```cpp
  uint256 hashPrevouts;
  uint256 hashSequence;
  uint256 hashOutputs;
  
  if (!(nHashType & SIGHASH_ANYONECANPAY)) {
      hashPrevouts = GetPrevoutHash(txTo);
  }
  
  if (!(nHashType & SIGHASH_ANYONECANPAY) && (nHashType & 0x1f) != SIGHASH_SINGLE && (nHashType & 0x1f) != SIGHASH_NONE) {
       hashSequence = GetSequenceHash(txTo);
  }
  
  if ((nHashType & 0x1f) != SIGHASH_SINGLE && (nHashType & 0x1f) != SIGHASH_NONE) {
       hashOutputs = GetOutputsHash(txTo);
  } else if ((nHashType & 0x1f) == SIGHASH_SINGLE && nIn < txTo.vout.size()) {
     CHashWriter ss(SER_GETHASH, 0);
      ss << txTo.vout[nIn];
      hashOutputs = ss.GetHash();
  }
  
  CHashWriter ss(SER_GETHASH, 0);
  // Version
  ss << txTo.nVersion;
  // Input prevouts/nSequence (none/all, depending on flags)
  ss << hashPrevouts;
  ss << hashSequence;
  // The input being signed (replacing the scriptSig with scriptCode + amount)
  // The prevout may already be contained in hashPrevout, and the nSequence
  // may already be contain in hashSequence.
  ss << txTo.vin[nIn].prevout;
  ss << static_cast<const CScriptBase&>(scriptCode);
  ss << amount;
  ss << txTo.vin[nIn].nSequence;
  // Outputs (none/one/all, depending on flags)
  ss << hashOutputs;
  // Locktime
  ss << txTo.nLockTime;
  // Sighash type
  ss << nHashType;
  
  return ss.GetHash();
````

Computation of midstates:

````cpp
uint256 GetPrevoutHash(const CTransaction &txTo) {
  CHashWriter ss(SER_GETHASH, 0);
  for (unsigned int n = 0; n < txTo.vin.size(); n++) {
    ss << txTo.vin[n].prevout;
  }

  return ss.GetHash();
}

uint256 GetSequenceHash(const CTransaction &txTo) {
  CHashWriter ss(SER_GETHASH, 0);
  for (unsigned int n = 0; n < txTo.vin.size(); n++) {
    ss << txTo.vin[n].nSequence;
  }

  return ss.GetHash();
}

uint256 GetOutputsHash(const CTransaction &txTo) {
  CHashWriter ss(SER_GETHASH, 0);
  for (unsigned int n = 0; n < txTo.vout.size(); n++) {
    ss << txTo.vout[n];
  }

  return ss.GetHash();
}
````

## References

<a name="bip143">[1]</a> https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki

<a name="bip143Motivation">[2]</a> https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki#Motivation



