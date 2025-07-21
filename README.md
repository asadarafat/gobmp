<p align="left">
  <img src="https://github.com/sbezverk/gobmp/blob/master/Hudson_Go_BMP_logo.png?raw=true" width="40%" height="40%">
</p>

# goBMP (asadarafat/gobmp fork)

This is a maintained fork of the original [sbezverk/gobmp](https://github.com/sbezverk/gobmp) project with additional features and enhancements.

🚨 **Notably, this fork (`asadarafat/gobmp`) adds support for BMP over TLS (BMPS)** as defined in
[draft-hmntsharma-bmp-over-tls](https://datatracker.ietf.org/doc/draft-hmntsharma-bmp-over-tls/02/), **which is not available in the original upstream repository.**

---

## Overview

**goBMP** is an implementation of the [OpenBMP (RFC 7854)](https://datatracker.ietf.org/doc/html/rfc7854) collector protocol written in Go. It processes incoming BMP sessions and publishes the parsed BGP data to **Kafka**, **NATS**, **stdout**, or saves it to a file in **JSON format**.

goBMP can run as a standalone binary, in a container, or as a Kubernetes deployment.

---

## Features

* Receives and parses BMP messages (BGP updates).
* Supports multiple BGP AFI/SAFI NLRIs.
* Publishes to Kafka/NATS or dumps to file/stdout.
* Exposes debugging and performance metrics via `pprof`.
* Easily deployable in Kubernetes or Docker.
* [Fork-specific] Support for TLS encryption (BMPS) as per [draft-hmntsharma-bmp-over-tls](https://datatracker.ietf.org/doc/draft-hmntsharma-bmp-over-tls/02/).
* [Fork-specific] Optional TCP Authentication Option (TCP-AO) to authenticate BMP/BMPS sessions.
---

## Supported NLRI and AFI/SAFI

| NLRI                 | AFI/SAFI |
| -------------------- | -------- |
| IPv4 Unicast         | 1/1      |
| IPv4 Labeled Unicast | 1/4      |
| IPv6 Unicast         | 2/1      |
| IPv6 Labeled Unicast | 2/4      |
| VPNv4 Unicast        | 1/128    |
| VPNv6 Unicast        | 2/128    |
| Link-State           | 16388/71 |
| L2VPN (VPLS)         | 25/65    |
| L2VPN (EVPN)         | 25/70    |
| SR Policy (IPv4)     | 1/73     |
| SR Policy (IPv6)     | 2/73     |

For additional support of extensions and drafts (e.g., SRv6, Flex-Algo), see:
➡️ [Supported RFCs and Drafts](https://github.com/sbezverk/gobmp/blob/master/BMP.md)

---

## Record Format

Each parsed BMP message is published using the structure defined in the [`types.go`](https://github.com/sbezverk/gobmp/blob/master/pkg/message/types.go) file inside the `message` package.

---

## 🚧 Project Status

goBMP is a **work in progress**. Many key AFI/SAFI types and BGP-LS attributes are supported, but contributions are welcome.

See [CHANGELOG.md](https://github.com/sbezverk/gobmp/blob/master/CHANGELOG.md) for the latest updates.

---

## 🔧 Building

```bash
git clone https://github.com/sbezverk/gobmp
cd gobmp
make gobmp
```

> The binary will be available in `./bin`.

---

## ▶️ Running goBMP

### As a Binary

```bash
./bin/gobmp --config=/path/to/gobmp.yaml
```

### Key Parameters

| Parameter                         | Description                                                                      |
| --------------------------------- | -------------------------------------------------------------------------------- |
| `--config`                        | Path to config YAML file (default: `/etc/gobmp`, `$HOME/.gobmp`, or current dir) |
| `--source-port`                   | Port to listen for BMP (default: `5000`)                                         |
| `--destination-port`              | Forward BMP messages to this port (default: `5050`)                              |
| `--intercept`                     | Intercept mode (`true/false`)                                                    |
| `--dump`                          | Output destination: `file`, `console`, `kafka`, or `nats`                        |
| `--msg-file`                      | File path for storing messages (default: `/tmp/messages.json`)                   |
| `--kafka-server`                  | Kafka server address                                                             |
| `--kafka-topic-retention-time-ms` | Kafka topic retention (default: `900000`)                                        |
| `--nats-server`                   | NATS server URI                                                                  |
| `--split-af`                      | Separate IPv4/IPv6 topics (`true/false`)                                         |
| `--performance-port`              | Port for debugging via `/debug/pprof` (default: `56767`)                         |
| `--v`                             | Log level (use `--v=6` for hex debug output)                                     |

## TLS / BMPS Support (Fork Exclusive)

> 🛡️ **TLS/BMPS support is only available in this fork: [asadarafat/gobmp](https://github.com/asadarafat/gobmp)**
> The original `sbezverk/gobmp` does **not** include BMP-over-TLS capabilities.

To enable BMP over TLS, use:

```bash
--tls-port=1790
--tls-cert=/etc/gobmp/server.crt
--tls-key=/etc/gobmp/server.key
--tls-ca=/etc/gobmp/ca.crt
```

> TLS handshake and message flow fully comply with the IETF draft mentioned above.
All BMP traffic is encrypted and authenticated via mutual TLS (`tls.Conn`).

When these options are provided, goBMP additionally supports BMP over TLS (BMPS) connections by listening on the specified port using mutual TLS, as defined in [draft-hmntsharma-bmp-over-tls](https://datatracker.ietf.org/doc/draft-hmntsharma-bmp-over-tls/02/). 

Packet behaviour on the wire with BMPS enabled - a session begins with a standard mutual TLS handshake on the configured port. The server presents its certificate, the client responds with its own, and both certificates are verified against the configured certificate authorities (CAs). Once the handshake is successfully completed, the connection is fully encrypted. 

From that point onward, BMP messages are transmitted over the established TLS tunnel just as they would be over a plain TCP connection. The application reads BMP headers and payloads from a net.Conn, which is a tls.Conn in the case of BMPS. As a result, all packets on the wire are encrypted at the transport layer, ensuring both confidentiality and integrity of BMP data.

## TCP-AO Support (Fork Exclusive) - EXPERIMENTAL

goBMP can optionally protect BMP or BMPS sessions with the [TCP Authentication Option (RFC 5925)](https://datatracker.ietf.org/doc/html/rfc5925). The implementation configures TCP-AO on the listening socket and on every accepted connection using Linux's `TCP_AO_ADD_KEY` (setsockopt option `38`).

This feature requires a Linux kernel with TCP-AO support (version 6.7 or newer) and the process must have `CAP_NET_ADMIN` privileges to apply the socket options.

Enable TCP-AO with:

```bash
--tcpao-key=<shared secret>
--tcpao-send-id=1
--tcpao-recv-id=1
--tcpao-algo=hmac(sha256)
```

When configured, gobmp authenticates all BMP and BMPS packets at the TCP layer using the provided key and algorithm.

---

### Using a YAML Config File

You can define all flags in a YAML file:

```yaml
source-port: 5000
destination-port: 5050
performance-port: 56767

tls-port: 0
tls-cert: /etc/gobmp/server.crt
tls-key: /etc/gobmp/server.key
tls-ca: /etc/gobmp/ca.crt

kafka-server: localhost:9092
kafka-topic-retention-time-ms: 900000
nats-server: nats://localhost:4222
dump: kafka
msg-file: /tmp/messages.json

intercept: false
split-af: true
tcpao-key: ""
tcpao-send-id: 0
tcpao-recv-id: 0
tcpao-algo: hmac(sha256)
```

Environment variables may also be used. Use uppercase and underscores (e.g., `SOURCE_PORT`, `KAFKA_SERVER`).

---

## ☸️ Running in Kubernetes

Apply the sample deployment:

```bash
kubectl apply -f ./deployment/gobmp-standalone.yaml
```

Monitor the status:

```bash
kubectl get pods
kubectl get svc
```

Example output:

```bash
NAME                      READY   STATUS    RESTARTS   AGE
gobmp-xxx-xxx             1/1     Running   0          12h

NAME     TYPE       CLUSTER-IP   EXTERNAL-IP      PORT(S)              AGE
gobmp    ClusterIP  10.XX.X.XX   192.168.80.254   5000/TCP,56767/TCP   17h
```

> To receive BMP data externally, ensure port `5000` is exposed via a `Service`.

---

## 🐳 Running in Docker

### Quickstart using RIPE RIS Live Feed

Start goBMP collector:

```bash
sudo docker run --net=host sbezverk/gobmp --dump=console
```

Start BMP feed converter:

```bash
sudo docker run --net=host sbezverk/ris2bmp:1
```

Example output:

```json
{"action":"add", "peer_ip":"80.81.195.241", "prefix":"223.104.44.0", "origin_as":56048, ...}
```

---

## 📎 Links

* [RFC 7854 – BMP](https://datatracker.ietf.org/doc/html/rfc7854)
* [Supported Drafts & Extensions](https://github.com/sbezverk/gobmp/blob/master/BMP.md)
* [Message Format Spec (`types.go`)](https://github.com/sbezverk/gobmp/blob/master/pkg/message/types.go)
* [Deployment YAML](https://github.com/sbezverk/gobmp/tree/master/deployment)
* [Changelog](https://github.com/sbezverk/gobmp/blob/master/CHANGELOG.md)