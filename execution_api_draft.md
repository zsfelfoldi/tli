## Trustless execution layer API

This document is a draft proposal for a new trustless execution layer API that uses REST API format similar to the consensus layer API instead of the current JSON RPC. Note that the fully trustless implementation of certain endpoints is very inefficient with the current consensus data structures. [EIP-7745](https://eips.ethereum.org/EIPS/eip-7745) provides an efficient solution for these use cases. It is indicated in the document which endpoints are possible to realize now and which ones depend on the mentioned EIP.

### Data format

Similarly to the consensus layer API, by default all requests send and receive JSON data while some request types might also allow binary (RLP) format. The sent and expected data format should be specified by setting the "Content-Type" and/or "Accept" HTTP headers to "application/json" or "application/octet-stream".

Note that the JSON response format is typically more verbose in order to ensure human readability and easy debugging. The binary version (if available) is intended for higher bandwidth efficiency and therefore only contains the minimal amount of data required to validate the response and reconstruct the queried object. 

Also note that unlike the beacon API that uses SSZ serialization for binary encoding, the execution layer API uses RLP because it is the native format of the execution layer and validating the transmitted data structures against the expected hashes requires hashing them in RLP format anyways. The [EIP-7745](https://eips.ethereum.org/EIPS/eip-7745) lookup index uses SSZ merkleization and [EIP-7919](https://eips.ethereum.org/EIPS/eip-7919) proposes switching more execution layer structures to SSZ merkleization but it is important to note that this is only a tree hashing scheme that is technically unrelated to the SSZ serialization, and RLP serialization is just as suitable for encoding SSZ merkleized data as SSZ serialization. For the time being, in most use cases RLP is a much more logical encoding choice while in a few use cases they are similarly convenient. SSZ serialization might be added later as an option for certain endpoints if there is a good reason to do so. 

#### Response format versioning

Most of the trustless execution layer API requests return responses that encode a part of a protocol defined data structure that can be validated against a root hash that is either explicitly specified in the request or assumed to be known by the client. The content and/or the hashed encoding of these data structures might change between protocol forks. In order to make the API future proof, each response consists of a "version" string field containing the fork name, and a "data" object that should be interpreted and hashed according to the fork in effect. This is also similar to the way the `light_client` endpoints of the consensus layer API work.

### Following the consensus

The trustless execution API works under the general assumption that the consumer has a way of following the consensus and knows the most recent (or at least fairly recent) chain head. The consumer can typically realize this either by running a full beacon node or by also consuming the `/eth/v*/beacon/light_client` endpoints of a consensus layer [Beacon API](https://ethereum.github.io/beacon-APIs/) provider. Both methods provide safe access to the latest head and finalized block (though the optimistic updates of the beacon light client API have a one slot delay).

It is also possible for a provider of the execution API to also implement the beacon light client API and provide the necessary endpoints over the same HTTP URL. Consumers should be prepared though to use different servers for consensus and execution APIs.

#### Identifying blocks by number or hash

The following notations are used throughout this document for block identification:

- `{block number}`: either a block number in decimal format or one of the `genesis`, `finalized` or `head` strings
- `{block hash}`: a block hash in hexadecimal format with `0x` prefix
- `{block id}`: either a `{block hash}` or a `{block number}`

Block identification by hash is automatically safe because everything is hashed into the block header and the block header's hash can be checked against the requested block hash. When identifying a block by number (that is not the head or finalized block) it is necessary to prove that the associated block hash is indeed canonical. One way to do this is by obtaining the header chain between the block in question and the known head/finalized block. This is possible with the `/eth/v1/exec/headers` endpoint but it is impractical for proving older blocks. Another way is to use the `/eth/v1/exec/history` endpoint which requires [EIP-7745](https://eips.ethereum.org/EIPS/eip-7745) and can prove the canonical hash of any historical block with a cca 1000-1500 bytes long proof.

Since there are multiple ways to prove a canonical hash, and also because it might not even be necessary to do every time as it might already be known to the client, the endpoints that allow block identification by number do not return a proof of canonicalness and leave it up to the client to obtain a proof separately when necessary.

### API endpoints

#### /eth/v1/exec/headers/{block id}

Query parameters:

> - __amount__ decimal (number of items requested, default = 1)

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ list of objects (block headers; not detailed here)

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list of list of items (block headers; not detailed here)

`headers` returns a series of consecutive block headers ending with the specified `{block id}`. Note that the server may return less items than the specified __amount__. If the specified `{block id}` is unavailable then an HTTP error is returned (404 for unknown block hashes and future block numbers, 410 for expired block numbers).

#### /eth/v1/exec/blocks/{block id}

Query parameters:

> - __amount__ decimal (number of items requested, default = 1)

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ list of objects (blocks; not detailed here)

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list of list of items (blocks; not detailed here)

`blocks` returns a series of consecutive blocks ending with the specified `{block id}`. Note that the server may return less items than the specified __amount__. If the specified `{block id}` is unavailable then an HTTP error is returned (404 for unknown block hashes and future block numbers, 410 for expired block numbers).

#### /eth/v1/exec/block_receipts/{block id}

Query parameters:

> - __amount__ decimal (number of items requested, default = 1)

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ list of objects (block receipts; not detailed here)

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list of list of items (block receipts; not detailed here)

`block_receipts` returns block receipts belonging to a series of consecutive blocks ending with the specified `{block id}`.  Note that the server may return less items than the specified __amount__. If the specified `{block id}` is unavailable then an HTTP error is returned (404 for unknown block hashes and future block numbers, 410 for expired block numbers).

Also note that the block receipts can be validated against the `receipts_root` fields of the corresponding headers and therefore it is assumed that the client also possesses the headers corresponding to the requested block receipts.

#### /eth/v1/exec/history/{block id}

_Requires EIP-7745_

Query parameters:

> - __ref_head__ hex string (reference head block hash) __*REQUIRED*__
> - __page_indices__ list of decimals (requested pages)
> - __page_period__ decimal (periodical pagination)

Response: __*index multiproof*__ (see below)

`history` proves the _block delimiter_ with the specified block number or hash if present in the canonical chain. Note that only blocks older than `ref_head` have a _block delimiter_ and therefore the reference head itself cannot be queried (it is assumed to be known by client already). For a valid query of a known canonical block (either by number or hash) the proof contains only the single _block delimiter_ in question. For unavailable block numbers an HTTP error is returned (404 for future blocks, 410 for expired). For block hashes not present in the known canonical chain, an exclusion proof is returned (a hash lookup for the entire available block range with zero results).

#### /eth/v1/exec/transaction/{tx hash}

Query parameters: none

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ object (signed transaction data; not detailed here)

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list  of items (signed transaction data; not detailed here)

`transaction` returns the signed transaction with the specified hash if available. If it is unknown then a HTTP error 404 is returned.

Note that a valid response does not prove that the transaction is canonical, neither does a failure prove that the transaction does not exist on the canonical chain.  

#### /eth/v1/exec/transaction_position/{tx hash}

_Requires EIP-7745_

Query parameters:

> - __ref_head__ hex string (reference head block hash) __*REQUIRED*__
> - __page_indices__ list of decimals (requested pages)
> - __page_period__ decimal (periodical pagination)

Response: __*index multiproof*__ (see below)

`transaction_position` proves the _transaction delimiter_ with the specified transaction hash if present in the canonical chain, thereby proving the canonical inclusion position of the transaction. If the transaction is not present in the known canonical chain, an exclusion proof is returned (a hash lookup for the entire available block range with zero results).

#### /eth/v1/exec/transaction_by_index/{block id}

Query parameters:

> - __indices__ list of strings

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ list of objects
>     - __index__ hex string
>     - __transaction__ object (signed transaction data; not detailed here)
>     - __proof__ list of hex strings (RLP trie nodes)

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list of list of items
>     - __index__ integer
>     - __proof__ list of byte arrays (RLP trie nodes)

`transaction_by_index` returns Merkle proofs of the transactions at the positions specified in the __indices__ list from the `transactions` tree of the specified block. The resulting root hash can be validated against the `transactions_root` field of the corresponding block header which needs to be obtained separately if not available. The items in the __indices__ list can either be decimal numbers or the `last` string in which case the last transaction in the tree is proven which also proves the number of transactions in the block since all right side siblings are empty. If the index points to a non-existent entry then the __transaction__ object of the JSON response is empty but an exclusion proof is still returned.

#### /eth/v1/exec/receipt_by_index/{block id}

Query parameters:

> - __indices__ list of strings

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ list of objects
>     - __index__ hex string
>     - __receipt__ object (receipt data; not detailed here)
>     - __proof__ list of hex strings (RLP trie nodes)

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list of list of items
>     - __index__ integer
>     - __proof__ list of byte arrays (RLP trie nodes)

`receipt_by_index` returns Merkle proofs of the receipts at the positions specified in the __indices__ list from the `receipts` tree of the specified block. The resulting root hash can be validated against the `receipts_root` field of the corresponding block header which needs to be obtained separately if not available. The items in the __indices__ list can either be decimal numbers or the `last` string in which case the last transaction in the tree is proven which also proves the number of receipts in the block since all right side siblings are empty. If the index points to a non-existent entry then the __receipt__ object of the JSON response is empty but an exclusion proof is still returned.

#### /eth/v1/exec/logs

_Requires EIP-7745_

Query parameters:

> - __from_block__  decimal (first block of query range)
> - __to_block__  decimal (last block of query range)
> - __addresses__ list of hex strings (addresses allowed)
> - __topics__ list of list of hex strings (topics allowed at each position)
> - __ref_head__ hex string (reference head block hash) __*REQUIRED*__
> - __page_indices__ list of decimals (requested pages)
> - __page_period__ decimal (periodical pagination)
> - __limit__ decimal (max number of results)
> - __reverse__  no value (return most recent results if present)

Response: __*index multiproof*__ (see below)

`logs` returns matches for the given log event search query, along with a proof of the  validity and completeness of the returned results. 

#### /eth/v1/exec/state/{block id}

Query parameters: none

POST request body:

> _Content-Type: application/json_
> 
> - list of objects
>     - __address__ hex string
>     - __getCode__ boolean
>     - __storage__ list of objects
>         - __key__ hex string

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ list of objects
>     - __address__ hex string
>     - __balance__ hex string
>     - __nonce__ hex string
>     - __storageHash__ hex string
>     - __codeHash__ hex string
>     - __accountProof__ list of hex strings (RLP trie nodes)
>     - __code__ hex string (only present if requested)
>     - __storage__ list of objects
>         - __key__ hex string
>         - __value__ hex string
>         - __storageProof__ list of hex strings (RLP trie nodes)

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list of list of items
>     - __accountProof__ list of byte arrays (RLP trie nodes)
>     - __code__ byte array (only present if requested, zero length otherwise)
>     - __storageProofs__ list of list of byte arrays (RLP trie nodes)

`state` returns Merkle proofs for all requested accounts, contract codes and storage values. 

#### /eth/v1/exec/call/{block id}

Query parameters: none

POST request body:

> _Content-Type: application/json_
> 
> - __transaction__ object (not detailed here)
> - __state_overrides__ object (not detailed here)
> - __block_overrides__ object (not detailed here)

Response (JSON):

> _Accept: application/json_
>
> - __version__ string
> - __data__ object
>     - __stateAccessList__ list of objects
>         - __address__ hex string
>         - __preBalance__, __postBalance__ hex string
>         - __preNonce__, __postNonce__ hex string
>         - __storageHash__ hex string
>         - __codeHash__ hex string
>         - __accountProof__ list of hex strings (RLP trie nodes)
>         - __code__ hex string (only present if accessed)
>         - __storage__ list of objects
>             - __key__ hex string
>             - __preValue__, __postValue__ hex string
>             - __storageProof__ list of hex strings (RLP trie nodes)
>     - __blockHashList__ list of objects (block hashes accessed by EVM)
>         - __blockNumber__ hex string
>         - __blockHash__ hex string
>     - __results__ hex string (return value of the executed contract method)
>     - __gasUsed__ hex string

Response (RLP):

> _Accept: application/octet-stream_
> 
> - __version__ byte array
> - __data__ list of items
>     - __stateAccessList__ list of list of items
>         - __address__ byte array
>         - __accountProof__ list of byte arrays (RLP trie nodes)
>         - __code__ byte array (only present if accessed, zero length otherwise)
>         - __storage__ list of list of items
>             - __key__ byte array
>             - __storageProof__ list of byte arrays (RLP trie nodes)
>     - __blockHashList__ list of integers (block numbers of hashes accessed by EVM)

`call` executes the given transaction on the state of the specified block with optional overrides for certain accounts and/or header fields, and returns proofs for all accessed state entries, allowing the client to re-execute the transaction locally.

Note that the more verbose JSON response format contains post values, execution results and gas used while the binary version only contains the data that is required to re-execute the transaction. Local execution is expected to give the same results as the extra JSON fields. A trustless client implementation should perform local execution even if it uses the JSON format, optionally validating the extra fields against the results of the execution for debug/verification purposes.

Also note that the response does not include a proof for the block hashes accessed by the `BLOCKHASH` opcode. These hashes might be available to the client already, also can be obtained in multiple ways (`/eth/v1/exec/history` or `/eth/v1/exec/headers`).

#### Omitting duplicated trie nodes in Merkle proofs

If a response contains multiple Merkle proofs of the same trie (main state trie, storage tries of contracts, transactions and receipts tries) then the first proof should contain all trie nodes, while the subsequent ones should omit the nodes that are the same in the previous proof. Omitted nodes are represented as zero length byte arrays. This optimization should only be applied when a binary encoded response is requested. In the JSON response format all trie nodes should be present to ensure simplicity and human readability.

#### Mapping old JSON RPC requests to trustless REST API

The following table shows how JSON RPC features can be realized trustlessly. In some cases this involves multiple requests and extra logic on the client side. REST API requests relying on [EIP-7745](https://eips.ethereum.org/EIPS/eip-7745) are marked with an asterisk (*). In these cases alternatives are also presented using requests that can be realized with current consensus, though these might only work in a very limited range as they might become prohibitively expensive when applied to older chain history. 

|JSON RPC    |Trustless execution layer REST API   |
|------------|---------------------|
|`eth_blockNumber`|none (assumed to be known by the client)|
|`eth_getBalance`|`/eth/v1/exec/state`|
|`eth_getBlockByHash`|`/eth/v1/exec/blocks`<br>if canonicalness needs to be proven, `/eth/v1/exec/history`*<br> *alternative: `/eth/v1/exec/headers` from queried block to head|
|`eth_getBlockByNumber`|`/eth/v1/exec/blocks`, `/eth/v1/exec/history`*<br> *alternative: `/eth/v1/exec/headers` from queried block to head|
|`eth_getBlockReceipts`|`/eth/v1/exec/block_receipts`|
|`eth_getBlockTransactionCountByHash`|`/eth/v1/exec/transaction_by_index`|
|`eth_getBlockTransactionCountByNumber`|`/eth/v1/exec/history`* or `/eth/v1/exec/headers` from queried block to head<br>`/eth/v1/exec/transaction_by_index`|
|`eth_getCode`|`/eth/v1/exec/state`|
|`eth_getLogs`|`/eth/v1/exec/logs`*<br> *alternative: `/eth/v1/exec/headers` from range start to head, `/eth/v1/exec/block_receipts` for all bloom filter matches|
|`eth_getProof`|`/eth/v1/exec/state`|
|`eth_getStorageAt`|`/eth/v1/exec/state`|
|`eth_getTransactionByBlockHashAndIndex`|`/eth/v1/exec/headers`, `/eth/v1/exec/transaction_by_index`|
|`eth_getTransactionByBlockNumberAndIndex`|`/eth/v1/exec/history`*, `/eth/v1/exec/headers` for queried block, `/eth/v1/exec/transaction_by_index`<br> *alternative: `/eth/v1/exec/headers` from queried block to head|
|`eth_getTransactionByHash`|`/eth/v1/exec/transaction`<br>`/eth/v1/exec/transaction_position`* if proof of canonical inclusion position is needed<br> *alternative for canonical transactions: `/eth/v1/exec/headers` from inclusion block to head, `/eth/v1/exec/transaction_by_index`<br> *alternative for exclusion proof: `/eth/v1/exec/blocks` for entire chain|
|`eth_getTransactionCount`|`/eth/v1/exec/state`|
|`eth_getTransactionReceipt`|`/eth/v1/exec/transaction_position`*, `/eth/v1/exec/receipt_by_index`<br> *alternative for canonical transactions: `/eth/v1/exec/headers` from inclusion block to head, `/eth/v1/exec/transaction_by_index`<br> *alternative for exclusion proof: `/eth/v1/exec/blocks` for entire chain|
|`eth_getUncleCountByBlockHash`|`/eth/v1/exec/headers`|
|`eth_getUncleCountByBlockNumber`|`/eth/v1/exec/history`*, `/eth/v1/exec/headers`<br> *alternative: `/eth/v1/exec/headers` from queried block to head|
|`eth_call`|`/eth/v1/exec/call`, `/eth/v1/exec/history`* for BLOCKHASH opcode<br> *alternative: `/eth/v1/exec/headers` from queried block to head|
|`eth_newBlockFilter`|`/eth/v1/exec/events`|
|`eth_newFilter`|`/eth/v1/exec/events`|
|`eth_sendRawTransaction`|`/eth/v1/exec/send_transaction`|
|`eth_createAccessList`|`/eth/v1/exec/call`, `/eth/v1/exec/history`* for BLOCKHASH opcode<br> *alternative: `/eth/v1/exec/headers` from queried block to head|
|`eth_estimateGas`|`/eth/v1/exec/call`, `/eth/v1/exec/history`* for BLOCKHASH opcode<br> *alternative: `/eth/v1/exec/headers` from queried block to head|
|`eth_gasPrice`|`/eth/v1/exec/headers`, `/eth/v1/exec/blocks` for some recent blocks|
|`eth_feeHistory`|`/eth/v1/exec/headers`, `/eth/v1/exec/blocks`, `/eth/v1/exec/block_receipts` for some recent blocks|
|`eth_maxPriorityFeePerGas`|`/eth/v1/exec/headers`, `/eth/v1/exec/blocks` for some recent blocks|
|`eth_baseFee`|`/eth/v1/exec/headers`|
|`eth_blobBaseFee`|`/eth/v1/exec/headers`|

### Notes on EIP-7745 API endpoints

[EIP-7745](https://eips.ethereum.org/EIPS/eip-7745) adds the root hash of a historical lookup index structure to each block header, allowing efficient trustless proofs of log events, canonical transactions and canonical block hashes. Each API endpoint that is based on this data structure has the same general response format which is a Merkle multiproof proving the relevant parts of this index structure (the exact encoding format of the __*index multiproof*__ response is not specified yet).

The index consists of the following components:
- _log entries_: one SSZ merkleized entry per emitted log event, mapped onto a continuous global linear index space
- _transaction delimiters_: one SSZ merkleized entry per transaction, mapped to the same global index space, placed before the _log entries_ generated by the transaction 
- _block delimiters_: one SSZ merkleized entry per block, mapped to the same global index space, placed after the _log entries_ and _transaction delimiters_ generated by the block 
- _filter maps_: a probabilistic lookup structure that can prove the occurence or abscence of certain log addresses and topics, block hashes and transaction hashes

Each __*index multiproof*__ response should prove a subset of these components defined by the specific request type and parameters.

#### Proof block range vs history expiry

In order to validate the results of an index lookup through _filter maps_, it is necessary to specify the searched range in the linear index space. If the proven range is specified as a block number range then the mapping between block numbers and the linear index space also needs to be proven by proving _block delimiters_. For this purpose, the __*index multiproof*__ contains a `first_block` and `last_block` field. If `first_block` is greater than zero then the block delimiter `first_block - 1` should be proven. If `last_block` is less than the block number of `ref_head` then the block delimiter `last_block` should be proven.

Note that the returned proof range may be limited by the historical data available to the server. Log filter query accepts a requested block range as a parameter while other requests that are never expected to return more than one actual search match (block by hash, transaction by hash) apply to the entire block range by default. All of these ranges can be limited by history expiry though, and this is why the final proof range is specified in the response.

#### Reference head

Since the index structure is updated with each block and has a new root hash embedded in each header that the client can validate against, it is also necessary to specify which version of the index the proof is made of. Because the consumer might know about an older head than the provider (typically one slot older if it uses the beacon light client API) it is the consumer that specifies the _reference head_ (the `ref_head` query parameter) that is the hash of the block header that refers to the version of the index that the Merkle proof in the response will be validated against.

Note that maintaining a few versions of the index Merkle tree is relatively cheap and simple with in-memory diff layers but the API providers should not be expected to maintain this tree for very old reference heads. Therefore a provider should only accept reference heads that are either one of the four most recent blocks or one of the two most recent finalized blocks, and a consumer should specify either the latest known block or the latest finalized block as `ref_head`.

#### Pagination

Some index queries can be expensive to serve depending on the complexity of the query and the number of search results. The optional query parameters `page_indices` and `page_period` provide an efficient way to split the request into multiple parts and have them served either sequentially or in parallel by multiple providers. The binary Merkle tree of the index structure is organized into fixed size _index epochs_ and splitting a Merkle proof at these epoch boundaries is efficient, therefore the results found in each _index epoch_ can be considered as a "page".

If `page_indices` are not specified then results from every epoch are returned. If it is specified then results from epochs where `epoch_index in page_indices` is true are returned. If `page_period` is also specified then results from epochs where `epoch_index % page_period in page_indices` is true are returned.

Note that the cost of the request (both server disk/CPU cost and network bandwidth cost) depends on both the length of the searched range and the number of results found. Requests which may potentially return a large number of results depending on their query parameters also have a `limit` parameter that limits the number of results returned. If this limitation takes effect then the proven block range of the proof is reduced and the client can resume the search in its next request that excludes the already proven range. Also note that the server might also apply its own limit on the results returned in a single response. If the `limit` parameter is applicable then there is also a `reverse` parameter which causes the search to start in reverse direction from the last block of the search range, and return the most recent results in case the limitation is applied.

This API logic also permits combining parallel and sequential requests, for example where the client requests even numbered epochs from one endpoint and odd numbered ones from another, in both cases re-sending the request multiple times with an updated block range until all results are returned.
