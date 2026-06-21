<!-- #ZEROPS_EXTRACT_START:name# -->
Home Assistant Proxy
<!-- #ZEROPS_EXTRACT_END:name# -->

<!-- #ZEROPS_EXTRACT_START:shape# -->
app
<!-- #ZEROPS_EXTRACT_END:shape# -->

<!-- #ZEROPS_EXTRACT_START:intro# -->
Expose a self-hosted Home Assistant instance to the internet — with a real domain and HTTPS — without opening any ports on
your home router.
<!-- #ZEROPS_EXTRACT_END:intro# -->

<!-- #ZEROPS_EXTRACT_START:description# -->
Home Assistant usually lives on your home network, unreachable from the outside. This recipe runs a small Nginx reverse
proxy on Zerops that connects back to Home Assistant over a WireGuard VPN: Home Assistant joins the Zerops private network
as a WireGuard client, and the Zerops proxy forwards all traffic to it. You get a public, TLS-terminated address on your
own domain, while your home network stays completely closed.

This recipe does not support one-click import — it needs a WireGuard tunnel between Zerops and your Home Assistant
instance. After deploying, follow the setup guide to generate the VPN config, install and configure the WireGuard Client
add-on in Home Assistant, and point the proxy at your instance.
<!-- #ZEROPS_EXTRACT_END:description# -->

<!-- #ZEROPS_EXTRACT_START:features# -->
- Public HTTPS access to Home Assistant without port forwarding or a static home IP
- Encrypted WireGuard tunnel between Zerops and your home network
- Bring your own custom domain with automatic TLS
- Lightweight single-container Nginx proxy — cheap to run for personal use
- Home network stays closed; only the outbound VPN connection is exposed
<!-- #ZEROPS_EXTRACT_END:features# -->

<!-- #ZEROPS_EXTRACT_START:takeover-guide# -->
This recipe requires manual setup — full step-by-step instructions (with screenshots) are in the repository README. In
short:

1. **Generate a WireGuard config** for your Zerops project with `zcli vpn config --output -`.
2. **Install the WireGuard Client add-on** in Home Assistant (from the `bigmoby/hassio-repository-addon` repository) and
   fill in the interface and peer values from the generated config.
3. **Trust the proxy** in Home Assistant's `configuration.yaml` (`use_x_forwarded_for` + `trusted_proxies` covering the
   VPN ranges) and fully restart Home Assistant — a config reload is not enough.
4. **Set `HA_ADDRESS`** on the `ha` service to your Home Assistant's VPN address and port (e.g. `10.2.164.3:8123`).
5. Verify on the Zerops preview domain, then add your custom domain and disable the preview domain.
<!-- #ZEROPS_EXTRACT_END:takeover-guide# -->

<!-- #ZEROPS_EXTRACT_START:knowledge-base# -->
### Architecture

```
Browser ──HTTPS──> Zerops (Nginx proxy) ──WireGuard tunnel──> Home Assistant (at home)
```

A single `ubuntu/nginx` service terminates TLS and reverse-proxies every request to Home Assistant across the WireGuard
tunnel. Home Assistant runs the WireGuard Client add-on and dials into the Zerops private network, so no inbound ports
need to be opened at home. The proxy config sets the upgrade/forwarding headers required for the Home Assistant web UI and
WebSocket connections.

### Environment variables

| Variable     | Required | Description                                                                                                              |
|--------------|----------|--------------------------------------------------------------------------------------------------------------------------|
| `HA_ADDRESS` | yes      | Home Assistant's address on the VPN, including its port — e.g. `10.2.164.3:8123`. Nginx proxies to `http://$HA_ADDRESS`. |

### Troubleshooting

- **502 / can't reach Home Assistant** — check the WireGuard add-on log for a successful handshake, and confirm
  `HA_ADDRESS` matches the `Address` from your VPN config (plus the `:8123` port).
- **Bad Request after deploy** — Home Assistant is rejecting the proxy. Add the VPN ranges to `trusted_proxies` in
  `configuration.yaml` and **fully restart** Home Assistant (a reload won't apply it). The Core log shows
  `Received X-Forwarded-For header from an untrusted proxy …`.
- **Tunnel drops after a while** — set `persistent_keep_alive` to `5` on the peer and enable the add-on's Watchdog.
<!-- #ZEROPS_EXTRACT_END:knowledge-base# -->
