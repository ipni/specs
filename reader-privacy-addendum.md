# Reader Privacy Implementation Details

This addendum goes through Reader Privacy protocol, hashing and cryptography choices as well as explains some less obvious details 
for the implementations to consider.

## Hashing

IPNI expects SHA256 as a hashing function. To avoid collisions between the hashes calculated from the same value but for different purposes (for example
to derive an encryption key versus derive a lookup key) a constant string can be appended to the value before calculating a hash over it. For example
`sha256("AESGCM" + multihash)` to derive an encryption key and `sha256("CR_DOUBLEHASH" + multihash)` to calculate a second hash over the multihash.

All hashed data that is used for lookups must be of `Multihash` format with `SHA_256` codec. Double hashed data must use `DBL_SHA_256` codec.

Multihashes must be prepended with `DBL_SHA_256` before calculating a second hash. Unhashed data must be prepended with `SHA_256` before calculating the first hash.

## Encryption

IPNI uses AES in GCM mode for encryption. AES keys must be 32-bytes long and derived from the passpharse by calculating a SHA256 over it. AES keys must be prepended with `AESGCM` before hashing.
One particular detail - is a careful choice of 12-byte IV. IPNI expects an explicit instruction to delete a record (comparing to the DHT where records expire).
Hence the IPNI server needs to be able to compare encrypted values without having to decrypt them as that would require a key that it is unaware of.
That means that the IV needs to be deterministically chosen so that `enc(IV, passphrase, payload)` produces the same output for the same 
passpharase + payload pair. One strategy could be to deterministically derive an IV from the passphrase or to generate it randomly and store 
the mapping on the client side. The IPNI specification doesn't enforce how an IV should be chosen and leaves that up to the client to decide. 
Encrypted payload must have algorithm and nonce encoded in it: `algorithm || nonce || enc(payload)`, where `algorithm` is X bytes (TODO: define) and `nonce` is 12 bytes.

## Data Formats

* All binary data must be b58 encoded when transferred over the wire. 
* `ProviderRecordKey`s must be created by concatenating binary `PeerID` with binary `ContextID`. There is no need for extra length / separators as they are
already encoded as a part of the `Multihash` format.