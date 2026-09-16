# Purpose
To manage media aquisition via requests and downloads.

# VPN
Download containers should always leverage a gluetun container.
If gluetun goes down, download container should go down or self heal.

# "Don't leak traffic without VPN" — already handled, if you haven't disabled it

This isn't something depends_on provides at all. It's gluetun's own firewall/kill-switch. Because qbittorrent_client, qbittorrent_whisparr, yt-dlp-webui, and jdownloader all share gluetun's network namespace via network_mode: service:gluetun_open, gluetun's internal iptables rules apply to all of them, not just gluetun itself. As one community summary put it plainly: gluetun can be used to force other containers onto only the gluetun network, so if you disconnect from VPN for whatever reason, the other containers don't suddenly send data over non-VPN network — that really depends on the implementation, but in the case of gluetun, no data can leak when the tunnel drops. 
stormux
stormux

This is on by default (FIREWALL defaults to on) — you don't have it explicitly set in your gluetun_open environment block, which is fine since the default is the safe one. Just don't ever add FIREWALL=off chasing some other bug (I saw a GitHub discussion where someone did exactly that to fix a Traefik routing issue — it "worked" but only by turning off the exact protection you want). If you ever hit a weird connectivity issue with a downstream container, do not reach for FIREWALL=off as the fix.

So: no action needed here beyond confirming you never set that variable. This part is already solid.

# Self-healing when gluetun itself goes unhealthy or restarts — this needs something extra

This is the part depends_on: condition: service_healthy doesn't cover, and it's a real known gap in gluetun's design: gluetun does not restart itself even if it loses connection — it just flips to an unhealthy state — and separately, containers attached via network_mode: service: can become completely inaccessible if gluetun's own network stack gets disrupted, requiring a manual restart of the dependent container to recover, since Compose doesn't cascade a restart to containers sharing another container's network namespace.

depends_on only ever fires once, at startup — it's not a supervisor. To get actual self-healing you need a watchdog that monitors container health continuously and restarts on failure. The standard pattern for this is willfarrell/autoheal, driven off each container's own healthcheck::