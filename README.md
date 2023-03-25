# natpunch-go

This is a [NAT hole punching](https://en.wikipedia.org/wiki/UDP_hole_punching) tool designed for creating Wireguard mesh networks. It was inspired by [Tailscale](https://www.tailscale.com/) and informed by [this example](https://git.zx2c4.com/wireguard-tools/tree/contrib/nat-hole-punching/).

This tools allows you to connect to other Wireguard peers from behind a NAT using a server for ip and port discovery. I'd recommend putting this tool behind a Wireguard connection with the server as there's no authentication yet. Also, the client is Linux only.

## Example:

Hey folks,

Many have asked about NAT traversal and hole punching, and I've
explained that since WireGuard is just usual UDP, you can use any of
the typical techniques. Not satisfied with that, people have demanded
examples. So, I coded up a very short proof of concept of the most
basic hole punching mechanism that integrates with WireGuard. Note:
this is PoC/example code, and as such it has a number of security
problems and thus should not be used in the real world (distros: do
NOT compile and install this); however, it suffices as a nice
illustration of the underlying concepts.

Voila: https://git.zx2c4.com/WireGuard/tree/contrib/examples/nat-hole-punching

Compile with:
$ gcc nat-punch-client.c -o client -lresolv

$ gcc nat-punch-server.c -o server

On the server, simply run "./server" and make sure UDP:49918 is open.
Then, for each client, configure the various peers of a wireguard
interface, as you would normally, except you can omit the endpoint.
That's what the hole punching client adds for us. For each client,
simply run:

    # ./client demo.wireguard.io wg0

It will run until it's received the correct path to all of the peers
of wg0. Replace demo.wiregaurd.io with your own server, or use (but do
not abuse!) the demo instance running on the demo box.

Demo:

# wg show wg0
interface: wg0
  public key: bqodvMJALCmDU32kcjA/cG4ZMTaX/IihN2NruSGhDXo=
  private key: (hidden)
  listening port: 25586

peer: aQoADFvA1zZmCs40G/gp1jDCEgRVyWwSWT463VIxXCQ=
  allowed ips: 192.168.88.2/32

peer: T3TEQxBh/+4sxuIOUhc2T8VVDhD8JBoM/V3/v72NNDI=
  allowed ips: 192.168.88.3/32

# ./client demo.wireguard.io wg0
[+] Requesting IP and port of
aQoADFvA1zZmCs40G/gp1jDCEgRVyWwSWT463VIxXCQ=: 65.182.136.126:999
[+] Requesting IP and port of
T3TEQxBh/+4sxuIOUhc2T8VVDhD8JBoM/V3/v72NNDI=: 88.190.101.12:51821

# wg show wg0
interface: wg0
  public key: bqodvMJALCmDU32kcjA/cG4ZMTaX/IihN2NruSGhDXo=
  private key: (hidden)
  listening port: 25586

peer: aQoADFvA1zZmCs40G/gp1jDCEgRVyWwSWT463VIxXCQ=
  endpoint: 65.182.136.126:999
  allowed ips: 192.168.88.2/32
  latest handshake: 36 seconds ago
  bandwidth: 110 B received, 290 B sent
  persistent keepalive: every 25 seconds

peer: T3TEQxBh/+4sxuIOUhc2T8VVDhD8JBoM/V3/v72NNDI=
  endpoint: 88.190.101.12:51821
  allowed ips: 192.168.88.3/32
  latest handshake: 36 seconds ago
  bandwidth: 110 B received, 290 B sent
  persistent keepalive: every 25 seconds


Enjoy!
Jason



## Usage

The client cycles through each peer on the interface until they are all resolved. Requires root to run due to raw socket usage.
```
Usage: ./client [OPTION]... WIREGUARD_INTERFACE SERVER_HOSTNAME:PORT SERVER_PUBKEY
Flags:
  -c, --continuous=false: continuously resolve peers after they've already been resolved
  -d, --delay=2: time to wait between retries (in seconds)
Example:
    ./client wg0 demo.wireguard.com:12345 1rwvlEQkF6vL4jA1gRzlTM7I3tuZHtdq8qkLMwBs8Uw=
```

The server associates each pubkey to an ip and a port. Doesn't require root to run.
```
Usage: ./server PORT [PRIVATE_KEY]
```

## Why

I want to have a VPN so that I can access all of my devices even when I'm out of the house. Unfortunately, using a traditional client-server model creates additional latency. Peer-to-peer connections are ideal for each client, however many of the devices are behind a NAT. The motivation for this tool was to allow p2p Wireguard connections through a NAT.

## How

UDP NAT hole punching allows us to open a connection when both clients are behind a NAT. Modern NATs may employ source port randomization, which means that clients cannot predict which port to connect to in order to punch through that NAT. We need a way to discover the port that the client is using.

We use a publicly facing server in order to determine the ip address and port of each client. Each client can connect using the same port as Wireguard (by spoofing the source port using raw sockets.) and its ip and port will be recorderd. It can also request the ip and port of another client using their public key. This breaks source port randomization and allows NAT punching on [every NAT type except symmetric](https://en.wikipedia.org/wiki/Network_address_translation#Methods_of_translation), as symmetric NATs may use a different external ip and port for each connection.

Once each client gets the ip and port, they simply set the peer's endpoint to the ip and port it learned about and set a persistent keepalive to start the packet flow. With the keepalive, the peers will keep trying to contact each other which will create the hole on both sides and maintain the connection.

## Why Go?

Go has great support for [raw sockets](https://pkg.go.dev/golang.org/x/net/ipv4?tab=doc), [packet filtering](https://pkg.go.dev/golang.org/x/net/bpf?tab=doc), and [packet construction/deconstruction](https://pkg.go.dev/github.com/google/gopacket?tab=doc). I plan on rewriting this in Rust one day but Go's library support is too good to pass up.
