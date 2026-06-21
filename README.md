# Zerops → Home Assistant proxy

Expose a self-hosted [Home Assistant](https://www.home-assistant.io/) instance to the internet - with a real domain and
HTTPS - without opening any ports on your home router.

> ⚠️ **This recipe does not support one-click import.** Follow the steps below in order.

## How it works

Home Assistant usually lives on your home network, unreachable from the outside. This recipe runs a small **Nginx reverse
proxy** on Zerops that connects back to Home Assistant over a **WireGuard VPN**:

```
Browser ──HTTPS──> Zerops (Nginx proxy) ──WireGuard tunnel──> Home Assistant (at home)
```

Home Assistant joins the Zerops private network as a WireGuard client, and the Zerops proxy forwards all traffic to it.
You get a public, TLS-terminated address; your home network stays closed.

## Prerequisites

- A Home Assistant instance you can install add-ons on (Home Assistant OS / Supervised).
- A Zerops account and an empty project.
- [`zcli`](https://docs.zerops.io/references/cli) installed on your machine.

---

## Step 1 - Generate the WireGuard config

1. Create (or import) an **empty** project in the Zerops UI - no services yet.
2. Install `zcli`, log in with `zcli login <token>`, then generate a VPN config:

   ```sh
   zcli vpn config --output -
   ```

   Select the project you just created. You'll get something like this:

   ```ini
   [Interface]
   PrivateKey = kGWjZjklpuzoJKLPUZOjklpuzo/TUDcVo=
   MTU = 1420

   Address = 10.2.164.3/32, fda0:5ef:a7:c0df:10:2:164:3/128
   DNS = 10.2.160.1, zerops

   [Peer]
   PublicKey = wOEl3OBqASDFasdfASDFasdfASDFpC6wShLSTh/Aw=
   AllowedIPs = 10.2.160.0/22, fda0:5ef:a7:c0de::/64, 10.2.164.0/22, fda0:5ef:a7:c0df::/64
   Endpoint = proxy.app-prg1.zerops.io:14504
   ```

3. **Keep this config handy** - you'll copy its values into Home Assistant in the next step, and you'll need the
   `Address` again in Step 3.

---

## Step 2 - Set up the WireGuard client in Home Assistant

### Install the add-on

1. Open Home Assistant.
2. Go to **Settings → Add-ons → Add-on Store**.
3. Click the **⋮** (three dots, top right) → **Repositories**.
4. Add this URL and click **Add**:

   ```
   https://github.com/bigmoby/hassio-repository-addon
   ```

5. Back in the store, search for **WireGuard Client**, then install it.

### Configure it

Open the add-on's **Configuration** tab and copy the values from your `zcli` config. The fields map directly:

| WireGuard config         | Home Assistant field        |
|--------------------------|-----------------------------|
| `[Interface] PrivateKey` | `interface` → `private_key` |
| `Address`                | `interface` → `address`     |
| `DNS`                    | `interface` → `dns`         |
| `MTU`                    | `interface` → `mtu`         |
| `[Peer] PublicKey`       | `peers` → `public_key`      |
| `Endpoint`               | `peers` → `endpoint`        |
| `AllowedIPs`             | `peers` → `allowed_ips`     |

Notes:

- For `address` and `allowed_ips`, add each comma-separated value as a **separate entry**.
- For `dns`, keep it as a **single entry** exactly as generated (e.g. `10.2.160.1, zerops`) - the first part is the DNS
  server and `zerops` is the search domain.
- Set `peers` → `persistent_keep_alive` to **5** to keep the tunnel alive behind NAT.
- Leave `pre_shared_key` empty.

### Start it

Save the config, then on the **Info** tab:

1. **Start** the add-on.
2. Enable **Start on boot** and **Watchdog** so the tunnel comes back automatically.

Check the **Log** tab to confirm the handshake succeeds.

### Trust the proxy

Home Assistant rejects forwarded requests unless it knows the proxy is trusted. Edit your `configuration.yaml` (the
**Studio Code Server** add-on makes this easy) and add:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - ::1
    - 127.0.0.1
    - 10.2.160.0/22
    - 10.2.164.0/22
```

The last two ranges come from `Peer.AllowedIPs` in your VPN config - adjust them if yours differ.

> ⚠️ You **must fully restart** Home Assistant for this to take effect - a config *Reload* is not enough. If you skip the
> restart, the proxy returns **Bad Request** and the Home Assistant Core log shows:
> `Received X-Forwarded-For header from an untrusted proxy 10.2.160.x`.

---

## Step 3 - Deploy the proxy on Zerops

1. In the Zerops UI, import the **`services`** section of this recipe's `import.yaml` into your project (paste just the
   `services:` part - leave out the `project:` part, since the project already exists). The build itself is driven by this
   repo's `zerops.yaml`.
2. Set the **`HA_ADDRESS`** environment variable to the IPv4 part of your `Address` field plus Home Assistant's port -
   **without** the `/32` suffix or the IPv6 part.

   Example: for `Address = 10.2.164.3/32, fda0:...`, set `HA_ADDRESS=10.2.164.3:8123` (`8123` is Home Assistant's default port).

3. The proxy should now be reachable through the Zerops **preview domain**. Open it and confirm Home Assistant loads.
4. Once verified, add your **custom domain** and disable the preview domain.

🎉 Done - Home Assistant is now available on your own domain over HTTPS.

---

## Troubleshooting

- **502 / can't reach Home Assistant:** check the WireGuard add-on **Log** for a successful handshake, and double-check
  `HA_ADDRESS` matches the `Address` from your config.
- **Tunnel drops after a while:** make sure `persistent_keep_alive` is set (e.g. `5`) and **Watchdog** is enabled.
