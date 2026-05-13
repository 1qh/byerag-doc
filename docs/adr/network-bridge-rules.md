# network-bridge-rules

The sandbox container attaches to a Docker bridge named `sandbox-egress`. The bridge is `internal: false` (allows DNS resolution + outbound from the bridge) but iptables FORWARD rules restrict outbound destinations to the Convex container's IP + port only.

## Bridge declaration

`compose.yml` declares two networks:

- `internal`: standard bridge for service-to-service (Postgres, Convex, Ollama, ClamAV, scan).
- `sandbox-egress`: bridge for the sandbox; only Convex is attached to both.

## iptables rules (host)

```
iptables -I DOCKER-USER -i br-sandbox-egress -o br-internal -d <convex-internal-ip> -p tcp --dport 3210 -j ACCEPT
iptables -I DOCKER-USER -i br-sandbox-egress -o br-internal -d <convex-internal-ip> -p tcp --dport 3211 -j ACCEPT
iptables -I DOCKER-USER -i br-sandbox-egress -j DROP
```

Effect: traffic from sandbox bridge to anywhere other than the Convex container's IPs on the Convex ports is dropped.

DNS resolution allowed only for the Convex container's hostname (`convex-backend`). The sandbox image's `/etc/hosts` has a baked `convex-backend <ip>` mapping; `/etc/resolv.conf` points at `127.0.0.1` with no real resolver listening — DNS queries fail by design, only the `/etc/hosts` entry resolves.

## Host-level egress

Outbound from the host (where Convex runs) is firewalled to `api.kimi.com:443` only. Separate `nftables` ruleset on the host:

```
table inet filter {
  chain output {
    type filter hook output priority 0; policy drop;
    ct state established,related accept
    oifname "lo" accept
    udp dport 53 ip daddr 127.0.0.53 accept
    tcp dport 443 ip daddr @kimi_ips accept
  }
}
```

`kimi_ips` is a named set refreshed by a cron job that resolves `api.kimi.com` and updates the set.

## Beats

- **Open sandbox network + app-layer SSRF defense alone**: belt without suspenders; one URL parse confusion bypasses everything.
- **Sandbox `--network none`**: sandbox can't reach Convex; CLI tool calls fail.
- **VPN tunnel for sandbox**: heavier; more attack surface.

## Real cost

- iptables rules need maintenance when Convex's bridge IP changes (re-pin on compose restart).
- Kimi IP rotation requires the cron-refreshed set; stale set → stale denials.

## Gotcha for Claude

- Docker's `DOCKER-USER` chain is the supported integration point; rules in `INPUT` / `FORWARD` get clobbered by Docker on restart.
- The `iptables -I DOCKER-USER ... -j DROP` rule must be LAST in the chain (deny-after-allow). Use `-A` to append, not `-I`. The two ACCEPT rules go first via `-I`; the DROP via `-A`.
- DNS poisoning defense via baked `/etc/hosts`: if the sandbox could resolve `api.kimi.com` itself, even denied egress would expose a successful lookup attempt to a man-in-the-middle. Bake the host file, kill external resolvers.
- Verify with: `docker run --rm --network sandbox-egress curlimages/curl curl -s https://api.kimi.com/` (must fail).
