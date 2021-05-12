# libp2p Kademlia DHT specification

| Lifecycle Stage | Maturity       | Status | Latest Revision |
|-----------------|----------------|--------|-----------------|
| 3A              | Recommendation | Active | r0, 2021-05-07  |

Authors: [@raulk], [@jhiesey], [@mxinden]
Interest Group:

[@raulk]: https://github.com/raulk
[@jhiesey]: https://github.com/jhiesey
[@mxinden]: https://github.com/mxinden

See the [lifecycle document][lifecycle-spec] for context about maturity level
and spec status.

[lifecycle-spec]: https://github.com/libp2p/specs/blob/master/00-framework-01-spec-lifecycle.md

---

## Overview

The Kademlia Distributed Hash Table (DHT) subsystem in libp2p is a DHT
implementation largely based on the Kademlia [0] whitepaper, augmented with
notions from S/Kademlia [1], Coral [2] and mainlineDHT \[3\].

This specification assumes the reader has prior knowledge of those systems. So
rather than explaining DHT mechanics from scratch, we focus on differential
areas:

1. Specialisations and peculiarities of the libp2p implementation.
2. Actual wire messages.
3. Other algorithmic or non-standard behaviours worth pointing out.

For everything else that isn't explicitly stated herein, it is safe to assume
behaviour similar to Kademlia-based libraries.

Code snippets use a Go-like syntax.

## Definitions

### Replication parameter (`k`)

The amount of replication is governed by the replication parameter `k`. The
default value for `k` is 20.

### Kademlia routing table

The data structure backing this system is a k-bucket routing table, closely
following the design outlined in the Kademlia paper [0]. The bucket size is
equal to the replication paramter `k`, and the maximum bucket count matches the
size of the SHA256 function, i.e. 256 buckets.

The routing table is unfolded lazily, starting with a single bucket at position
0 (representing the most distant peers), and splitting it subsequently as closer
peers are found, and the capacity of the nearmost bucket is exceeded.

### Alpha concurrency parameter (`α`)

The concurrency of node and value lookups are limited by parameter `α`, with a
default value of 3. This implies that each lookup process can perform no more
than 3 inflight requests, at any given time.

### Distance

In all cases, the distance between two keys is `XOR(sha256(key1),
sha256(key2))`.

## DHT operations

The libp2p Kademlia DHT offers the following types of routing operations:

- **Peer routing** - _Finding_ the closest nodes to a given key (`FIND_NODE`).
- **Value routing** - _Putting_ a value to the nodes closest to the value's key
  (`PUT_VALUE`) and _getting_ a value by its key from the nodes closest to that
  key (`GET_VALUE`).
- **Content routing** - _Adding_ oneself to the list of providers for a given
  key at the nodes closest to that key (`ADD_PROVIDER`) and _getting_ providers
  for a given key from the nodes closest to that key (`GET_PROVIDERS`).

In addition the libp2p Kademlia DHT offers the auxiliary _bootstrap_ operation.

### Peer routing

The below is one possible algorithm to find nodes closest to a given key on
the DHT. Implementations may diverge from this base algorithm as long as they
continue to adhere to the wire format.

Let's assume we’re looking for nodes closest to key `Key`. We then enter an
iterative network search.

We keep track of:

* the set of peers we've already queried (`Pq`) and the set of next query
  candidates sorted by distance from `Key` in ascending order (`Pn`).

**Initialization**: seed `Pn` with the `α` peers from our routing table we know
are closest to `Key`, based on the XOR distance function.

**Then we loop:**

1. > The lookup terminates when the initiator has queried and gotten responses
   from the k (see [#replication-parameter-k]) closest nodes it has seen.

   (See Kademlia paper [0].)

   The lookup might terminate early in case the local node queried all known
   nodes, with the number of nodes being smaller than `k`.
2. Pick as many peers from the candidate peers (`Pn`) as the `α` concurrency
   factor allows. Send each a `FIND_NODE(Key)` request, and mark it as _queried_
   in `Pq`.
3. Upon a response:
	2. If successful the response will contain the `k` closest nodes the peer
       knows to the key `Key`. Add them to the candidate list `Pn`, except for
       those that have already been queried.
	3. If an error or timeout occurs, discard it.
4. Go to 1.

### Value routing

Value routing can be used both to (1) _put_ and _get_ arbitrary values and (2)
to _put_ and _get_ the public keys of nodes. Node public keys are stored in
records under the `/pk` namespace. That is, the entry `/pk/<peerID>` will store
the public key of peer `peerID`.

DHT implementations may optimise public key lookups by providing a
`GetPublicKey(peer.ID) (ci.PubKey)` method, that, for example, first checks if
the key exists in the local peerstore.

The lookup for public key values is identical to the lookup of arbitrary values,
except that a custom value validation strategy is applied. It checks that
equality `SHA256(value) == peerID` stands when:

1. Receiving a response from a `GET_VALUE` lookup.
2. Storing a public key in the DHT via `PUT_VALUE`.

The record is rejected if the validation fails.

#### Putting values

To _put_ a value the DHT finds the `k` closest peers to the key of the value
using the `FIND_NODE` RPC (see [peer routing section](#peer-routing)), and then sends
a `PUT_VALUE` RPC message with the record value to each of the `k` peers.

#### Getting values

When _gettting_ a value in the DHT, the implementor should collect at least `Q`
(quorum) responses from distinct nodes to check for consistency before returning
an answer.

Should the responses be different, the implementation should use some validation
mechanism to resolve the conflict and select the _best_ result.

**Entry correction.** Nodes that returned _worse_ records are updated via a
direct `PUT_VALUE` RPC call when the lookup completes. Thus the DHT network
eventually converges to the best value for each record, as a result of nodes
collaborating with one another.

The below is one possible algorithm to lookup a value on the DHT.
Implementations may diverge from this base algorithm as long as they continue to
adhere to the wire format.

Let's assume we’re looking for key `Key`. We first try to fetch the value from the
local store. If found, and `Q == { 0, 1 }`, the search is complete.

Otherwise, the local result counts for one towards the search of `Q` values. We
then enter an iterative network search.

We keep track of:

* the number of values we've fetched (`cnt`).
* the best value we've found (`best`), and which peers returned it (`Pb`)
* the set of peers we've already queried (`Pq`) and the set of next query
  candidates sorted by distance from `Key` in ascending order (`Pn`).
* the set of peers with outdated values (`Po`).

**Initialization**: seed `Pn` with the `α` peers from our routing table we know
are closest to `Key`, based on the XOR distance function.

**Then we loop:**

1. If we have collected `Q` or more answers, we cancel outstanding requests,
   return `best`, and we notify the peers holding an outdated value (`Po`) of
   the best value we discovered, by sending `PUT_VALUE(Key, best)` messages.
   _Return._
2. Pick as many peers from the candidate peers (`Pn`) as the `α` concurrency
   factor allows. Send each a `GET_VALUE(Key)` request, and mark it as _queried_
   in `Pq`.
3. Upon a response:
	1. If successful, and we receive a value:
		1. If this is the first value we've seen, we store it in `best`, along
           with the peer who sent it in `Pb`.
		2. Otherwise, we resolve the conflict by e.g. calling
           `Validator.Select(best, new)`:
			1. If the new value wins, store it in `best`, and mark all formerly
               “best" peers (`Pb`) as _outdated peers_ (`Po`). The current peer
               becomes the new best peer (`Pb`).
			2. If the new value loses, we add the current peer to `Po`.
	2. If successful without a value, the response will contain the closest
       nodes the peer knows to the key `Key`. Add them to the candidate list `Pn`,
       except for those that have already been queried.
	3. If an error or timeout occurs, discard it.
4. Go to 1.

#### Entry validation

Implementations should validate DHT entries during retrieval and before storage
e.g. by allowing to supply a record `Validator` when constructing a DHT node.
Below is a sample interface of such a `Validator`:

``` go
// Validator is an interface that should be implemented by record
// validators.
type Validator interface {
	// Validate validates the given record, returning an error if it's
	// invalid (e.g., expired, signed by the wrong key, etc.).
	Validate(key string, value []byte) error

	// Select selects the best record from the set of records (e.g., the
	// newest).
	//
	// Decisions made by select should be stable.
	Select(key string, values [][]byte) (int, error)
}
```

`Validate()` is a pure function that reports the validity of a record. It may
validate a cryptographic signature, or else. It is called on two occasions:

1. To validate values retrieved in a `GET_VALUE` query.
2. To validate values received in a `PUT_VALUE` query before storing them in the
   local data store.

Similarly, `Select()` is a pure function that returns the best record out of 2
or more candidates. It may use a sequence number, a timestamp, or other
heuristic of the value to make the decision.

### Content routing

Nodes must keep track of which nodes advertise that they provide a given key
(CID). These provider advertisements should expire, by default, after 24 hours.
These records are managed through the `ADD_PROVIDER` and `GET_PROVIDERS`
messages.

When the local node wants to indicate that it provides the value for a given
key, the DHT finds the closest peers to `key` using the `FIND_NODE` RPC (see
[peer routing section](#peer-routing)), and then sends a `ADD_PROVIDER` RPC with
its own `PeerInfo` to each of these peers.

Each peer that receives the `ADD_PROVIDER` RPC should validate that the
received `PeerInfo` matches the sender's `peerID`, and if it does, that peer
must store a record in its datastore the received `PeerInfo` record.

_Getting_ the providers for a given key is done in the same way as _getting_ a
value for a given key (see [getting values section](#getting-values)) expect
that instead of using the `GET_VALUE` RPC message the `GET_PROVIDERS` RPC
message is used.

When a node receives a `GET_PROVIDERS` RPC, it must look up the requested
key in its datastore, and respond with any corresponding records in its
datastore, plus a list of closer peers in its routing table.

For performance reasons, a node may prune expired advertisements only
periodically, e.g. every hour.

### Bootstrap process

The bootstrap process is responsible for keeping the routing table filled and
healthy throughout time. It runs once on startup, then periodically with a
configurable frequency (default: 5 minutes).

On every run, we generate a random node ID and we look it up via the process
defined in [*Node lookups*](#node-lookups). Peers encountered throughout the
search are inserted in the routing table, as per usual business.

This process is repeated as many times per run as configuration parameter
`QueryCount` (default: 1). Every repetition is subject to a `QueryTimeout`
(default: 10 seconds), which upon firing, aborts the run.

## RPC messages

Remote procedure calls are performed by:

1. Opening a new stream.
2. Sending the RPC request message.
3. Listening for the RPC response message.
4. Closing the stream.

On any error, the stream is reset.

All RPC messages conform to the following protobuf:

```protobuf
// Record represents a dht record that contains a value
// for a key value pair
message Record {
	// The key that references this record
	bytes key = 1;

	// The actual value this record is storing
	bytes value = 2;

	// Note: These fields were removed from the Record message
	// hash of the authors public key
	//optional string author = 3;
	// A PKI signature for the key+value+author
	//optional bytes signature = 4;

	// Time the record was received, set by receiver
	string timeReceived = 5;
};

message Message {
	enum MessageType {
		PUT_VALUE = 0;
		GET_VALUE = 1;
		ADD_PROVIDER = 2;
		GET_PROVIDERS = 3;
		FIND_NODE = 4;
		PING = 5;
	}

	enum ConnectionType {
		// sender does not have a connection to peer, and no extra information (default)
		NOT_CONNECTED = 0;

		// sender has a live connection to peer
		CONNECTED = 1;

		// sender recently connected to peer
		CAN_CONNECT = 2;

		// sender recently tried to connect to peer repeatedly but failed to connect
		// ("try" here is loose, but this should signal "made strong effort, failed")
		CANNOT_CONNECT = 3;
	}

	message Peer {
		// ID of a given peer.
		bytes id = 1;

		// multiaddrs for a given peer
		repeated bytes addrs = 2;

		// used to signal the sender's connection capabilities to the peer
		ConnectionType connection = 3;
	}

	// defines what type of message it is.
	MessageType type = 1;

	// defines what coral cluster level this query/response belongs to.
	// in case we want to implement coral's cluster rings in the future.
	int32 clusterLevelRaw = 10; // NOT USED

	// Used to specify the key associated with this message.
	// PUT_VALUE, GET_VALUE, ADD_PROVIDER, GET_PROVIDERS
	bytes key = 2;

	// Used to return a value
	// PUT_VALUE, GET_VALUE
	Record record = 3;

	// Used to return peers closer to a key in a query
	// GET_VALUE, GET_PROVIDERS, FIND_NODE
	repeated Peer closerPeers = 8;

	// Used to return Providers
	// GET_VALUE, ADD_PROVIDER, GET_PROVIDERS
	repeated Peer providerPeers = 9;
}
```

These are the requirements for each `MessageType`:

* `FIND_NODE`: In the request `key` must be set to the binary `PeerId` of the
  node to be found. `closerPeers` is set in the response; for an exact match
  exactly one `Peer` is returned; otherwise `k` closest `Peer`s are returned.

* `GET_VALUE`: In the request `key` is an unstructured array of bytes. If `key`
  is a public key (begins with `/pk/`) and the key is known, the response has
  `record` set to that key. Otherwise, `record` is set to the value for the
  given key (if found in the datastore) and `closerPeers` is set to the `k`
  closest peers.

* `PUT_VALUE`: In the request `key` is an unstructured array of bytes. The
  target node validates `record`, and if it is valid, it stores it in the
  datastore.

* `GET_PROVIDERS`: In the request `key` is set to a CID. The target node
  returns the closest known `providerPeers` (if any) and the `k` closest known
  `closerPeers`.

* `ADD_PROVIDER`: In the request `key` is set to a CID. The target node verifies
  `key` is a valid CID, all `providerPeers` that match the RPC sender's PeerID
  are recorded as providers.

* `PING`: Target node responds with `PING`. Nodes should respond to this message
  but it is currently never sent.

Note: Any time a relevant `Peer` record is encountered, the associated
multiaddrs are stored in the node's peerbook.

## Appendix A: differences in implementations

The `addProvider` handler behaves differently across implementations:
  * in js-libp2p-kad-dht, the sender is added as a provider unconditionally.
  * in go-libp2p-kad-dht, it is added once per instance of that peer in the
    `providerPeers` array.

---

## References

[0]: Maymounkov, P., & Mazières, D. (2002). Kademlia: A Peer-to-Peer Information System Based on the XOR Metric. In P. Druschel, F. Kaashoek, & A. Rowstron (Eds.), Peer-to-Peer Systems (pp. 53–65). Berlin, Heidelberg: Springer Berlin Heidelberg. https://doi.org/10.1007/3-540-45748-8_5

[1]: Baumgart, I., & Mies, S. (2014). S / Kademlia : A practicable approach towards secure key-based routing S / Kademlia : A Practicable Approach Towards Secure Key-Based Routing, (June). https://doi.org/10.1109/ICPADS.2007.4447808

[2]: Freedman, M. J., & Mazières, D. (2003). Sloppy Hashing and Self-Organizing Clusters. In IPTPS. Springer Berlin / Heidelberg. Retrieved from www.coralcdn.org/docs/coral-iptps03.ps

[3]: [bep_0005.rst_post](http://bittorrent.org/beps/bep_0005.html)

[5]: [GitHub - multiformats/multihash: Self describing hashes - for future proofing](https://github.com/multiformats/multihash)