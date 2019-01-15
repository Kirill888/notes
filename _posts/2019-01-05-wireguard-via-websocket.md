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


## Prerequisite

I'm going to assume that you have already got WireGuard working over UDP with
Linux server in the cloud and MacOS client. If not, there are plenty of guides
on-line. For me the main stumbling block was not realising early enough that
even though WireGuard itself doesn't have a concept of server/client, the way
you configure the *server* and the way you configure *clients* is actually
somewhat different. On a client side you have a single *peer* with the public
key of your server and with `Endpoint` pointing to the domain of your server,
you also probably want to configure DNS.

Something like this example:
```
[Interface]
PrivateKey = oCkFT5ZmTJZapiCm2zZ/vNRhdVRFhKFhnkVFKRJW+2U=
Address = 10.10.0.2/24
DNS = 1.1.1.1
DNS = 8.8.8.8

[Peer]
PublicKey = xWdX6PjqZPG+So5ndzgjBa3OxSEgPA5Exi+GMLknHWA=
AllowedIPs = 0.0.0.0/0, ::0
Endpoint = yourhost.tld:51820
```

The important part above is `AllowedIPs = 0.0.0.0/0, ::0`, which tells
`wg-quick` to route all the traffic (v4 and v6) through the tunnel when setting
up the connection. Also you should avoid using `SaveConfig` option on the client
side as it will overwrite domain name of the server with IP address, which is
probably not what you want.

On the server side things might look something like this:

```
[Interface]
PrivateKey = AL7WeXT59GebMA5RLnI97fMarjKS1dnSFIDCLhxTymE=
Address = 10.10.0.1/24
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
ListenPort = 51820

[Peer]
PublicKey = kAodaLwCyX6t4Olxh0r6/ohxoIvYTQ24QIT/sijAAB0=
AllowedIPs = 10.10.0.2/32

[Peer]
...
```

Note differences in the `[Interface]` section, it includes `PostUp/PostDown`
rules to setup/tear down packet forwarding from the wireguard interface (`%i`)
to your main network interface (`ens3` in this case). There is also `ListenPort`
directive and no `DNS`.

In the `[Peer]` section, `AllowedIPs` is set to the value of `Interface.Address`
in the *client* config file, also `Endpoint` is omitted. Each peer has to have
unique address, and different from that of a server.


## UDP Tunnel

1. Head over to [wstunnel releases](https://github.com/erebe/wstunnel/releases/) and download linux version for your server
2. Compile mac version yourself, not currently available from releases page
   - Alternative is to run Linux version in docker
   - Or download binary I have [compiled](https://github.com/Kirill888/notes/releases/tag/wg-tunnel-update)

On a server we run

```
wstunnel -v -s wss://0.0.0.0/ --restrictTo 127.0.0.1:51820
```

This will listen for a TLS connection on port 443 and will only forward packets
destined to a localhost and wireguard port.

Client will run this:

```
wstunnel -v --udp --udpTimeoutSec -1 -L 127.0.0.1:51820:127.0.0.1:51820 wss://yourhost.tld/
```

This will listen on port `51820` on localhost only and forward these packets to
port `51280` on `yourhost.tld`.


### Nginx as Proxy

If you would like to run webserver on the same machine that runs `wstunnel` then
you don't want port `443` to be used solely for UDP tunnelling. With `nginx`,
websockets tunnelling is possible with a configuration similar to below:

<details><summary>Sample Nginx Config (click to expand)</summary><div markdown="1">

```nginx
server {
    server_name yourhost.tld;
    listen 443 ssl;
    # ssl config
    ssl on;
    ssl_certificate /etc/letsencrypt/live/yourhost.tld/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourhost.tld/privkey.pem;
    #  more SSL option here
    # ...

    location / {
        # .. whatever your usual site has
    }

    # Websocket reverse proxy
    #
    # Using random string here helps with reducing abuse potential
    # it's a kind of pre-shared secret between client and nginx proxy
    # On a client add `--upgradePathPrefix E7m5vGDqryd55MMP`
    location /E7m5vGDqryd55MMP/ {
        proxy_pass http://127.0.0.1:33344;
        proxy_http_version  1.1;
        proxy_set_header    Upgrade $http_upgrade;
        proxy_set_header    Connection "upgrade";
        proxy_set_header    Host $http_host;
        proxy_set_header    X-Real-IP $remote_addr;

        proxy_connect_timeout       10m;
        proxy_send_timeout          10m;
        proxy_read_timeout          90m;
        send_timeout                10m;
    }
}
```
</div></details>

You will then run `wstunnel` server on port `33344`, binding to `localhost` and
without TLS:

```bash
wstunnel --server ws://127.0.0.1:33344 --restrictTo=127.0.0.1:51820
```

You probably want to make this auto-start on your server:

<details><summary>wstunnel: Systemd Service File</summary><div markdown="1">
Copy to `/etc/systemd/system/wstunnel.service`

```ini
[Unit]
Description=Tunnel WG UDP over websocket
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/wstunnel -q --server ws://127.0.0.1:33344 --restrictTo=127.0.0.1:51820
Restart=no

[Install]
WantedBy=multi-user.target
```
then:
- `systemctl enable wstunnel`
- `systemctl start wstunnel`
</div></details>

Extra advantage you gain by using `nginx` as reverse proxy is a kind of
"authentication". Only requests made to
`https://yourhost.tld/{longish-random-string}/...` will be forwarded to
`wstunnel` server, and since we are using TLS `{longish-random-string}`
shouldn't be visible to any middleman. You also get access logs and proper TLS
(`wstunnel` only has one hard-coded certificate).


## Configure wg-quick to use UDP Tunnel

Copy `wg0.conf` into `wg1.conf` and make this change

```diff
[Peer]
PublicKey = xWdX6PjqZPG+So5ndzgjBa3OxSEgPA5Exi+GMLknHWA=
AllowedIPs = 0.0.0.0/0, ::0
- Endpoint = yourhost.tld:51820
+ Endpoint = 127.0.0.1:51820
```

If at this point you try `wg-quick up wg1`, things won't work for the following reasons:

1. `wg-quick` will route all traffic through tunnel including traffic to `yourhost.tld`
   - We can solve this by adding custom route to `yourhost.tld` in `PreUp`

2. `wg-quick` will route all traffic to `127.0.0.1` through your default
   gateway, and so won't be able to talk to `wstunnel` in the first place
   - No easy solution to this one, have to disable routing within `wg-quick`
     altogether with `Table = off` option, then setup routing manually

3. `wstunnel` will fail to connect due to DNS failure, since DNS traffic will be
   routed through a tunnel that hasn't been established yet. This might not
   happen right away due to DNS caching, but will become a problem when trying to re-connect
   - Option 1: use IP address of the server on a client side, downside no
     `vhost` routing if using `nginx`, no `TLS` verification.
   - Option 2: write current server IP to `/etc/hosts`
   - Option 3: run `dnsmasq` on client side, configure `dnsmasq` rather than `/etc/hosts`

I have written
[this script](https://github.com/Kirill888/notes/blob/b8cb9d652a2d3558f75d7ce7b7d44817f0b0f997/wireguard/scripts/wstunnel.sh)
that helps with setting up correct routes, launching `wstunnel` and updating `/etc/hosts`. To use it copy it to `/etc/wireguard/` directory and add the following to your `[Interface]` section of `wg1.conf`

```diff
[Interface]
PrivateKey = oCkFT5ZmTJZapiCm2zZ/vNRhdVRFhKFhnkVFKRJW+2U=
+ Table = off
+ PreUp = source /etc/wireguard/wstunnel.sh && pre_up %I
+ PostUp = source /etc/wireguard/wstunnel.sh && post_up %i %I
+ PostDown = source /etc/wireguard/wstunnel.sh && post_down %i %I
```

Setup script reads configuration from `/etc/wireguard/{wg_interface}.wstunnel`

```bash
REMOTE_HOST=yourhost.tld
REMOTE_PORT=51820
UPDATE_HOSTS='/etc/hosts'
# if using nginx with custom prefix for added security, configure it here
WS_PREFIX='E7m5vGDqryd55MMP'

# Can change local port of the wstunnel, don't forget to change Peer.Endpoint
#LOCAL_PORT=${REMOTE_PORT}

# If using dnsmasq can supply other file than /etc/hosts
# UPDATE_HOSTS='/usr/local/etc/dnsmasq.d/hosts/tunnels'
# Will send -HUP to dnsmasq to reload hosts
# USING_DNSMASQ=1
```

Save above to `/etc/wireguard/wg1.wstunnel` and customise. Then all you have to
do is `sudo wg-quick up wg1`. Behind the scenes `wstunnel.sh` will:

1. Obtain current IP of `yourhost.tld`
2. Update `/etc/hosts` with current IP of `yourhost.tld`
3. Add custom route to `yourhost.tld` via default gateway
4. Launch client side of `wstunnel` (as `nobody`)
5. Route all traffic through wireguard tunnel
6. Clean up when you are done `sudo wg-quick down wg1`
   - stop tunnel app
   - cleanup `/etc/hosts`

## Limitations

Currently Wi-Fi disconnects are likely to cause non-recoverable errors and will
require bringing wireguard interface down and then back up manually. This is
because route to the server is set up once and is not updated as you connect to
a new router with possibly different gateway.

I have only needed this on MacOS, Linux client will be very similar, but will
likely need some changes here and there.

## Notes on Security

It's important to use `--restrictTo=127.0.0.1:51820` option of `wstunnel` on the
server as `wstunnel` is not authenticated and you don't want to let others use
your machine as a proxy. With that option "bad guys" would only be able to send
UDP packets to a port that is already open anyway and will appear silent unless
they have your private key.

Don't run `wstunnel` as `root`, systemd file above launches `wstunnel` as `nobody`.

Make sure that `wstunnel` on a client side listens on `localhost` only, and
doesn't run as `root`.

Make sure files under `/etc/wireguard/` are accessible by `root` only,
`wg-quick` runs as `root` and so is `wstunnel.sh`, and it sources `wg1.wstunnel`
as `root` also, so make sure they are not writable by anyone except `root`.


## Notes on Debugging

- `netstat -nr -f inet` display routing table for ipv4
- Use `nc` (netcat) for testing UDP tunnel
   - server `nc -u -l 51820` (stop wireguard first)
   - client `nc -u 127.0.0.1 51820`
- Check your public ip: `curl http://api.ipify.org/`
- If using `nginx` view logs: `sudo tail -f /var/log/nginx/access.log`
- Change `-q` to `-v` on the server, then `sudo journalctl -fu wstunnel`
- `wg-quick` is just a [bash script](https://github.com/WireGuard/WireGuard/blob/master/src/tools/wg-quick/darwin.bash)


{% comment %}
These were produced with `wg genkey`/`wg pubkey`

fake server keys:
  private: AL7WeXT59GebMA5RLnI97fMarjKS1dnSFIDCLhxTymE=
  public:  xWdX6PjqZPG+So5ndzgjBa3OxSEgPA5Exi+GMLknHWA=

fake client keys
  private: oCkFT5ZmTJZapiCm2zZ/vNRhdVRFhKFhnkVFKRJW+2U=
  public:  kAodaLwCyX6t4Olxh0r6/ohxoIvYTQ24QIT/sijAAB0=
{% endcomment %}
