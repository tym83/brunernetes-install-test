Title: install --preset garage exits 0 but leaves the platform non-functional (helm "optional" but required; controller image 403; kubeconfig not set for invoking user)

## Environment
- Fresh Ubuntu 22.04 (x86_64), cgroup v2, swap off, 4 vCPU / 16 GiB / 40 GiB, root via sudo, internet access.
- brunernetes @ main (commit b887e7b), Go 1.24.0.
- Followed docs/installation.md verbatim.

## Summary
The single-node kubeadm bootstrap works (node becomes Ready, control-plane, v1.32.3), but a
by-the-docs install does not yield a working platform. Three issues, in order of impact:

### 1. Controller image is not pullable — `ghcr.io/brunka09/brunernetes:0.1.0` → 403 Forbidden
With helm installed, step 8 applies the chart (CRDs are created), but the controller never starts:
```
Failed to pull image "ghcr.io/brunka09/brunernetes:0.1.0": ... failed to fetch anonymous token:
unexpected status from GET request to https://ghcr.io/token?...: 403 Forbidden
```
Result: pod `brunernetes-...` ImagePullBackOff, `helm ls` shows the release **failed**, and
`curl http://127.0.0.1:30080/healthz` → connection refused. The image appears private/unpublished.
Please publish it publicly, or document building & pushing your own and overriding the chart image.
(Also: `helm upgrade --install --wait --timeout 5m` makes `install` hang ~5 min on the bad pull
before failing — a faster/clearer failure would help.)

### 2. helm is documented as "optional" but is effectively required
README lists helm as optional ("missing triggers a warning, not failure"). Without helm, step 8 is
skipped, so no CRDs are created, and step 9 fails:
```
WARN Factory resources could not be applied: ... no matches for kind "Togliatti" in version
"bruner.dev/v1alpha1" / no matches for kind "AvtoVAZ" ...
```
Despite this, `brunerctl install` returns **exit code 0** — a misleading success. Suggest either
making helm a hard preflight requirement, or returning non-zero when the chart/CRDs weren't applied.

### 3. kubeconfig is not available to the invoking (sudo) user after install
Docs say install copies admin kubeconfig to `$HOME/.kube/config`. Run via `sudo`, `$HOME` is root's,
so a normal user's kubectl is unconfigured; every verify command in the docs fails:
```
kubectl get nodes           -> The connection to the server localhost:8080 was refused
brunerctl status/diagnose   -> kubernetes connection settings could not be resolved ... 
                               "The factory office has no key to the floor"
```
Suggest copying to `$SUDO_USER`'s home (chown’d), or documenting `export KUBECONFIG=/etc/kubernetes/admin.conf`.

### 4. (environment note) default service CIDR collides with host DNS inside another cluster's network
When installed on a VM whose upstream DNS is 10.96.0.10 (a Kubernetes tenant), kubeadm's default
CoreDNS ClusterIP is also 10.96.0.10; kube-proxy then hijacks it and host DNS breaks after install.
Not a bug on a normal host, but a `--service-cidr` / DNS note in the docs would save time.

## What worked
preflight/dry-run (exit 0), kernel modules, sysctls, pinned packages + containerd, kubeadm init,
taint removal, CNI — single node Ready (v1.32.3); CRDs apply once helm is present.

## Repro (by the docs)
```
make build
sudo ./bin/brunerctl install --preset garage --dry-run
sudo ./bin/brunerctl install --preset garage
kubectl get nodes            # fails: kubeconfig not set for user
kubectl get crd | grep bruner.dev   # empty without helm
# with helm installed, re-run install: CRDs appear, controller -> ImagePullBackOff (403)
curl -fsS http://127.0.0.1:30080/healthz   # connection refused
```
