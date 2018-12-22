Here is a somewhat tentative/radical proposal for how to refactor the existing DHT to make it more extensible in the future. This is roughly based on what @Stebalien and I discussed last week. Any feedback welcome!

The first step is a backwards-compatible refactor into the components described below; the second step would be a new RPC protocol sketched out at the end.

I propose the following components:

1. Network Handler
This component handles connection open/close events, errors, and inbound and outbound RPCs. Outbound RPCs coming from other components will have their responses routed back to the originating component. Initially, this component will handle the current DHT protobuf format. Eventually, driver components (below) will register which `DhtProtocol` types of inbound RPCs they wish to handle in the proposed extensible DHT protocol (bottom). 

2. Outgoing Peer Routing Driver
This is the component that exposes the [Peer Routing interface](https://github.com/libp2p/interface-peer-routing). Its primary job is implementing `findPeers` function, which should return a stream (as opposed to just a single callback in the current JS version of the DHT). This component schedules outbound `FIND_NODE` RPCs and decides what to do with the responses. In other words, this is the part that determines how many requests are outstanding per query and whether we use multiple disjoint lookup paths (as in S/Kademlia). In the longer term, this will be where many of the improved lookup heuristics will reside, including layering behavior like Coral.

Additionally, this component should be notified of all incoming RPCs so that it can maintain the routing table for use by other DHT modules. This also includes handling routing table eviction (determining whether a peer is still alive) and perhaps blacklisting/filtering in the future.

Finally, this component handles retrieving and verifying public keys for other nodes, which should be included in all PeerInfo records (eventually).

3. Incoming Peer Routing Driver (not present in client-only mode)
This handles incoming `FIND_NODE` RPCs by looking in the routing table and sending appropriate responses. Since all nodes require a routing table, that has to be maintained in the Outgoing driver. Complexity in this component should be minimal.

4. Outgoing Content Routing Driver
This component exposes the [Content Routing interface](https://github.com/libp2p/interface-content-routing). It implements `findProviders` (which should provide a stream of responses) and `provide`. Internally, this involves sending `GET_PROVIDERS` and `ADD_PROVIDER` RPCs to relevant nodes, and running validators on the responses. Like the corresponding Peer Rouding driver, this will be where useful heuristics for finding content live. To enable that, we will need to design an interface for callers to `findProviders` to provide feedback on which responses were ultimately useful; e.g. whether each returned provider actually provided the content advertised.

5. Incoming Content Routing Driver (not present in client-only mode)
This handlies incoming `GET_PROVIDERS` and `ADD_PROVIDERS` RPCs. To do so, it manages the list of providers. This includes validating providers advertised in `GET_PROVIDERS`, maintaining time-to-live and expiration for each advertised provider, and (eventually) heuristics to block bad provider advertisements. Potentially this could go so far as interfacing with an external component that actually tries to retreive a given piece of content before advertising it to other peers.

6. Additional components
It is then possible to keep adding to this pattern of components for new types of records. For example, `PutValue` and `GetValue` would be provided by an Outgoing Value Driver, and the inbound corresponding RPCs would be handled by an Incoming Value Driver. In general, every incoming RPC request will trigger both the Incoming Peer Routing driver (for closer peers) and the type-specific Incoming Driver (for values/providers/etc.); the responses from these will then be bundled back into the response by the Network Handler.

Some ideas for more driver pairs:
* PubSub membership
* Arbitrary value put/get (although we very well may not want this!)
* Privacy-preserving mapping between public and private `PeerID`s

Initially, the Network Handler would handle the same protobuf format as now. Eventually, I envision a new RPC protocol where each request/response looks similar to this:

struct DhtRPC {
    type DhtProtocol // enum determining which driver (other than PeerRouting which is implicit) will handle the message; values of PeerRouting, ContentRouting, Value, ...
    supported []DhtProtocol // list/bitfield of all supported DhtProtcols for the sending peer. Allows sharing one global routing table for all types
    key bytes // key for request
    response bool // indicates request vs. response
    requestId bytes // random nonce to match request to response
    closerPeers []PeerInfo // addresses for peer routing (present in all responses). Should be signed and include Ed25519 public key
    data bytes // binary blob format specific to the given message type (present in all responses)
}

Note that due to the requestId field this can also be sent over a message-oriented (as opposed to stream-oriented) transport.

The Network Handler would deal with both this new format and the existing DHT protobuf and route messages to the appropriate driver.