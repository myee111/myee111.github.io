---
layout: post
title:  "Add the Rustdesk Relay Server"
date:   2026-10-09 12:51:00 -0700
categories: podman rustdesk
---

## Introduction

A followup to [How to install Rustdesk]({% post_url 2025-12-17-rustdesk %}). That setup only runs `hbbs`, so there is no relay to fall back on when two peers can't reach each other directly. This adds `hbbr`.

1. Write the hbbr systemd unit file.
1. Change the volume label from `:Z` to `:z`.
1. Reload systemd and start the service.
1. Check that both services are running.

## Why bother with hbbr.

The rustdesk server is two separate daemons.

`hbbs` is the rendezvous server. Clients register their ID with it, and when you connect to a peer it introduces the two machines and coordinates NAT hole punching. If the hole punching works, the session goes straight between the two machines and the server never sees the traffic.

`hbbr` is the relay. It only matters when the hole punching fails, which happens behind symmetric NAT, a strict office firewall, or CGNAT on a phone hotspot. In that case both peers make ordinary outbound connections to port 21117 and `hbbr` forwards bytes between them. It is a dumb pipe and the traffic stays end to end encrypted, so the relay can't read it.

Without `hbbr`, a session that can't go direct has nowhere to fall back to and just fails to connect. Direct connections are still preferred either way, so nothing that already worked changes.

No firewall changes are needed. The [previous post]({% post_url 2025-12-17-rustdesk %}) already opened 21117/tcp and 21119/tcp.

## Write the hbbr systemd unit file.

In `~/.config/containers/systemd/hbbr.container`, write the following configuration.

```ini
[Container]
AutoUpdate=registry
Image=ghcr.io/rustdesk/rustdesk-server:latest
Exec=hbbr
Volume=rustdesk.volume:/root:z
Network=host
PublishPort=21117:21117/tcp
PublishPort=21119:21119/tcp

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Same image and same volume as `hbbs`, just `Exec=hbbr` instead. Both daemons have to read the same `id_ed25519` keypair, which is why they share one volume.

## Change the volume label from `:Z` to `:z`.

This is the part that will bite you. The original `rustdesk.container` mounts the volume with `:Z`:

```ini
Volume=rustdesk.volume:/root:Z
```

Uppercase `:Z` tells podman to relabel the volume with a *private* SELinux MCS category belonging to one container. You can see the category on the files:

```bash
myee@protectli1 ~> ls -lZ .local/share/containers/storage/volumes/systemd-rustdesk/_data/
-rw-r--r--. 1 myee myee system_u:object_r:container_file_t:s0:c157,c400   24576 Oct  9 11:07 db_v2.sqlite3
-rw-r--r--. 1 myee myee system_u:object_r:container_file_t:s0:c157,c400      88 Dec 17  2025 id_ed25519
-rw-r--r--. 1 myee myee system_u:object_r:container_file_t:s0:c157,c400      44 Dec 17  2025 id_ed25519.pub
```

Those `c157,c400` categories match the `hbbs` process label. If you mount the same volume into `hbbr` with `:Z` as well, podman relabels everything with *hbbr's* categories instead, and `hbbs` loses access to its own keypair and database the next time it starts.

Lowercase `:z` marks the volume as shared, so change it in **both** files.

```bash
sed -i 's|Volume=rustdesk.volume:/root:Z|Volume=rustdesk.volume:/root:z|' ~/.config/containers/systemd/rustdesk.container
```

The files end up with no MCS category, just `s0`, and both containers can read them.

```bash
myee@protectli1 ~> ls -lZ .local/share/containers/storage/volumes/systemd-rustdesk/_data/
-rw-r--r--. 1 myee myee system_u:object_r:container_file_t:s0   24576 Oct  9 11:43 db_v2.sqlite3
-rw-r--r--. 1 myee myee system_u:object_r:container_file_t:s0      88 Dec 17  2025 id_ed25519
-rw-r--r--. 1 myee myee system_u:object_r:container_file_t:s0      44 Dec 17  2025 id_ed25519.pub
```

The tradeoff is that the two containers no longer have SELinux isolation from each other on that volume, which doesn't mean much when they are two halves of the same service.

## Reload systemd and start the service.

```bash
systemctl --user daemon-reload
systemctl --user restart rustdesk.service
systemctl --user start hbbr.service
```

The `rustdesk.service` restart is needed so `hbbs` picks up the relabelled volume. It's about five seconds of downtime.

There is nothing to `enable`. Quadlet handles `WantedBy=default.target` from the generated unit, which you can confirm with:

```bash
myee@protectli1 ~> systemctl --user list-dependencies default.target | grep -E 'hbbr|rustdesk'
● ├─hbbr.service
● ├─rustdesk.service
```

## Check that both services are running.

{% raw %}
```bash
myee@protectli1 ~> podman ps --format '{{.Names}}\t{{.Status}}\t{{.Command}}'
systemd-rustdesk	Up 29 seconds	hbbs
systemd-hbbr	Up 24 seconds	hbbr
```
{% endraw %}

All five ports should be listening, 21115/21116/21118 from `hbbs` and 21117/21119 from `hbbr`.

```bash
myee@protectli1 ~> ss -lntu | grep -E '2111[4-9]' | awk '{print $1, $5}' | sort -u
tcp *:21115
tcp *:21116
tcp *:21117
tcp *:21118
tcp *:21119
udp *:21116
```

The `hbbr` log should show it listening on the relay ports.

```bash
myee@protectli1 ~> podman logs --tail 5 systemd-hbbr
[src/relay_server.rs:82] Listening on tcp :21117
[src/relay_server.rs:84] Listening on websocket :21119
[src/relay_server.rs:87] Start
[src/relay_server.rs:125] LIMIT_SPEED: 32Mb/s
[src/relay_server.rs:136] TOTAL_BANDWIDTH: 1024Mb/s
```

Worth checking for SELinux denials too, since that was the risky part. This should come back empty.

```bash
sudo ausearch -m AVC -ts recent
```

## Notes.

No client reconfiguration is needed. `hbbs` hands out the relay address itself, which is why `relay-servers=[]` in its log is fine when `hbbr` runs on the same host. You only need the `-R` option if the relay lives somewhere else.

Relayed sessions use home upload bandwidth in both directions since the box sits in the middle. `hbbr` caps this at 128Mb/s per session and 1024Mb/s total by default.

If everything is on the LAN this path never gets used. It earns its keep connecting in from a hotel or a phone hotspot. `podman logs systemd-hbbr` shows relayed sessions if you want to know whether it's doing anything.
