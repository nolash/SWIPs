---
SWIP: 35
title: Kademlia forwarding service
author: Louis Holbrook <dev@holbrook.no> (https://holbrook.no)
discussions-to: <URL>
status: Draft
type: Core
created: 2019-12-11
requires (*optional): <SWIP number(s)>
replaces (*optional): <SWIP number(s)>
---

## Simple Summary

Provide a service that provides the most eligible peers for receiving a given message.


## Abstract

Currently all components that need to send messages to peers make their own calculations to determine which peer to send to. In particular the retrieve, push sync and pss protocols all need this feature.

This SWIP proposes to provide this feature as a standard service of the network package.

## Motivation

Avoid parallell implementations of the same feature.

## Specification

The reference implementation is specified in `golang`.

```

New(capabilityIndex) SessionInterface


type SessionInterface interface {
	Subscribe() <-ForwardPeer
	Get(numberOfPeers int) ([]ForwardPeer, error)
	Close()
}

// also implements context.Context
type SessionContext struct {
	CapabilityIndex string
	SessionId string
}

type SessionRPCInterface interface {
	NewForwardPeerSession(ctx context.Context) (context.Context, error)
	Subscribe(ctx context.Context, "kademlia", ch chan PeerEvent, "forward") (sub rpc.Subscription, err error)
	GetForwardPeer(ctx context.Context, numberOfPeers int) (peers []ForwardPeer, err error)
	CloseForwardPeerSession(ctx context.Context)
}
```

### Library interface

The `capabilityIndex` argument to `New` and `CapabilityIndex` member of `SessionContext` refers to the key of a  `Capability` object registered with `Kademlia.RegisterCapabilityIndex`. Provide an empty string means peers will not be filtered by capability.

`Get` provides two modes of operation:

* If `Subscribe` has not been called, `Get(n)` returns a slice of up to `n` peers.
* If `Subscribe` has been called, `Get(n)` returns an empty slice, and up to `n` peers will be put on the channel instead.

`Get` called with `numberOfPeers = 0` returns _all_ matching peers in order.

`Close` _MUST_ always be called to free up the component's session peer cache/cursor, and the subscription channel if subscription has been made.

### RPC interface

If `Get` is called with `SessionContext` _and_ the `SessionId` variable is set _and_ matches a `Session` created through `rpc.Subscribe`, then the next eligible peers for the `Session` will be returned on the subscription channel.

Otherwise, the `Session` terminates immediately after completing the `Get` request.

To initiate a `Session`, `NewForwardPeerSession` should be called with a `SessionContext` as the parameter (a different context will generate an error). The returned context will contain a unique `sessionId` key generated by the server. This context can in turn be passed to `Get` and (optionally) `Subscribe` to attain the corresponding functionality provided by the library interface.

If `NewForwardPeerSession` has been called, `CloseForwardPeerSession` _MUST_ be called with the `SessionContext` updated by the server to terminate the `Session`.

## Rationale

The component shall provide a service where given a specific Swarm Overlay Address, connected peers will be returned in the order of closest to farthest.

The following caller requirements are considered:

1. **Cardinality:** Should the message be forwarded to only one or several peers?
2. **Symmetry:** Is a reply expected/required from the peer?
3. **Capability:** What capabilities must a peer have?

The component must also provide a way to react to peer disconnections and new peer connections that are better suited than the request momentarily in flight.

It is assumed that when a caller makes a new request, the usefulness of the peers previously returned has been exhausted.

### Cardinality

The caller should be able to receive one or more peers per request.

The caller and component need to establish a session, where the caller can traverse the full set of eligible peers in order across requests.

The component must keep track of which peers have already been returned.

The component must also update the set corresponding to disconnections or new connections.

In case of a message address

### Symmetry

If a reply from the peer is required, some time passes between the message is sent and reply is received.

In the meantime peers may disconnect, or new peers may connect.

To detect new connections, the caller must periodically make new requests, _or_ an event subscription must be provided to notify the caller in real time.

To detect disconnections, an event subscription seems to be the only logical choice.

### Capability

With the kademlia implementation of capabilities this is fairly straightforward:

The capability set must have been registered in the Kademlia at startup. It keeps a subset index of peers that have said capabilities.

The request session is created against this index.

### Conclusion

Caller and component establish a _session_ against a _Swarm Overlay Address_.

Within a _session_, a caller may request connected peers from the component.

The caller can specify a set of capabilities peers must have. This set of capabilities must be an existing kademlia capability filter.

Sequential requests shall return the _closest_ peer(s) available for the _session_ that have not been returned since the last connection made with the peer.

Component provides an _optional_ channel that notifies:

1. Disconnection of a peer returned to the last request
2. A new peer has connected that was not part of the last request.

Note that 2. may result in a peer being returned more than once, if the peer was disconnected and then reconnected within the same _session_.

## Backwards Compatibility

This feature replaces duplicated code in components that need this feature. The component can be independently constructed, and the dependent components can be individually updated thereafter.

## Test Cases

The current components that implement some sort of forwarding inform the necessary test cases to achieve the same functionality with the new component.

### Component

* Peers received in order across requests in session, regardless of amount per request
* Only peers with specific capability received in order across requests in session
* New (re-)connects that are closer are returned in subsequent requests
* Disconnect and connect notifications

### Retrieval

* Closest peer(s) with retrieval capability to chunk address (forwarding kademlia phase)
* If warranted, change peer pending chunk. (symmetric request)

### Push Sync

* Closest peer with push sync capability to chunk address (forwarding kademlia phase)
* If warranted, change peer pending receipt. (symmetric request)
* Peers in chunk address' neighborhood (for the proximity send phase)

### Pss 

1. Closest peer with pss forward capability to chunk address (forwarding kademlia phase)
2. Peers in caller address' neighborhood (for the redundancy phase)
3. Peers closer than partial address (custom luminosity phase)
4. All peers (broadcast phase)

At the current time a replacement feature, either partial or in full, has been suggested for `pss`, using special chunk types disseminated by the normal chunk syncing mechanisms. Relevance of these test cases are pending a decision on this.

## Implementation

Pending 

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).