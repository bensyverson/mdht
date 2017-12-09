## mdht.js -- Mainline DHT

Dynamic Hash Table customized for the Mainline DHT used by bittorrent to locate torrent peers without using a tracker.
Includes BEP44 data storage. IPv4 only. References: [BEP5](http://www.bittorrent.org/beps/bep_0005.html), [BEP44](http://www.bittorrent.org/beps/bep_0044.html)

### Quick start

```
$ npm i mdht
$ echo "require('mdht/server')" > server.js
$ node server.js
```
server.js will connect to the DHT via default UDP port 6881, by bootstraping from
router.bittorrent.com:6881, and then begin to respond to requests from other nodes,
using a random node id. Configuration information and incomming requests are displayed on the
console. The id and routing table are periodically saved to disk for future restarts.

server.js interfaces between mdht/mdht.js (which in turn interfaces with the DHT network), the user
(via the command line) and the disk (to preserve its state between sessions). It also accepts
commands to issue outgoing requests to other nodes via HTTP on the same port.

mdht/client.js (under development) shows how to make HTTP requests for getting and putting
BEP44 data, for example.

### Terminology:

Term | Description
-----|------------
*location* | network *location* (6-byte buffer: 4-byte IPv4 address + 2-byte port); example: Buffer.from('ff0000011ae1', 'hex') is '127.0.0.1:6881'
*id* | DHT node *id*, infohash of a torrent, or target of BEP44 data (20-byte buffer)
node | a member of the Mainline DHT network which uses UDP
peer | a bittorrent client associated with a DHT node which uses TCP, usually on the same port

### Usage (API):
```
const dhtInit = require('mdht')
const dht = dhtInit(options, update) // options is an object, update is a callback function
```
#### dhtInit options:

Option | Description
-------|------------
options.port | local UDP server port (int, default 6881)
options.id | local node *id* (default random)
options.seed | seed for generating ed25519 key pair for signing mutable data (32-byte buffer, default random)
options.bootLocs | remote node *locations* to contact at startup (buffer of concatenated *locations*, default empty)

#### dhtInit returns an object with the following methods:
```
dht.announcePeer(ih, ({ numVisited: ..., numAnnounced: ... }) => {}, onV) // assumes local peer uses options.port
dht.getPeers(ih, ({ numVisited: ..., values: ... }) => {}, onV)
dht.putData(v, mutableSalt, resetTarget, ({ numVisited: ..., numStored: ..., target: ..., v: ..., salt: ..., seq: ..., k: ..., sig: ... }) => {}, onV)
dht.getData(target, mutableSalt, ({ numVisited: ..., numFound: ..., v: ..., seq: ... }) => {}, onV)
dht.makeMutableTarget(k, mutableSalt)
dht.makeImmutableTarget(v)
```
##### where:

announcePeer, getPeers, putData, getData initiate outgoing DHT queries to remote nodes and return
information and/or results in the first callback function after all nodes have responded.

The first callback function returns a single object with multiple properties pertinent to the method.
Object properties that do not apply or were not found are ommitted. For example, there will be no
.values or .v property if no values are found; and .salt, .seq, .k and .sig are only returned for
mutable data.

The second callback function, onV, if not null or undefined, is called immediately whenever peer
locations or BEP44 data are received from a remote node, with a single argument: an object with
.target and .values for getPeers and announce Peers, or .ih and .v for getData and putData,
plus .socket in both cases.

makeMutableTarget and makeImmutableTarget are utilities for computing targets (node *ids*) for use
with getData or putData, which can be called with the arguments target or resetTarget, respectively,
along with mutableSalt, as provided by whomever stored the data. If the target is unknown, it can be
computed with makeMutableTarget (if k and mutableSalt are known) or makeImmutableTarget
(if v is known).

Argument | Description
---------|------------
ih | infohash, *id* of a torrent
values | array of peer *locations* which have the torrent with infohash ih
target | *id* of BEP44 data
v | BEP44 data stored in or retrieved from the DHT (object, buffer, string or number)
mutableSalt | if immutable BEP44 data then *false* or *''*; if mutable data then *true* if no salt, or *salt* (non-empty string or buffer -- string will be converted to buffer, buffer will be truncated to 64 bytes)
resetTarget | if not null, a target used to reset the timeout of previously stored mutable data (v is ignored in this case)

Property | Description
---------|------------
seq | sequence number (int) of mutable data
sig | ed25519 signature of salt, v and seq (64-byte buffer)
k | public key used to make a mutable target and to sign and verify mutable data (32-byte buffer); if null, local public key is used
socket | node socket, object version of node 'location' { address: (string), port: (int) }

#### update is a function which signals the calling program and is called with two arguments (key, value)

Key | Signal | Value
----|--------|------
'udpFail' | initialization failed | local port (int) that failed to open; calling program should restart using a different port
'id' | initialized | *id* actually used to create routing table
'publicKey' | initialized | public key (k) actually used for ed25519 signatures
'listening' | local udp socket is listening | local socket
'ready' | bootstrap is complete | number of nodes visited during bootstrap
'incoming' | incoming query object | { q: query type (string), socket: remote socket }
'error' | incoming error object | { e: [error code (int), error message (string)], socket: remote socket }
'locs' | periodic report | buffer packed with node *locations* from the routing table; may used for disk storage
'closest' | periodic report | array of node *ids* from the routing table, the closest nodes to the table *id*
'peers' | periodic report | { numPeers: number of stored peer locations, numInfohashes: number of stored infohashes }
'data' | periodic report | number of BEP44 stored data items
'spam' | detected spammer node, temporarily blocked| 'address:port'
'dropNode' | node dropped from routing table | 'address:port'
'dropPeer' | peer location dropped from storage | 'address:port'
'dropData' | data dropped from BEP44 storage | 'target' (hex string)

### test.js example program
This program provides a command line interface for mdht.js as well as
an interface with disk storage. The *id*, seed and boot *locations* are saved in separate files
between sessions. Without these files, the DHT will use random values for *id* and seed, but would
require a boot *location* as a command line argument. Usage: `require('mdht/test.js')` alone in a
file named, for example, `test.js`.

When executed, the program displays configuration information and incoming requests from other nodes.
An HTTP server is bound to the same port as the DHT, but uses TCP instead of UDP. The server
can accept commands via HTTP to initiate outgoing queries. See client.js, for example.

### shim.js interface with Webtorrent
This program is a shim between mdht.js and [webtorrent](https://github.com/webtorrent/webtorrent)
as a replacement for [bittorrent-dht](https://github.com/webtorrent/bittorrent-dht), which is problematic.
[webtorrent/index.js](https://github.com/webtorrent/webtorrent/blob/master/index.js) needs to be modified locally
in `node_modules/webtorrent` so that it requires `mdht/shim` rather than `bittorrent-dht/client`. Then, invoke webtorrent like so:
```
const WebTorrent = require('webtorrent')
// must modify webtorrent to require mdht/shim instead of bittorrent-dht/client

const client = new WebTorrent({ torrentPort: port, dhtPort: port, dht: { nodeId: *id*, bootstrap: bootLocs, seed: seed } })
// `port` is a port number and `*id*`, `bootLocs` and `seed` are buffers destined for mdht.js (see dhtInit options above).
```

Then use (see [torr.js](https://github.com/metamystical/torr) for an example):
```
client.dht.once('ready', function () { )) // bootstrap complete, ready for new torrents
client.dht.on('nodes', function (nodes) { }) // periodic report of DHT routing table node *locations* for saving
client.dht.nodeId // actual nodeId used
const ret = client.dht.put(v, mutableSalt, resetTarget, function (numVisited, numStored) { })
client.dht.get(target, mutableSalt, function (numVisited, { v: (object), seq: (int), numFound: (int) }) { } )
```
