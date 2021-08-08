# IPFS-Chat

1. Real-time peer-to-peer messaging using [IPFS pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-pubsub). Works over both internet and LAN. In-chat file-sharing is not implemented yet but will be soon.
2. The usual *Create Alias/Nick* + *Create/Join room* workflow (akin to [IRC](https://en.wikipedia.org/wiki/Internet_Relay_Chat)). Additionally, any online chatroom peer may be sent private messages (PM) that cannot be seen by the other online peers.
3. Peers are discovered using [DHT](https://docs.ipfs.io/concepts/dht/), [pubsub](https://docs.libp2p.io/concepts/publish-subscribe/) and [mDNS](https://en.wikipedia.org/wiki/Multicast_DNS) (See [Peer discovery](#peer-discovery)). No need for any rendezvous server.
4. Chat-messages are authenticated and end-to-end encrypted (See [Security](#security)).
5. Very basic terminal-based UI without any eye candy.
6. Written entirely in [Bash](https://www.gnu.org/software/bash/manual/bash.html); just a single shell-script. Apart from [go-ipfs](https://docs.ipfs.io/install/command-line/#official-distributions) and possibly `argon2`, depends only on standard GNU/Linux tools and tools that can be easily downloaded from the native package repository.

***Keywords*:** p2p; distributed; server-less; broker-less; TUI; secure; texting; file-sharing; ipfs; pubsub; privacy

## Usage

```shell
ipfs-chat -r <room> -n <nick> -d <download directory> -c <ipfs repo> -o <chat log> -wWlL
ipfs-chat -g # Generating a random room name, for when your brain can't do it
ipfs-chat -v # Version
ipfs-chat -u # Update
ipfs-chat -h # Help
```

All command-line options/flags are optional. Unobvious options are explained below.

`-c` passes the directory where `ipfs-chat` would host the IPFS node; similar to the `-c` option in the `ipfs` cli. Unlike `ipfs` cli however, the environment variable `IPFS_PATH` has no effect.

`-d` passes the directory where the files received from peers would be downloaded.

`-o` passes the file where the chat from the present session would be logged.

`-w` or `-W` denotes WAN-only mode. Local discovery is disabled. Everything happens over internet only. Uses WAN-DHT.

`-l` or `-L` denotes LAN-only mode. Peers are discovered only locally, no connection to the IPFS public network is formed over the internet (no bootstrap node, uses LAN-DHT). Saves resources when all chatroom peers are known to be present across LAN. Launches `ipfs-chat` faster when not connected to the internet.

`-u` does a manual update of `ipfs-chat`. This option is not very necessary as `ipfs-chat` auto-updates whenever there is internet connection and it is not running in LAN-only mode.

**Defaults**:

room: `Salon`

nick: `${USER}`

download directory: `${HOME}/ipfs-chat-downloads`

repo: `${HOME}/.ipfs-chat`

chat log: N/A

WAN + LAN

**Multiple-instances**:

Multiple instances of `ipfs-chat` may be run for accessing different chatrooms concurrently. This may be done in 2 ways:

1. Provide a separate IPFS repository with the `-c` flag for every instance. Each instance then runs its own IPFS node independent of the others. Example: 

   ```shell
   ipfs-chat -c '/tmp/repo1' -r chatroom1 -n nick1 # In one terminal
   ipfs-chat -c '/tmp/repo2' -r chatroom2 -n nick2 # In another terminal
   ```

2. If you are okay with using the same *nick* for every chatroom, then it is much more efficient to run the multiple instances on the same IPFS node, i.e. with the same IPFS repository. Example: 

   ```shell
   ipfs-chat [-c common-repo] -r chatroom1 -n myNick # In one terminal
   ipfs-chat [-c common-repo] -r chatroom2 -n myNick # In another terminal
   ```

## Snapshot

![ipfs-chat_snapshot](./screenshot.png)

## Testing

You can test `ipfs-chat` by running multiple instances on your computer, or running one instance on your machine and another instance on another computer or a free cloud shell such as [Google's](https://cloud.google.com/shell). Simply use different nicknames and join the same chatroom. Example: Connect to the internet and do the following.

```shell
ipfs-chat -b -c '/tmp/ipfs-chat' -n 'partner' # In one terminal
ipfs-chat -b # In another terminal, computer or cloud shell
# -b above enables bandwidth stats. Drop it if you are not interested in those stats.
```

Test further. Disconnect from the internet and run the above two instances on your local machine or two LAN-connected hosts with the `-l` or `-L` flag.

## Changing terminal window size

Should you ever need to change the size of the terminal `ipfs-chat` is running on, you would find that it messes up the screen. Don't panic. Simply press ENTER or any of the buttons on the `ipfs-chat` screen and everything will be fixed.

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

All general messages are encrypted with a symmetric key (AES128) derived from the room name using HMAC.

All private messages are encrypted with a public key (ECDH/cv25519) belonging to the recipient.

The pubsub topics are separate HMAC keys derived from the room name.

Therefore, the public network, that mediates the pubsub and passes the messages along, never knows the actual room name and hence, the encryption key.

Before deriving the keys, the room name is itself hashed using a memory- and CPU-hard password-hashing algorithm ([Argon2](https://github.com/P-H-C/phc-winner-argon2)) [Not implemented yet].

## Messaging

Every peer publishes its nick and a self-signed PGP public key (primary-key EDDSA/ed25519 for signing + subkey ECDH/cv25519 for encryption) under its peer ID over IPNS at the start of every session. This authenticates its claim over the nick and the public key.

After discovering a peer, other peers resolve its IPNS entry and caches its nick and pubkey for use throughout the session.

For general messaging, a peer signs its message with its private key and encrypts with the aforementioned symmetric key derived from the room name. For private messaging, the message is encrypted with the recipient's public key instead. The whole encrypted object is encoded in base64 and published over pubsub along with the sender's peer ID.

Other peers receive this over pubsub, decrypt the message and verify the signature. If everything is ok, they then display the message in their UI against the sender's nick, peer ID and timestamp.

[IPNS-pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipns-pubsub) is enabled to aid in resolving IPNS names as quickly as possible, and reducing the dependence on DHT.

## Efficiency

[IPFS usually consumes a lot of bandwidth](https://github.com/ipfs/go-ipfs/issues/3429). To make it more efficient, all unnecessary connections are closed at regular intervals: The node maintains a list of all chatroom peers seen or discovered in a session. All other connections, except connections to the relays the node itself or its chatroom peers are connected to, are culled on a frequent basis.

New connections are always being formed to nodes within the general IPFS network. The interval between consecutive cullings is large enough to gather sufficient number of nodes for performing DHT operations such as providing the rendezvous file, finding other providers and finding the multiaddresses of those providers. Once these operations are finished, the culling is performed.

Connections to the chatroom peers are never closed.

To achieve this, `ipfs-chat` uses its own connection manager and does not use the [basic connection manager](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#basic-connection-manager) that ships with `go-ipfs`.

The go-ipfs config file has been tuned to reduce resource (CPU/bandwidth/memory) consumption.

WAN-only and LAN-only modes are available (see [Usage](#usage)) to further optimize resource consumption under different situations.

**Tip**: To show bandwidth usage by the node at the end of a session, launch `ipfs-chat` with the `-b` flag. Note that this shows the cumulative bandwidth consumption by all `ipfs-chat` instances using the same node at the same time.

## Warning

`ipfs-chat` does a `pkill -s0` while exiting which closes all processes in the current terminal session. Fixing this limitation is in the TODO list. For the time being, however, it is advised that `ipfs-chat` be opened in a new terminal whenever possible.

## Future directions

1. Secure file sharing. Files to be shared can simply be dragged and dropped in the text-input area. Support for WSL. Download limit (byte size *int*) can be specified with option `-D` to control bandwidth consumption - the user would be prompted before larger files downloads. `-D 0` : prompt always; `-D n` : prompt before downloading files larger than *n* bytes; `-D -n` : prompt never.
2. Using Argon2 for more security (See [Security](#security)).
4. Command-line option for manual update: `-u`. Try auto-update in background always and prompt user when new update is available.
5. Detect and block malicious peers. All direct connections to blocked peers are culled. Users can also block (and unblock later) specific nicks (regex pattern), peer IDs. While blocking users can opt for - 1. Block permanently; 2. For present session only. **TBD**: News of this blocking (who blocked whom and when) may or may not be published over pubsub for other peers to see and decide for themselves.
9. Secure directory transfer
10. Offline mode such that even if a peer goes offline, it can obtain the missed messages when it comes back online [This may well be beyond my capabilities]. Once [`orbit-db-cli`](https://github.com/orbitdb/orbit-db-cli) matures, it might help achieve this. Random idea: Online peers publish CIDs of time-based logs at regular intervals over pubsub. Logs are directories containing chats - one message in one file. Even if logs of two peers don't match exactly, they will have many files in common, thus achieving major deduplication and also helping availability across the ipfs-chat network.

## Bug-reports and Feedbacks

Post at [issues](https://github.com/SomajitDey/ipfs-chat/issues) and [discussion](https://github.com/SomajitDey/ipfs-chat/discussions), or [write to me](mailto://hereistitan@gmail.com).

------

###### [GNU GPL v3-or-later](https://github.com/SomajitDey/ipfs-chat/blob/main/LICENSE) &copy; 2021 Somajit Dey
