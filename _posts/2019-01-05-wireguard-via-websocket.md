---
layout: post
title: Tunnel WireGuard via Websockets
tags:
  - vpn
  - wireguard
---

Setting up WireGuard vpn to work in restricted networks that block UDP traffic.

## Basic Idea

- Run [wstunnel](https://github.com/erebe/wstunnel) to tunnel UDP traffic to vpn server
- Configure local `wg-quick` to use `localhost` as endpoint

Sounds easy, and it's not hard, but there are some gotchas to be aware off:

- Have to do your own routing setup
- Possible issues with DNS when `wstunnel` needs to re-connect

client side:

```
[Interface]
PrivateKey = oCkFT5ZmTJZapiCm2zZ/vNRhdVRFhKFhnkVFKRJW+2U=
Address = 10.10.0.2/24

[Peer]
PublicKey = xWdX6PjqZPG+So5ndzgjBa3OxSEgPA5Exi+GMLknHWA=
AllowedIPs = 0.0.0.0/0, ::0
Endpoint = yourhost.com:51820
```

server side:

```
[Interface]
PrivateKey = AL7WeXT59GebMA5RLnI97fMarjKS1dnSFIDCLhxTymE=
Address = 10.10.0.1/24
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
ListenPort = 51820
SaveConfig = true

[Peer]
PublicKey = kAodaLwCyX6t4Olxh0r6/ohxoIvYTQ24QIT/sijAAB0=
AllowedIPs = 10.10.0.2/0
```

<!--

fake server keys:
  private: AL7WeXT59GebMA5RLnI97fMarjKS1dnSFIDCLhxTymE=
  public:  xWdX6PjqZPG+So5ndzgjBa3OxSEgPA5Exi+GMLknHWA=

fake client keys
  private: oCkFT5ZmTJZapiCm2zZ/vNRhdVRFhKFhnkVFKRJW+2U=
  public:  kAodaLwCyX6t4Olxh0r6/ohxoIvYTQ24QIT/sijAAB0=
-->
