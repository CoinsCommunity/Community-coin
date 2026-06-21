# Section 1: Network and Peer Discovery

## 1.1 Overview

Community is a peer-to-peer network. There are no servers, no master nodes, and no privileged participants. Every node is simultaneously a client and a server. The network has no central coordinator and requires none to function. Nodes discover each other, exchange transactions and blocks, and maintain consensus without any party having special authority over any other.

---

## 1.2 Transport

Community nodes communicate over TCP. The default listening port is DEFAULT_PORT, a genesis parameter.

Nodes may listen on a non-default port. When advertising their address to peers, nodes include the port they are listening on. Connections to non-default ports are accepted and treated identically to connections on the default port.

All connections are encrypted and authenticated using a session key established during the handshake defined in Section 1.5. Plaintext connections are not accepted.

---

## 1.3 Node Identity

Each node generates a persistent identity keypair on first launch using the signature scheme defined in Section 9. The identity keypair is stored locally. The public key of this keypair is the node's identity, referred to as NODE_ID.

NODE_ID is used during the handshake to authenticate connections and prevent certain eclipse attack vectors. It is not linked to wallet identity. A node operator's financial activity is not attributable to their NODE_ID.

Nodes may rotate their identity keypair at any time by generating a new one. Rotation causes existing peers to treat the node as a new unknown peer on next connection.

---

## 1.4 Peer Addresses

A peer address is the combination of an IP address, a port, and a NODE_ID. Both IPv4 and IPv6 addresses are supported. Nodes that support both maintain separate peer lists for each address family and actively maintain connections on both.

Peer addresses are stored in a local peer database that persists across restarts. The peer database records each known address along with metadata including last successful connection time, last attempted connection time, and the source that provided the address.

The peer database has a maximum capacity of MAX_PEER_DB_SIZE entries, a configurable local parameter with a recommended default. When the database is full and a new address is learned, the oldest entry by last successful connection time is evicted.

---

## 1.5 Connection Handshake

When node A initiates a connection to node B:

1. A sends a VERSION message containing: protocol version, NODE_ID of A, listening address and port of A, current block height of A, genesis block hash.

2. B validates the VERSION message. B rejects the connection if: the protocol version is incompatible, the genesis block hash does not match the Community genesis block hash, or A's NODE_ID is on B's ban list.

3. B sends a VERACK message containing: NODE_ID of B, listening address and port of B, current block height of B.

4. A validates the VERACK. A disconnects if B's genesis block hash does not match or B's NODE_ID is banned.

5. Both nodes derive a shared session encryption key from their identity keypairs using the key encapsulation mechanism defined in Section 9. All subsequent communication on this connection is encrypted with this session key.

6. The connection is established. Both nodes add each other to their peer database.

A connection that fails at any step is closed. The failed address is recorded in the peer database with the failure timestamp.

---

## 1.6 Bootstrap and Initial Peer Discovery

A node with an empty peer database has no known peers to connect to. Bootstrap is the process of finding initial peers.

**Hardcoded bootstrap addresses.** Client software ships with a list of hardcoded bootstrap node addresses. These are community-operated nodes with stable long-term addresses. Any participant may operate a bootstrap node. The hardcoded list is a client software concern, not a protocol concern. The protocol does not define specific bootstrap addresses. Client implementations maintain this list and may update it in new software versions.

**DNS seeds.** Client software may include DNS seed hostnames that resolve to lists of currently active node addresses. DNS seeds are operated by community members. The protocol does not define specific DNS seed hostnames. DNS seeds are queried on first launch and when the peer database falls below a minimum threshold.

**Local network discovery.** Nodes broadcast a discovery message on the local network segment using UDP. Nodes on the same local network respond with their address. This allows home users and small networks to find each other without relying on external bootstrap infrastructure.

**Peer exchange.** Once connected to any peer, a node may send a GETADDR message requesting that peer's known addresses. The receiving node responds with an ADDR message containing up to MAX_ADDR_RESPONSE addresses from its peer database, selected randomly. This spreads knowledge of the network organically through existing connections.

A node that cannot connect to any bootstrap address, DNS seed, or local network peer cannot join the network. This is the expected behavior for a node that is offline or firewalled. There is no fallback that degrades security to enable connectivity.

---

## 1.7 Connection Management

**Target connections.** Each node maintains between MIN_OUTBOUND_CONNECTIONS and MAX_OUTBOUND_CONNECTIONS outbound connections to peers, and accepts up to MAX_INBOUND_CONNECTIONS inbound connections. These are configurable local parameters with recommended defaults.

Outbound connections are initiated by the node to peers of its choosing. Inbound connections are initiated by remote peers. Nodes actively seek to maintain at least MIN_OUTBOUND_CONNECTIONS outbound connections at all times. If outbound connections fall below this threshold the node attempts new connections from its peer database.

**Peer selection for outbound connections.** Peers are selected from the peer database with preference for: peers with recent successful connections, peers not recently attempted, and peers from diverse network ranges as defined in Section 1.8. Random selection weighted by these preferences prevents any single peer from monopolizing a node's outbound connections.

**Keepalive.** Nodes send PING messages to connected peers at regular intervals. A peer that does not respond to PING within PING_TIMEOUT seconds is disconnected. The disconnected peer's address remains in the peer database and may be retried later.

**Disconnection.** Either party may disconnect at any time by closing the TCP connection. Ungraceful disconnection (network failure, process crash) is handled by connection timeout detection.

---

## 1.8 Eclipse Attack Mitigations

An eclipse attack occurs when an attacker controls all of a node's connections, isolating it from the honest network and feeding it false information. Community implements the following mitigations.

**Subnet diversity requirement.** No more than MAX_CONNECTIONS_PER_SUBNET outbound connections may be made to addresses within the same /16 IPv4 subnet or the same /48 IPv6 prefix. MAX_CONNECTIONS_PER_SUBNET is a configurable local parameter with a recommended value of 1. This prevents an attacker operating many nodes from the same hosting provider from dominating a victim's outbound connections.

**Address source tracking.** The peer database records which peer provided each address. Addresses sourced entirely from a single peer are deprioritized for outbound connection attempts. An attacker who feeds a victim node many addresses cannot guarantee those addresses will be used for connections.

**Inbound connection limits per subnet.** No more than MAX_INBOUND_PER_SUBNET inbound connections are accepted from the same /16 IPv4 subnet or /48 IPv6 prefix. This limits an attacker's ability to exhaust a node's inbound connection slots from a single infrastructure.

**Feeler connections.** Periodically, nodes open short-lived test connections to randomly selected untried addresses from the peer database. Addresses that successfully complete a handshake are promoted in the peer database. Addresses that fail are recorded as failed. Feeler connections ensure the peer database contains live, reachable addresses and prevents stagnation from relying only on currently-connected peers.

**Anchor connections.** Nodes maintain a small number of anchor connections to peers with long connection histories. Anchor peers are reconnected preferentially on restart. An attacker who briefly eclipses a node during a restart cannot sustain the eclipse if anchor peers are reachable.

---

## 1.9 Message Types

All messages between peers use a common envelope format:

| Field | Size | Description |
|---|---|---|
| MAGIC | 4 bytes | Network magic bytes identifying the Community network |
| COMMAND | 12 bytes | ASCII command name, null-padded |
| LENGTH | 4 bytes | Length of the payload in bytes |
| CHECKSUM | 4 bytes | First 4 bytes of HASH(HASH(payload)) |
| PAYLOAD | variable | Message-specific data |

MAGIC is a genesis parameter. Its value distinguishes Community messages from messages belonging to other protocols or networks. A node receiving a message with an unrecognized MAGIC value closes the connection immediately.

The defined message commands are:

| Command | Direction | Description |
|---|---|---|
| VERSION | outbound on connect | Initial handshake |
| VERACK | response to VERSION | Handshake acknowledgment |
| PING | either | Keepalive request |
| PONG | response to PING | Keepalive response |
| GETADDR | either | Request peer addresses |
| ADDR | response to GETADDR | List of known peer addresses |
| INV | either | Inventory announcement (block or transaction hash) |
| GETDATA | either | Request full block or transaction by hash |
| BLOCK | response to GETDATA | Full block data |
| TX | response to GETDATA | Full transaction data |
| REJECT | either | Notification that a message was rejected and why |

---

## 1.10 Transaction and Block Relay

**Inventory announcements.** When a node learns of a new transaction or block, it sends an INV message to its connected peers containing the hash of the item. INV messages do not contain the full item, only its hash.

A peer that receives an INV message and does not have the referenced item sends a GETDATA message requesting the full item. The originating node responds with a BLOCK or TX message containing the full data.

A peer that already has the referenced item ignores the INV message without responding.

**Relay policy for transactions.** Nodes relay transactions that pass basic validity checks: valid signature, valid structure as defined in Section 3, fee above the minimum relay fee. Nodes do not relay transactions that fail these checks. This prevents the network from being flooded with invalid transactions.

**Relay policy for blocks.** Nodes relay pending blocks (valid PoW, valid body, awaiting staker votes) and finalized blocks. Nodes do not relay blocks with invalid PoW solutions or invalid bodies. A node that receives an invalid block records the sending peer as misbehaving and may disconnect and ban that peer.

**Compact block relay.** To reduce bandwidth, nodes may negotiate compact block relay during the handshake. Under compact block relay, a node receiving a new block announcement sends a compact representation containing only the block header and short transaction identifiers rather than the full transaction data. The receiving node reconstructs the full block from transactions already in its mempool, requesting only missing transactions from the sender. Compact block relay is an optimization and does not change block validity rules.

---

## 1.11 Misbehavior and Banning

Nodes track a misbehavior score for each connected peer. Specific protocol violations add to a peer's misbehavior score. If a peer's misbehavior score exceeds BAN_THRESHOLD, the node disconnects the peer and adds their address to a ban list. Banned addresses are refused connections for BAN_DURATION seconds.

Actions that increase misbehavior score include sending invalid blocks, sending invalid transactions, sending messages with invalid checksums, exceeding message rate limits, and sending messages that violate protocol rules. The specific score increments for each violation are implementation-defined within the constraint that a single critical violation (such as sending an invalid block with a valid PoW) results in an immediate ban.

BAN_THRESHOLD and BAN_DURATION are configurable local parameters.

---

## 1.12 Genesis Parameters Summary

| Parameter | Description |
|---|---|
| DEFAULT_PORT | Default TCP port for node connections |
| MAGIC | 4-byte network identifier in message envelopes |
| MAX_ADDR_RESPONSE | Maximum addresses returned in an ADDR message |
