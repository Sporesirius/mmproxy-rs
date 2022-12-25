# mmproxy-rs

A Rust implementation of MMProxy! 🚀

## Rationale

Many previous implementations only support [PROXY Protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) for either TCP or UDP, whereas this version supports both TCP and UDP.

Another reason to choose mmproxy-rs may be if you want to avoid interference from Garbage Collection pauses, which is what originally triggered the re-write from the amazing [go-mmproxy](https://github.com/path-network/go-mmproxy).

## Features

- [x] TCP - Accepts PROXY Protocol enabled requests from [Nginx](https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/#proxy-protocol-for-a-tcp-connection-to-an-upstream), [HAProxy](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)
- [x] UDP - Accepts PROXY Protocol enabled requests from [udppp](https://github.com/b23r0/udppp)
- [x] No Garbage Collection pauses

## Usage

```
$ mmproxy -h
Usage: mmproxy [-h] [options]

Options:
  -h, --help              Prints the help string.
  -4, --ipv4 <addr>       Address to which IPv4 traffic will be forwarded to.
                          (default: "127.0.0.1:443")
  -6, --ipv6 <addr>       Address to which IPv6 traffic will be forwarded to.
                          (default: "[::1]:443")

  -a, --allowed-subnets <path>
                          Path to a file that contains allowed subnets of the
                          proxy servers.

  -c, --close-after <n>   Number of seconds after which UDP socket will be
                          cleaned up. (default: 60)

  -l, --listen-addr <string>
                          Address the proxy listens on. (default:
                          "0.0.0.0:8443")

  --listeners <n>         Number of listener sockets that will be opened for the
                          listen address. (Linux 3.9+) (default: 1)
  -p, --protocol <p>      Protocol that will be proxied: tcp, udp. (default:
                          tcp)
  -m, --mark <n>          The mark that will be set on outbound packets.
                          (default: 0)
```

### Example

:warning: needs root permissions.

```terminal
$ address=X.X.X.X # get this via "ip addr" command - don't use 0.0.0.0!
$ bind_port=8080
$ upstream_port=8081
$ sudo ip rule add from 127.0.0.1/8 iif lo table 123
$ sudo ip route add local 0.0.0.0/0 dev lo table 123
$ sudo mmproxy  -l $address:$bind_port -4 127.0.0.1:$upstream_port -p udp
```

## Benchmarking

Tests were run on a `Linux 6.0.12-arch1-1` box with an AMD Ryzen 5 5600H @ 3.3GHz (12 logical cores).

### TCP mode

#### Setup

[bpf-echo](https://github.com/path-network/bpf-echo) server simulated the upstream service that the proxy sent traffic to. The traffic was generated using [tcpkali](https://github.com/satori-com/tcpkali).

The following command was used to generate load:
```
tcpkali -c 50 -T 10s -e1 'PROXY TCP4 127.0.0.1 127.0.0.1 \{connection.uid} 25578\r\n' -m 'PING\r\n' 127.0.0.1:1122
```
which specifies 50 connections, a runtime of 10 seconds, sending a PROXYv1 header for each connection, and using the message `PING\r\n` over TCP.

#### Results

|            | ↓ Mbps    | ↑ Mbps    | ↓ pkt/s   | ↑ pkt/s   |
| ---------- | --------- | --------- | --------- | --------- |
| no-proxy   | 34662.036 | 53945.378 | 3173626.3 | 4630027.6 |
| go-mmproxy | 27527.743 | 44128.818 | 2520408.4 | 3787491.3 |
| mmproxy-rs | 27228.169 | 50173.384 | 2492924.1 | 4306284.7 |

### UDP Mode

#### Setup

```
iperf client -> udppp -> mmproxy-rs/go-mmproxy -> iperf server
```

```
$ udpp -m 1 -l 25578 -r 25577 -h "127.0.0.1" -b "127.0.0.1" -p           // udppp
# mmproxy -l "127.0.0.1:25577" -4 "127.0.0.1:1122" -p udp -c 1           // mmproxy-rs
# mmproxy -l "127.0.0.1:25577" -4 "127.0.0.1:1122" -p udp -close-after 1 // go-mmproxy
$ iperf -sup 1122                                                        // iperf server
$ iperf -c 127.0.0.1 -p 25578 -Rub 10G                                   // iperf client
```

#### Results

|            | transfer    | bandwidth      |
|------------|-------------|----------------|
| no-proxy   | 6.31 GBytes | 5.42 Gbits/sec |
| go-mmproxy | 3.13 GBytes | 2.69 Gbits/sec |
| mmproxy-rs | 3.70 GBytes | 3.18 Gbits/sec |

The iperf test was run in reverse mode, with the server sending data to the client. The results suggest that mmproxy-rs has higher throughput from upstream to downstream compared to go-mmproxy.

## Acknowledgements and References

- https://blog.cloudflare.com/mmproxy-creative-way-of-preserving-client-ips-in-spectrum/
- https://github.com/cloudflare/mmproxy
- https://github.com/b23r0/udppp
- https://github.com/path-network/go-mmproxy