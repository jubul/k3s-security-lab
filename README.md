# k3s Security Lab

***English** · [Español](README.es.md)*

A Kubernetes security lab built from scratch on KVM virtual machines: a multi-node **k3s** cluster with **admission control** (Kyverno), **runtime detection** (Falco), and **alert centralization** in Elasticsearch/Kibana.

The point of this repository isn't to show that the stack comes up. It's to document **how every decision and every failure was reasoned through**. The full diagnosis of each problem encountered lives in [`docs/journal.md`](docs/journal.md), with root cause and evidence — including my own mistakes.

---

## Current status

| # | Module | Status |
|---|--------|--------|
| 0 | KVM/libvirt prerequisites on the host | ✅ Done |
| 1 | Two Debian 13 VMs provisioned with cloud-init | ✅ Done |
| 2 | k3s cluster: control plane + worker node | ✅ Done |
| 3 | Fundamentals: Namespace, Deployment, Service, labels | ✅ Done |
| 4 | Kyverno: admission control and policies | ✅ Done |
| 5 | Falco: runtime detection with eBPF | ✅ Done |
| 6 | Falcosidekick → Elasticsearch → Kibana | ✅ Done |
| 7 | NetworkPolicies and Pod Security | 🔨 2 of 4 |
| 8 | Final documentation and diagrams | ⬜ Pending |

**Verified:** the policy negative-test suite passes 5 of 5 ([`tests/`](tests/)); the custom Falco rule detects Service Account token theft across two images with different mount paths, with noise tuned down from 804 alerts to 1; the pipeline indexes into Elasticsearch end to end; and the Pod Security policies reject privileged and root pods **while still allowing the Falco DaemonSet to recreate its own pods** — which is the test that actually validates the exemption design.

---

## Architecture

```
        HOST (CachyOS · 16 cores · 15 GB RAM)
        kubectl · helm · browser → Kibana
        ufw enabled, with explicit rules for the lab
                        │
              virbr0 — libvirt NAT — 192.168.122.0/24
                        │
        ┌───────────────┴────────────────┐
        │                                │
┌───────────────────┐          ┌───────────────────┐
│    k3s-server     │          │     k3s-agent     │
│  192.168.122.6    │◄────────►│  192.168.122.128  │
│   3 GB · 2 vCPU   │  flannel │   6 GB · 4 vCPU   │
│                   │  VXLAN   │                   │
│  control plane    │          │  Elasticsearch    │
│  Kyverno          │          │  Kibana           │
│  Falco (DaemonSet)│          │  Falco (DaemonSet)│
└───────────────────┘          └───────────────────┘
   Debian 13.6 · kernel 6.12 · UEFI boot · BTF present
```

Falco runs as a **DaemonSet**: one instance per node, each watching the syscalls of its own kernel.

---

## Stack, and why

| Component | Choice | Rationale |
|---|---|---|
| Cluster | **k3s** | A single binary containing the whole control plane. CNCF-certified and used in production at the edge — not a toy. |
| Infrastructure | **libvirt/KVM** | Real kernels per node. Falco observes syscalls, and in containers sharing the host kernel the rules don't behave the way they would in production. |
| Guest | **Debian 13 genericcloud** | Minimal image, virtio only. Kernel 6.12 with `CONFIG_DEBUG_INFO_BTF=y`, which is what makes eBPF work without compiling modules. |
| Provisioning | **cloud-init (NoCloud)** | Reproducible nodes from version-controlled YAML, no manual installation. |
| Admission control | **Kyverno** | Policies are Kubernetes objects written in YAML: they live in git and can be explained without translating a separate language. OPA/Gatekeeper is more powerful and considerably steeper. |
| Runtime security | **Falco** | The CNCF standard for syscall-based detection. |
| Alerting | **Falcosidekick + ELK** | Falco detects; the pipeline turns detection into something queryable and historical. |

### Prevention and detection are different layers

The project deliberately combines two complementary approaches:

- **Kyverno acts at the door.** A pod rejected at admission never executed a single line of code. That's prevention, and it only covers what can be decided by looking at the manifest.
- **Falco acts inside.** A legitimate container that three weeks later opens an unexpected shell or reads `/etc/shadow` passed every admission check. That is only visible by observing behaviour.

Neither layer replaces the other, and that's the thesis of the lab.

---

## Repository layout

```
├── docs/
│   ├── journal.md          Incidents with diagnosis and root cause (English)
│   ├── bitacora.md         The same incidents in Spanish
│   └── estado-sesion.md    Progress state and resume point
├── infra/
│   └── cloud-init/         user-data and meta-data for each node (NoCloud)
├── manifests/              Sample app and the attack-simulation pod
├── policies/               Kyverno ClusterPolicies
├── falco/
│   └── values.yaml         Full Falco configuration: driver, output and custom rules
├── elk/                    Elasticsearch (StatefulSet) and Kibana (Deployment)
├── netpol/                 Isolation NetworkPolicies
└── tests/                  Policy negative tests, with expected results
```

---

## Reproducing it

Requires a Linux host with hardware virtualization (`vmx` or `svm`), ~10 GB of free RAM and 45 GB of disk.

### 1. Host: virtualization

```bash
sudo pacman -S --needed qemu-desktop libvirt virt-install dnsmasq openbsd-netcat libisoburn
sudo systemctl enable --now libvirtd.socket
sudo usermod -aG libvirt $USER      # requires logging out and back in
sudo virsh net-start default && sudo virsh net-autostart default
```

On fish, so you don't have to repeat `-c qemu:///system` on every command:

```fish
set -Ux LIBVIRT_DEFAULT_URI qemu:///system
```

### 2. Host firewall

If you run `ufw`, without these rules the guests get neither an IP address nor outbound connectivity ([journal #6](docs/journal.md)):

```bash
sudo ufw allow in on virbr0 to any port 67 proto udp comment 'libvirt DHCP'
sudo ufw allow in on virbr0 to any port 53 comment 'libvirt DNS'
sudo ufw route allow in on virbr0 out on <uplink-interface> comment 'k3s lab egress'
sudo ufw reload
```

### 3. Base image and disks

```bash
cd /tmp
curl -LO https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2
curl -LO https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
sha512sum --ignore-missing -c SHA512SUMS      # verify before you execute it
sudo mv debian-13-genericcloud-amd64.qcow2 /var/lib/libvirt/images/

cd /var/lib/libvirt/images
sudo qemu-img create -f qcow2 -F qcow2 -b debian-13-genericcloud-amd64.qcow2 k3s-server.qcow2 20G
sudo qemu-img create -f qcow2 -F qcow2 -b debian-13-genericcloud-amd64.qcow2 k3s-agent.qcow2 20G
```

The disks are copy-on-write overlays: they start out at ~200 KB. **The base image must not be modified or deleted**, or both nodes are corrupted.

### 4. cloud-init seeds

```bash
cd infra/cloud-init
mkdir -p seed/server seed/agent
cp user-data-server.yaml seed/server/user-data
cp user-data-agent.yaml  seed/agent/user-data
printf 'instance-id: k3s-server-01\nlocal-hostname: k3s-server\n' > seed/server/meta-data
printf 'instance-id: k3s-agent-01\nlocal-hostname: k3s-agent\n'  > seed/agent/meta-data

xorrisofs -output /tmp/seed-server.iso -volid cidata -joliet -rock seed/server
xorrisofs -output /tmp/seed-agent.iso  -volid cidata -joliet -rock seed/agent
sudo mv /tmp/seed-*.iso /var/lib/libvirt/images/
```

Replace the public key in the `user-data-*.yaml` files with your own. The NoCloud contract is strict: volume label `cidata`, and files named exactly `user-data` and `meta-data`, with no extension.

### 5. The VMs

```bash
sudo virt-install \
  --name k3s-server --memory 3072 --vcpus 2 --cpu host-passthrough \
  --boot uefi \
  --disk /var/lib/libvirt/images/k3s-server.qcow2,format=qcow2,bus=virtio \
  --disk /var/lib/libvirt/images/seed-server.iso,device=cdrom \
  --osinfo debian13 --network network=default,model=virtio \
  --graphics none --console pty,target_type=serial --import
```

Same for the agent, with 6144 MB, 4 vCPUs and its own files.

⚠️ **`--boot uefi` is mandatory.** With BIOS boot (SeaBIOS) the guests never come up: the firmware enters a retry loop and the VM sits there "running" without ever reaching the kernel ([journal #5](docs/journal.md)).

⚠️ **Do not use `virt-install --cloud-init`.** Combined with `--noautoconsole` it aborts the first boot and discards the seed ISO ([journal #4](docs/journal.md)). That's why the seed is built by hand in step 4.

### 6. k3s

On the server:

```bash
curl -sfL https://get.k3s.io | sudo sh -s - server \
  --node-ip 192.168.122.6 --tls-san 192.168.122.6
sudo cat /var/lib/rancher/k3s/server/node-token
```

On the agent:

```bash
curl -sfL https://get.k3s.io | sudo \
  K3S_URL=https://192.168.122.6:6443 K3S_TOKEN='<token>' \
  sh -s - agent --node-ip 192.168.122.128
```

On the host:

```bash
sudo pacman -S --needed kubectl helm
mkdir -p ~/.kube
ssh jubul@192.168.122.6 'sudo cat /etc/rancher/k3s/k3s.yaml' > ~/.kube/config
chmod 600 ~/.kube/config
sed -i 's|127.0.0.1|192.168.122.6|' ~/.kube/config
kubectl get nodes -o wide
```

### 7. Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/ && helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace \
  --set admissionController.replicas=1 --set backgroundController.replicas=1 \
  --set cleanupController.replicas=1 --set reportsController.replicas=1

kubectl apply -f policies/
kubectl get clusterpolicy
```

Run the negative-test suite — the first four must be rejected and the fifth accepted:

```bash
kubectl apply -f tests/ -n demo --dry-run=server
```

### 8. Falco

All configuration lives in [`falco/values.yaml`](falco/values.yaml): the driver, the output format and the custom rules.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts && helm repo update
helm install falco falcosecurity/falco -n falco --create-namespace -f falco/values.yaml
kubectl rollout status daemonset/falco -n falco
```

To update after editing the values file, **without `--reuse-values`** (the file already holds the complete configuration):

```bash
helm upgrade falco falcosecurity/falco -n falco -f falco/values.yaml
helm list -n falco      # confirm the REVISION went up
```

Confirm the right driver loaded — you should see `Opening 'syscall' source with modern BPF probe`:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=40 | grep -i "BPF probe"
```

`driver.kind: modern_ebpf` uses eBPF with **CO-RE** (*Compile Once, Run Everywhere*): the program ships precompiled and adapts to the local kernel by reading the BTF at `/sys/kernel/btf/vmlinux`. Without BTF you'd have to compile a module against each node's headers, and installing compiler toolchains on production nodes is exactly what you don't want.

**Test it with a simulated intrusion:**

```bash
kubectl apply -f manifests/04-intruso.yaml
kubectl exec -it intruso -n demo -- sh
# inside:  cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

And read the alerts **without filtering by severity** — filtering on `warning` hides the `NOTICE` rules, which are most of the reconnaissance activity:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=200 \
  | grep '"rule":' \
  | jq -r '.priority + " | " + .rule + " | " + (.output_fields["k8s.pod.name"] // "-")'
```

### 9. Elasticsearch and Kibana

```bash
kubectl apply -f elk/
kubectl rollout status statefulset/elasticsearch -n elk
kubectl rollout status deployment/kibana -n elk
```

Falcosidekick is deployed as a subchart from `falco/values.yaml` (`falcosidekick.enabled: true`), which also wires up Falco's `http_output` automatically. Verify the pipeline is indexing:

```bash
kubectl exec -n elk elasticsearch-0 -- curl -s 'localhost:9200/_cat/indices/falco-*?v'
```

Kibana is exposed at **http://192.168.122.128:30601** via NodePort. The data view must use the **`falco-*`** wildcard pattern: indices are daily (`falco-2026.08.20`), so a pattern without the wildcard works today and breaks tomorrow.

### 10. NetworkPolicies and Pod Security

```bash
kubectl apply -f netpol/
kubectl apply -f policies/
```

The Pod Security policies are in `Enforce`. The test that validates the exemption design is **not** that they reject a privileged pod — it's that Falco can still recreate its own:

```bash
kubectl delete pod -n falco -l app.kubernetes.io/name=falco
kubectl get daemonset falco -n falco    # must return to 2/2
```

If the exemption is written wrong, nothing fails visibly: the DaemonSet simply never reaches its desired replicas, and the cluster is left without detection.

---

## Security decisions

Deliberate choices, with their rationale:

- **The kubeconfig is copied over SSH with `sudo cat`, not via `--write-kubeconfig-mode 644`.** That file is the cluster's admin credential; loosening its permissions to save one `sudo` would contradict the point of the project. It stays at mode `600`.
- **`ufw` stays enabled.** Disabling it would have fixed the DHCP problem in one command. Instead, minimal rules were added, commented and documented.
- **The base image is SHA512-verified before being executed.** Verifying the artifact you're about to run is the starting point, not a formality.
- **The image's default `debian` user is not created.** Declaring `users:` in cloud-init replaces the default list: fewer accounts, less surface.
- **Images carry explicit, pinned tags — never `latest`.** Enforced by policy, not by convention.
- **The Falco policy exemption matches on namespace AND label.** Exempting the whole namespace would also have covered `falcosidekick`, which needs no privileges at all.
- **Owned workloads were fixed rather than exempted.** `kube-system` is exempted because k3s manages it; `demo` and `elk` were corrected instead. If you exempt everything that fails, the policy protects nothing.
- **The k3s `node-token` is a credential.** Whoever holds it can join nodes to the cluster — that is, run workloads on it. It must never reach the repository.

### What this lab is **not**

It's a learning environment, not a production reference. Conscious limitations:

- Single-node control plane backed by SQLite, no high availability.
- Passphrase-less SSH keys and password-less `sudo` on the nodes.
- All Kyverno controllers at one replica, due to RAM constraints.
- **Elasticsearch has no authentication and no TLS.** Today it's protected by network isolation alone: anyone who compromises a pod in the `falco` namespace or the Kibana pod retains full read and delete access. Enabling `xpack.security` is pending.
- **The SIEM lives inside the cluster it monitors.** Convenient for a lab, an anti-pattern in production: collection should happen outside the trust boundary of the system being watched.
- Only network ingress is controlled; egress is wide open.
- Nodes get their addresses over DHCP, so IPs can change if the VMs are recreated.

---

## The interesting parts

If you only read one thing, read the [journal](docs/journal.md). Twelve incidents with full diagnosis. A few examples:

**A guest executing code without ever booting.** The VMs showed as running, burned CPU and read 3.37 GB from disk, yet wrote not a single byte and transmitted not a single packet. With the disk file write-locked by the running VM, the diagnosis came from the hypervisor's counters and the QEMU monitor: the vCPU registers showed `CS=f000` in 16-bit real mode — BIOS code. Ten minutes after boot, control had never been handed to the kernel.

**An admission policy that passed its own test and could be bypassed two ways.** It correctly blocked `nginx:latest`. But it only validated `spec.containers`, leaving `initContainers` and `ephemeralContainers` wide open; and because it matched the literal string `:latest`, an untagged image — which the runtime resolves to `latest` anyway — sailed through. On the rewrite, a capitalization error (`initcontainers` instead of `initContainers`) made the validation **skip silently**, inside a policy that reported `Ready`.

**A detection rule that was correct and, as written, useless.** The custom Service Account token-theft rule worked: it caught the attack. It also produced **804 alerts with 2 true positives** — 0.25% signal, because every Kubernetes component reads its own token to authenticate. Tuning it required fixing visibility first (Falco only emits the fields referenced in the rule's `output` template, so the process fields came back empty), and surfaced two findings about the available fields: `proc.name` is truncated at 15 characters because it comes from the kernel's `comm`, and `proc.exepath` collapses multi-call binaries — on Alpine, `cat` reports `/bin/busybox`. Neither field is right for every case.

**A SIEM the attacker could delete.** Auditing the cluster before writing policies revealed that any pod could reach Elasticsearch unauthenticated. It was verified by creating and deleting a throwaway index **from the compromised pod**, hand-crafting the HTTP request with `nc` because BusyBox's `wget` doesn't support `--method`. In other words: an attacker could erase the very indices that recorded their own intrusion, using tooling that already shipped in a minimal image. Mitigated with NetworkPolicies; the design lesson is that the SIEM shouldn't live inside the trust boundary of the system it watches.

**Hardening also turns off the telemetry.** Moving the simulation pod from root to uid 1000 changed the same attack from two alerts to one: `cat /etc/shadow` now returns *permission denied*, and Falco's `open_read` macro requires an actually-opened descriptor (`fd.num >= 0`), so **a failed open matches no rule at all**. The attack failed, but the attempt went unrecorded too. That's an interaction between the prevention and detection layers that tutorials don't cover.

**A pattern that recurred seven times.** The measuring instrument producing the conclusion: a `head -5` that led to the claim there was no DHCP server when there was one; a `grep -i warning` that hid a `NOTICE`-priority detection; an `ignore_above: 256` that makes a Kibana aggregation return empty with no error; a `jq` expression touching an absent field that left a violation inventory blank; a `tail -4` that concealed one of the two rules that had in fact rejected a pod; a test case that didn't cover the intersection of two conditions; and a capitalization error that made a validation skip without failing. **None of the seven announced that it was truncating** — and several happened *after* the pattern had been identified, which is precisely what makes it worth writing down.

---

## Roadmap

- [x] Kyverno policies with negative tests versioned under `tests/`
- [x] Falco with modern eBPF, a custom rule, and validation against real events
- [x] Falcosidekick → Elasticsearch → Kibana pipeline
- [x] NetworkPolicies isolating the evidence from cluster workloads
- [x] Pod Security: forbid `privileged`, require `runAsNonRoot`, with surgical exemptions
- [ ] `automountServiceAccountToken: false` where it isn't needed: the prevention that complements the token-theft rule
- [ ] Demonstrate `Drop and execute new binary in container` — the detection that covers an attacker in a distroless container, who has to bring their own binary
- [ ] Finish Pod Security: `readOnlyRootFilesystem`, forbid `hostPath`, `hostNetwork` and `hostPID`, require `seccompProfile: RuntimeDefault`
- [ ] Egress NetworkPolicies (only ingress is controlled today)
- [ ] Enable `xpack.security` with TLS on Elasticsearch: today the isolation is network-only
- [ ] Migrate `tests/` to `kyverno test` so it can run in CI: with `kubectl apply --dry-run` the exit code is inverted for negative tests
- [ ] Dedicated RBAC for the `falco` namespace, which is now the privileged path to the evidence
- [ ] Architecture diagram and threat model
- [ ] `infra/lab-up.sh` and `lab-down.sh` scripts to bring the lab up and down
- [ ] *(candidate)* GitOps with Argo CD or Flux, which eliminates by design the drift between what the repo declares and what the cluster runs
