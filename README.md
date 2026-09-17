# brunernetes install test — artifacts

By-the-docs install test of [brunernetes](https://github.com/brunka09/brunernetes) on a fresh
Ubuntu 22.04 host (cgroup v2, internet access). See `ISSUE_DRAFT.md` for findings.

- `bruner-install.gif` — strict docs run: build → dry-run → install → verify → demo
- `bruner-install2.gif` — re-run with helm installed (chart applies CRDs; controller ImagePullBackOff)
- `*.cast` — original asciinema recordings

## Re-verification (2026-09-17)
Captured on a fresh Ubuntu 22.04 VM, actual command output (not from the asciinema casts):
- `describe-pod.txt` — `kubectl -n brunernetes-system describe pod` + events: controller
  `ghcr.io/brunka09/brunernetes:0.1.0` in ImagePullBackOff.
- `crictl-pull.txt` — `sudo crictl pull` of the controller image. First fails on DNS
  (`lookup ghcr.io ... server misbehaving`); after pointing host DNS away from 10.96.0.10 the
  real cause surfaces: **403 Forbidden** (private/unpublished image). Direct token endpoint = 403.
- `dns-check.txt` — `kube-dns` (CoreDNS) ClusterIP = **10.96.0.10**, equal to the host's
  upstream DNS; `getent github.com` OK before install, FAIL after. Confirms the service-CIDR /
  host-DNS collision.
