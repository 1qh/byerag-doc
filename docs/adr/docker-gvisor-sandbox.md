# docker-gvisor-sandbox

Per-owner sandbox runs in a Docker container with gVisor (`runsc`) runtime. One container per owner, paused between user messages, killed on chat delete / liveness failure / explicit revocation.

## Beats

- **E2B (Firecracker microVM as a service)**: SaaS — violates self-host invariant + adds outbound dest. Drop.
- **Self-host Firecracker**: ~125ms boot, kernel-level isolation, but ops complexity (VM image building, kernel pinning, jailer config) is large.
- **Plain Docker w/ `runc`**: faster boot than gVisor but shares the host kernel; kernel CVE escapes the container. Acceptable threat for trusted internal team, but cheap gVisor swap closes the hole.
- **Kata Containers**: each container is a lightweight VM; isolation strong but cold-start higher than gVisor.

## Real cost

- gVisor's syscall interception adds ~10-20% CPU overhead vs `runc` (acceptable: workload is grep + read + LLM round trip, not CPU-bound).
- gVisor doesn't support every syscall; some niche tools may behave differently. Mitigated by curated sandbox image with known-good tools.
- Per-owner container survives across user messages (E2B-like resume semantics) — needs a persistent volume bind for the agent's scratch dir.

## Gotcha for Claude

- Install `runsc` (apt package or upstream tarball); register in `/etc/docker/daemon.json` under `runtimes: { runsc: ... }`; restart Docker daemon. Per-container `--runtime=runsc` opts into gVisor.
- `--cap-drop ALL` + `--security-opt no-new-privileges` + `--read-only` rootfs + `tmpfs` scratch.
- `--memory` and `--cpus` quotas per container.
- `--network` attached to a restricted bridge (not default `bridge`). The bridge's iptables egress allows Convex internal endpoint only.
- One container per owner = `sandboxes.owner` is the unique key; `sandboxes.sandboxId` is the Docker container id.
- Lifecycle: connect-or-create on first chat message of a session; pause container between messages (Docker `pause`); unpause on next message; kill on chat delete or 14-day stale.
- The substrate reference's `sandboxClient.ts` shape (`createSandbox`, `connectSandbox`, `commands.run`) is preserved; implementation swaps `e2b` SDK calls for `dockerode` calls behind the same interface.
