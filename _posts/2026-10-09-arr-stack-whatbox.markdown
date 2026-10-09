---
layout: post
title:  "The *arr Stack on a Seedbox"
date:   2026-10-09 13:48:00 -0700
categories: seedbox arr plex
---
## Introduction

This is a write up of the media automation stack running on my seedbox.

The stack is the usual *arr lineup. [Sonarr](https://sonarr.tv) for TV, [Radarr](https://radarr.video) for movies, [Lidarr](https://lidarr.audio) for music, [Prowlarr](https://prowlarr.com) to feed all three from one indexer list, [Bazarr](https://www.bazarr.media) for subtitles, [SABnzbd](https://sabnzbd.org) to do the downloading, Seerr to take requests, and [Plex](https://www.plex.tv) to play it all back.

## Overview

1. The two tiers of app, and why it matters.
2. What each app is actually for.
3. What's installed and on which port.
4. How a request actually flows through the stack.
5. Where each app keeps its config.
6. Keeping apps running without systemd.

## The two tiers of the apps.

There are two completely different ways an app ends up running here.

**Hosting managed.** The control panel installs these system wide. The binaries live in `/usr/bin` or `/usr/share`, the per user config lives in `~/.config`, and a daemon called `helmd` starts them at boot. You never write a unit file or a cron entry. You click a button.

**Self installed.** Anything the panel doesn't offer, or any version you want to pin yourself, you unpack into `$HOME` and run by hand. Nothing starts these for you at boot, and nothing restarts them when they die. That's the problem the back half of this post solves.

You can tell the two apart instantly from `ps`.

```bash
myee1@hostname ~> ps -u myee1 -o etime,comm,args --sort=etime | cut -c1-120
 7-17:07:58 python  /usr/lib/python-exec/python3.12/python -u /usr/share/bazarr/bazarr/main.py -c /home/myee1/.config/bazarr
20-04:44:11 prowlarr /usr/bin/prowlarr
20-04:44:16 radarr  /usr/bin/radarr
20-05:15:02 sonarr  /usr/bin/sonarr
28-03:53:24 SABnzbd.py /usr/bin/python3.12 -OO /usr/share/sabnzbd/SABnzbd.py --daemon --browser 0 --inet_exposure 4
40-03:23:07 node    node dist/index.js
41-14:39:48 Lidarr  /home/myee1/Lidarr/Lidarr
```

Anything rooted at `/usr` belongs to the hosting service. Anything rooted at `/home/myee1` is mine.

## What each app is for.

The naming convention is not helpful when you're starting out. Five apps ending in *arr, all with near identical interfaces, and nothing about the name tells you which does what. Here's the short version.

**Sonarr** is for TV. You give it a series and it takes over from there: it knows the full episode list from TheTVDB, works out which episodes you're missing, watches for them to appear, grabs them, then renames and files the result into the right show and season folder. The ongoing part is the point. Once a series is added, new episodes land on their own as they air.

**Radarr** is the same idea for movies. It was literally forked from Sonarr, which is why the two look identical. The differences are the ones you'd expect from the subject matter: no episodes to track, but it does have a notion of how early to start looking, since a film can be announced a long time before anything is actually out there to find.

**Lidarr** is the same idea again for music, tracking artists instead of series and pulling its metadata from MusicBrainz.

**Prowlarr** is the one that isn't like the others. It doesn't manage any media at all. It's an indexer manager: the single place where every indexer and tracker is configured, which then pushes that list out to Sonarr, Radarr and Lidarr over their APIs. That's the whole value. Without it you configure every indexer three times and update them three times. With it you add an indexer once and all three apps have it a second later. It also proxies searches, so when Sonarr goes hunting for an episode the query actually runs through Prowlarr.

**Bazarr** handles subtitles, and it's a companion rather than a peer. It doesn't search for media or talk to indexers. It connects to Sonarr and Radarr, reads what they have, works out which of it is missing subtitles in the languages you want, and fetches them from whichever subtitle providers you've enabled. It runs continuously because subtitles for a new release often don't exist on day one, so it keeps checking back.

**Seerr** is the front door, and the one app here not named *arr. It's a request manager: a browsable catalogue where someone picks a film or show and asks for it, with approval and quota rules in front. On approval it hands the request to Sonarr or Radarr via their APIs and then reports status back, so the person who asked can see it move from requested to downloading to available without ever touching the *arr apps themselves. It's the piece that makes the stack usable by someone who isn't you.

The short version of the short version: Prowlarr finds things, Sonarr, Radarr and Lidarr decide what to look for and where it goes, Bazarr adds subtitles, Seerr is how people ask.

## What's installed and on which port.

The hosting service assigns each account its own high ports, so these numbers are specific to this box.

Panel managed:

| App | Port | Config |
| --- | --- | --- |
| Sonarr | 24279 | `~/.config/Sonarr/config.xml` |
| Radarr | 25492 | `~/.config/Radarr/config.xml` |
| Prowlarr | 17665 | `~/.config/Prowlarr/config.xml` |
| Bazarr | 26096 (127.0.0.1 only) | `~/.config/bazarr/config/config.yaml` |
| SABnzbd | 12011 | `~/.config/sabnzbd/sabnzbd.conf` |

Self installed:

| App | Port | How it's installed |
| --- | --- | --- |
| Lidarr | 16751 | tarball unpacked to `~/Lidarr` |
| Seerr | 11304 | source checkout in `~/seerr`, built with pnpm |
| Plex | 32400 | `~/Library/Plex` |

Lidarr is self installed because I wanted a specific build, `Lidarr.master.2.4.3.4248.linux-core-x64`. The tarball is still sitting in `$HOME` next to the unpacked directory. Seerr runs straight from a git checkout rather than a release, which is why the process is `node dist/index.js` under pnpm rather than anything more official looking.

## How a request flows.

Seerr is the front door. Someone asks for a show, and from there:

1. **Seerr** hands the request to Sonarr or Radarr over HTTPS. It reaches them through a reverse proxy.
2. **Sonarr / Radarr** work out exactly what's missing and ask **Prowlarr** for releases, which runs the search across every indexer it has configured and hands back the results.
3. **SABnzbd** does the actual downloading over NNTP, from a couple of Usenet providers on port 563.
4. Finished files land in `~/files/completed`. The rest of the tree is working space.

   ```bash
   myee1@hostname ~> ls -1 ~/files
   completed
   intermediate
   music
   nzb
   queue
   tmp
   ```

5. **Plex** serves the result, and **Bazarr** comes along behind to fill in subtitles.

Seerr's own scheduler keeps the picture current, on a fairly busy cadence:

```
plex-recently-added-scan   0 */5 * * * *
plex-watchlist-sync        0 */3 * * * *
radarr-scan                0 0 4 * * *
sonarr-scan                0 30 4 * * *
availability-sync          0 0 5 * * *
download-sync              0 * * * * *
```

## Where each app keeps its config.

The three .NET apps all use the same `config.xml` shape. Here's Sonarr's, with the key taken out:

```xml
<Config>
  <Port>24279</Port>
  <UrlBase></UrlBase>
  <BindAddress>*</BindAddress>
  <SslPort>23156</SslPort>
  <EnableSsl>False</EnableSsl>
  <ApiKey>REDACTED</ApiKey>
  <AuthenticationMethod>Forms</AuthenticationMethod>
  <LogLevel>Info</LogLevel>
  <Branch>master</Branch>
  <LaunchBrowser>False</LaunchBrowser>
  <UpdateAutomatically>False</UpdateAutomatically>
  <UpdateMechanism>BuiltIn</UpdateMechanism>
  <InstanceName>Sonarr</InstanceName>
  <AuthenticationRequired>Enabled</AuthenticationRequired>
</Config>
```

Radarr, Prowlarr and Lidarr are identical apart from the ports and `InstanceName`. Two things worth noting. `UpdateAutomatically` is `False` on all of them, because on a shared host you want updates to happen when you're watching. And `AuthenticationRequired` is `Enabled` with `Forms`, which matters a lot given `BindAddress` is `*`.

Bazarr uses YAML instead, and is the one app bound to loopback only:

```yaml
general:
  base_url: ''
  ip: 127.0.0.1
  port: 26096
plex:
  ip: ''
  port: 32400
```

Seerr keeps everything in one `~/seerr/config/settings.json`, including its own API key, its session secret, VAPID keys for push notifications, the Plex token and a copy of the Sonarr and Radarr API keys. It is comfortably the most secret dense file on the box. If you ever paste a Seerr config anywhere, strip it hard.

SABnzbd's config is `~/.config/sabnzbd/sabnzbd.conf`, holding the Usenet account passwords along with the usual:

```ini
host = ::
port = 12011
inet_exposure = 4
url_base = ""
```
