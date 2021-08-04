# IPFS-Chat

1. Real-time peer-to-peer messaging using [IPFS pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-pubsub). Works over both internet and LAN. File-sharing is not implemented yet but will be soon.
2. The usual *Create Alias/Nick* + *Create/Join room* workflow (akin to [IRC](https://en.wikipedia.org/wiki/Internet_Relay_Chat)).
3. Peers are discovered using [DHT](https://docs.ipfs.io/concepts/dht/), [pubsub](https://docs.libp2p.io/concepts/publish-subscribe/) and [mDNS](https://en.wikipedia.org/wiki/Multicast_DNS) (See [Peer discovery](#peer-discovery)). No need for any rendezvous server.
4. Chat-messages are authenticated and end-to-end encrypted (See [Security](#security)).
5. Very basic terminal-based UI without any eye candy.
6. Written entirely in [Bash](https://www.gnu.org/software/bash/manual/bash.html); just a single shell-script. Apart from [go-ipfs](https://docs.ipfs.io/install/command-line/#official-distributions) and possibly `argon2`, depends only on standard GNU/Linux tools and tools that can be easily downloaded from the native package repository.

***Keywords*:** p2p; distributed; server-less; broker-less; TUI; secure; texting; file-sharing; ipfs; pubsub; privacy

## Usage

```shell
ipfs-chat -r <room> -n <nick> -d <download directory> -c <ipfs repo>
ipfs-chat -g # Generating a random room name, for when your brain can't do it
ipfs-chat -v # Version
ipfs-chat -h # Help
```

## Snapshot

![ipfs-chat_snapshot](./screenshot.png)

## Peer discovery

Name of the room is the shared secret. 

Every participant provides a (rendezvous) file at regular intervals whose content is a time-based nonce derived from the shared secret, viz. the room name. Along with this, participants publish their multiaddresses at a pubsub topic that is also derived from the room name.

To join the chat network, viz. the room, one needs to connect to as many online participants as possible. This is done by first querying the DHT for the providers of the rendezvous file and then swarm connecting to those peer IDs. Once in the net, participants can also listen to the pubsub topic where the multiaddresses are published in order to discover peers they are not directly connected to. To accommodate for peers leaving and joining the room, the query and swarm connect steps are iterated at regular intervals.

Peers behind NAT use [autorelay or p2p-circuit](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#autorelay) to be accessible by others.

Local (LAN based) discovery is also enabled ([Discovery.MDNS.Enabled=true](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md)).

Also, if a peer sees a message (over pubsub) from a peer that it is not directly connected to, it tries to connect to it immediately.

**Note**: The rendezvous nonce changes every 2 mins. Due to this, a peer is shown to be online upto 2 mins after it goes offline.

## Security

Authenticity of the messages is established through IPNS over pubsub (See [Messaging](#messaging)).

All messages are encrypted with a symmetric key derived from the room name using HMAC.

The pubsub topics are separate HMAC keys derived from the room name. 

Therefore, the public network, that mediates the pubsub and passes the messages along, never knows the actual room name and hence, the encryption key.

Before deriving the keys, the room name is itself hashed using a memory- and CPU-hard password-hashing algorithm ([Argon2](https://github.com/P-H-C/phc-winner-argon2)) [Not implemented yet].

## Messaging

Every peer publishes its nick and a self-signed PGP ed25519 pubkey under its peer ID over IPNS at the start of every session. This authenticates its claim over the nick and pubkey.

After discovering a peer, other peers resolve its IPNS entry and caches its nick and pubkey for use throughout the session.

For messaging, a peer signs its message with its private key and encrypts with the aforementioned symmetric key derived from the room name. The whole encrypted object is encoded in base64 and published over pubsub along with the sender's peer ID.

Other peers receive this over pubsub, decrypt the message and verify the signature. If everything is ok, they then display the message in their UI against the sender's nick, peer ID and timestamp.

[IPNS-pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipns-pubsub) is enabled to aid in resolving IPNS names as quickly as possible, and reducing the dependence on DHT.

## Efficiency

[IPFS usually consumes a lot of bandwidth](https://github.com/ipfs/go-ipfs/issues/3429). To make it more efficient, all unnecessary connections are closed at regular intervals: The node maintains a list of all chatroom peers seen or discovered in a session. All other connections, except connections to the relays the node itself or its chatroom peers are connected to, are culled on a frequent basis.

New connections are always being formed to nodes within the general IPFS network. The interval between consecutive cullings is large enough to gather sufficient number of nodes for performing DHT operations such as providing the rendezvous file, finding other providers and finding the multiaddresses of those providers. Once these operations are finished, the culling is performed.

Connections to the chatroom peers are never closed.

To achieve this, `ipfs-chat` uses its own connection manager and does not use the [basic connection manager](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#basic-connection-manager) that ships with `go-ipfs`.

The go-ipfs config file has been tuned to reduce resource (CPU/bandwidth/memory) consumption.

**Tip**: To show bandwidth usage by the node at the end of a session, launch `ipfs-chat` with the `-b` flag. Note that this shows the cumulative bandwidth consumption by all `ipfs-chat` instances using the same node at the same time.

## Future directions

1. Secure file sharing. Files to be shared can simply be dragged and dropped in the text-input area. Support for WSL.
2. Using Argon2 for more security (See [Security](#security)).
3. Private messages: To PM a peer simply prefix your message with @nick(peer ID).
4. Offline mode such that even if a peer goes offline, it can obtain the missed messages when it comes back online [This may well be beyond my capabilities].

## Bug-reports and Feedbacks

Post at [issues](https://github.com/SomajitDey/ipfs-chat/issues) and [discussion](https://github.com/SomajitDey/ipfs-chat/discussions), or [write to me](mailto://hereistitan@gmail.com).

------

###### [GNU GPL v3-or-later](https://github.com/SomajitDey/ipfs-chat/blob/main/LICENSE) &copy; 2021 Somajit Dey
