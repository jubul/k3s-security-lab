# Incident journal

***English** · [Español](bitacora.md)*

A record of the real problems hit while building this lab, with the diagnosis and root cause of each one. The value of this file isn't the list of commands that worked — it's the reasoning that found the ones that didn't.

Format of each entry: symptom → diagnosis → root cause → fix → takeaway.

---

## #1 — Can't install QEMU: 404s on every mirror, then broken dependencies

**Date:** 2026-08-17 · **Module:** 0 (prerequisites) · **Status:** resolved

### Symptom

Installing the virtualization stack, `pacman` failed in two distinct stages:

1. First, 404 errors downloading `qemu-system-x86`, `qemu-common`, `edk2-ovmf`, `rdma-core` and others — against **every** mirror in the list (40+).
2. After refreshing the database with `pacman -Syy`, the 404s vanished and a different error appeared:

```
:: installing nettle (4.0-1) breaks dependency 'nettle=3.10.2' required by lib32-nettle
:: installing nettle (4.0-1) breaks dependency 'libnettle.so=8-64' required by wget
```

### Diagnosis

Both errors are symptoms of the same underlying problem, in two phases:

- The **404s** came from a stale local database: pacman was asking for package versions that no longer existed on the mirrors. That *every* mirror failed rather than one was the tell — the problem was client-side, not server-side.
- The **broken dependencies** only surfaced once the database was current. QEMU required `nettle 4.0`, but the installed `wget` and `lib32-nettle` were linked against `nettle 3.10.2`. Installing QEMU alone would have left those packages pointing at a `.so` that no longer existed.

`pacman -Qu` revealed the real scale: **775 pending packages**, roughly four months behind.

### Root cause

Arch Linux is a rolling release and **does not support partial upgrades**. There is no way to install a new package without upgrading the whole system, because repository packages are always built against the current versions of their dependencies.

### Fix

Before touching anything, the Arch news from April to August 2026 were reviewed for mandatory manual interventions. Four candidates, none applicable: `virtualbox-ext-vnc` and `varnish` weren't installed, and `iptables` was already on the nft backend.

Procedure:

```bash
# 1. Safety net: snapshot before touching the system
sudo snapper -c root create --description "pre-upgrade lab-k3s 775 pkgs" --cleanup-algorithm number

# 2. Keyring first, and only the keyring
sudo pacman -Sy archlinux-keyring cachyos-keyring

# 3. Full upgrade
sudo pacman -Syu

# 4. Mandatory reboot
```

The keyring goes first and on its own for a concrete reason: the installed one was from 2026-04-20, and with rotated signing keys the verification of the other 774 packages fails mid-transaction, leaving the system half-upgraded. It is the **only legitimate exception** to the no-partial-upgrades rule.

The reboot isn't optional: the kernel (7.0.1 → 7.1.8), systemd (260 → 261), glibc (2.43 → 2.44) and the NVIDIA driver (595.58.03 → 610.57.04) all changed. After the upgrade, the running kernel's modules no longer exist on disk.

Result: no incidents. Kernel 7.1.8 booted clean, `kvm_intel` loaded, NVIDIA 610 responding, Plasma Wayland fine.

### Takeaway

`pacman -Sy <package>` is an anti-pattern: it leaves the database ahead of the installed packages and is the shortest path to a broken system. **Always `-Syu`.** And on a system months behind, the correct order is snapshot → keyring → full upgrade → reboot.

### Related security note

The Arch news of 2026-06-12 reports an incident of malicious package adoptions in the AUR, without naming specific packages. This machine's 10 AUR packages date from late April, **predating the incident**, so there was no exposure. Exposure would appear on updating them: `paru -Syu` remains pending and requires reviewing every PKGBUILD first.

---

## #2 — `virsh net-list` returns an empty table while the network works

**Date:** 2026-08-17 · **Module:** 0 (prerequisites) · **Status:** resolved

### Symptom

Two consecutive commands with contradictory results:

```
virsh -c qemu:///system version   → responds fine (libvirt 12.6.0, QEMU 11.1.0)
virsh net-list --all              → completely empty table
ip -br addr show virbr0           → virbr0  DOWN  192.168.122.1/24
```

A non-existent network can't have a bridge with an assigned IP.

### Diagnosis

The difference between the two commands was the `-c` flag. Evidence that the network was in fact up:

- `/etc/libvirt/qemu/networks/default.xml` existed, timestamped at the `net-start`.
- The autostart symlink was in place: `autostart/default.xml → ../default.xml`.
- And the clincher: **dnsmasq running** with `--conf-file=/var/lib/libvirt/dnsmasq/default.conf`. That process only exists if the `default` network is active.

### Root cause

Without `-c`, `virsh` run by an unprivileged user doesn't connect to the system daemon: it defaults to **`qemu:///session`**, a per-user daemon that is an entirely separate space. Virtual networks don't exist in `session`, and not because of a bug — an ordinary user can't create bridges or NAT rules. The empty table was the correct answer to the wrong question.

`virbr0` showing `DOWN` was a separate red herring. The actual flag was **`NO-CARRIER`**: a bridge with no ports attached, like a powered switch with no cables. It goes `UP` when the first VM attaches its virtual interface.

### Fix

Pin the default URI persistently (shell: fish):

```fish
set -Ux LIBVIRT_DEFAULT_URI qemu:///system
```

`-U` makes it universal (fish persists it across sessions without editing config files) and `-x` exports it to child processes.

### Takeaway

`qemu:///system` and `qemu:///session` are two different worlds, not two views of the same one. A lab that needs bridges and NAT necessarily lives in `system`. Pin `LIBVIRT_DEFAULT_URI` on day one to avoid diagnosing ghosts.

**Incidental confirmation of the socket-activation theory:** `libvirtd.service` reported `enabled=disabled` but `active=active`. Nobody enabled it at boot; systemd woke it when the first `virsh` touched the socket — exactly the behaviour intended by enabling `libvirtd.socket` instead of the `.service`.

---

## #3 — Design correction: the BTF that matters is the guest's

**Date:** 2026-08-17 · **Module:** 1 (VMs) · **Status:** corrected before impact

### Symptom

No failure. A reasoning error caught before building.

### Diagnosis

During the initial host survey, the presence of `/sys/kernel/btf/vmlinux` was verified and the conclusion drawn that Falco could use modern eBPF without compiling modules. The check was correct, but it **applied to the discarded architecture** (k3s directly on the host).

In the chosen architecture, Falco runs as a DaemonSet **inside the VMs**. The kernel that needs BTF is the guest's. The host's BTF is irrelevant to this path.

### Fix

Move the BTF check inside each VM, and pick the guest distro accordingly: **Debian 13 (trixie)** instead of Debian 12, for its 6.12 kernel with better eBPF support. Debian kernels are built with `CONFIG_DEBUG_INFO_BTF=y`, so the plan doesn't change — what changes is where it's verified.

### Takeaway

When the architecture changes, re-audit the inherited technical assumptions, not just the steps. A correct check against the wrong component is indistinguishable from a failed check until something breaks three modules later.

---

## #4 — `virt-install --cloud-init` aborts the first boot and discards the seed

**Date:** 2026-08-17 · **Module:** 1 (VMs) · **Status:** resolved

### Symptom

After running `virt-install` for both nodes, the domains were defined but **powered off**, with no DHCP leases:

```
sudo virsh list --all              → k3s-server: shut off / k3s-agent: shut off
sudo virsh net-dhcp-leases default → empty table
sudo virsh console k3s-agent       → error: domain is not running
```

### Diagnosis

Three independent signals were cross-referenced, and all three pointed to the same place:

1. **The disk was never written.** `qemu-img info` reported `disk size: 196 KiB` on a 20 GiB virtual overlay. That's pure qcow2 metadata: zero bytes written by the guest. Had cloud-init run, `growpart` and `apt update` would have left tens of megabytes.
2. **The seed ISO had disappeared.** The domain XML showed a cdrom entry **with no `<source file>`**: the device existed, empty.
3. **The VM had started at least once.** The base image's owner had changed to `libvirt-qemu:libvirt-qemu` — the result of libvirt's *dynamic ownership*, which hands the file to the QEMU user when a domain starts.

A guest that boots, writes not a single byte, and shuts down leaving an empty cdrom isn't a guest that failed to boot: it's a guest whose boot was cut short from outside.

### Root cause

The flag combination **`--cloud-init` together with `--noautoconsole`**. The `--cloud-init` flag makes `virt-install` treat the boot as a two-phase install: phase 1 with the seed ISO attached, and once it considers that finished it detaches the ISO, redefines the domain and leaves it powered off. With `--noautoconsole` there's nothing waiting for the boot to complete, so phase 1 is declared over immediately: the VM dies before Debian comes up, and the seed is discarded.

Simply starting the domains wouldn't have been enough: without the seed ISO, cloud-init has nowhere to read the user or the SSH key from, and the result is a system you can't log into.

### Fix

Stop relying on `virt-install`'s implicit behaviour and build the seed explicitly.

**The NoCloud contract:** on boot, cloud-init looks for a device with the volume label **`cidata`** and reads two files from it with exact names and no extension: `user-data` and `meta-data`.

```bash
sudo virsh undefine k3s-server && sudo virsh undefine k3s-agent   # without --remove-all-storage

mkdir -p seed/server seed/agent
cp user-data-server.yaml seed/server/user-data
cp user-data-agent.yaml  seed/agent/user-data
printf 'instance-id: k3s-server-01\nlocal-hostname: k3s-server\n' > seed/server/meta-data
printf 'instance-id: k3s-agent-01\nlocal-hostname: k3s-agent\n'  > seed/agent/meta-data

xorrisofs -output /tmp/seed-server.iso -volid cidata -joliet -rock seed/server
xorrisofs -output /tmp/seed-agent.iso  -volid cidata -joliet -rock seed/agent
sudo mv /tmp/seed-*.iso /var/lib/libvirt/images/
```

And in `virt-install`, replace `--cloud-init` with the ISO attached as an ordinary cdrom:

```
--disk /var/lib/libvirt/images/seed-server.iso,device=cdrom
```

The overlays were reused without recreating them: at 196 KiB it was proven they held not a byte of guest data.

### Takeaway

Two lessons. The technical one: convenience flags that hide a state machine (`--cloud-init` and its two-phase install) fail in hard-to-read ways when combined with flags that alter that flow. Building the NoCloud seed by hand is four extra commands and removes all the ambiguity.

The methodological one: **none of the three signals was conclusive on its own.** The disk size could have been a guest that booted and didn't write; the empty cdrom, a VM that never had one; the ownership change, a successful boot. Together they left exactly one possible explanation. When a symptom is ambiguous, the way out is to add independent signals, not to stare harder at the same one.

### Side benefit

The explicit seed is reproducible and version-controlled: it ended up as declared infrastructure in the repository instead of the side effect of a flag. Better for the project than the solution that failed.

---

## #5 — The guests never leave firmware: SeaBIOS won't boot the image

**Date:** 2026-08-17 · **Module:** 1 (VMs) · **Status:** resolved

### Symptom

With the NoCloud seed built by hand, both domains started and stayed running. The bridge went `UP`, confirming attached virtual interfaces. But there were no DHCP leases:

```
sudo virsh list --all              → k3s-server: running / k3s-agent: running
ip -br addr show virbr0            → virbr0  UP  192.168.122.1/24
sudo virsh net-dhcp-leases default → empty table
```

### Diagnosis

The diagnosis proceeded by ruling out layers from the bottom up, and **three intermediate hypotheses turned out to be wrong**. They're worth recording, because the wrong path is informative too.

**1. Host networking: ruled out.** Interfaces `vnet2` and `vnet3` were attached to the bridge in `UP,LOWER_UP`. But `virbr0`'s ARP table was empty and `virbr0.status` was 0 bytes: the guests had emitted **not a single packet**. The problem wasn't that DHCP was failing — there was nobody talking on the other side.

**2. The hypervisor's counters reoriented everything.** Rather than fighting `qemu-img`'s write lock on a running VM, libvirt's counters were queried:

```
virsh domblkstat k3s-server vda   → rd_bytes 3372751872 (3.37 GB) / wr_req 0
virsh domifstat  k3s-server vnet2 → tx_packets 0 / rx_packets 1
virsh cpu-stats  k3s-server       → cpu_time 554 s
```

This revealed a guest that **was executing code and reading disk heavily**, but writing nothing and transmitting nothing. It wasn't hung doing nothing: it was working hard and getting nowhere.

**3. Software emulation (TCG): ruled out.** High CPU with zero progress suggested missing hardware acceleration. False: `<domain type='kvm'>`, `kvm support: enabled` in the QEMU monitor, and `/dev/kvm` at mode `666`.

**4. The decisive evidence: the QEMU monitor.** Querying the virtual CPU's registers:

```
virsh qemu-monitor-command k3s-server --hmp 'info registers'

CS = f000    base 000f0000     ← BIOS ROM segment
EIP = 0000f81a                 ← 16-bit real mode
HLT = 0                        ← actively executing, not halted
```

Ten minutes after starting, the CPU was still executing BIOS code in 16-bit real mode. Control was never handed to a kernel. `HLT=0` rules out "firmware halted showing an error message": it's retrying in a loop, and that loop explains both the 3.37 GB read and the CPU consumed.

An important nuance, to avoid over-reading it: `CS=f000` indicates BIOS code executing, which could be **SeaBIOS itself or an `INT 13h` call issued by GRUB** to read disk. Either way, we're before the kernel, in real mode.

**5. The base image isn't the culprit.** The partition table was inspected by extracting the first 4 MB with `qemu-img dd` and reading the raw GPT entries with `od`:

```
MBR offset 0:         eb 63 90            → GRUB's boot.img present
Partition signature:  55 aa               → valid
BIOS boot partition:  PRESENT             → BIOS boot supported
EFI System Partition: PRESENT             → UEFI boot supported
Partition 0:          4f68bce3-e8cd-...   → Linux x86-64 root
```

Debian 13's `genericcloud` image is hybrid-boot and supports both paths.

### Root cause

SeaBIOS fails to complete the GRUB chainload from this image and enters a retry loop. The exact mechanism of the loop was left undetermined: the decision was made not to continue the archaeology, because an alternative path — supported by the image itself and strictly better — existed.

### Fix

Move the boot from legacy BIOS to **UEFI with OVMF** (`edk2-ovmf`, already installed). Three reasons, in order of weight:

1. The EFI partition is present in the image, so the path is supported out of the box.
2. **OVMF writes to the serial port by default.** With `--graphics none` there is no video device, and SeaBIOS sends its messages to the VGA framebuffer — which is why `virsh console` returned a blank screen and the failure was invisible. UEFI fixes the observability problem, not just the boot problem.
3. UEFI is what real cloud providers use for these images.

```bash
sudo virsh destroy k3s-server && sudo virsh undefine k3s-server --nvram
sudo virsh destroy k3s-agent  && sudo virsh undefine k3s-agent  --nvram
# then recreate, adding --boot uefi to virt-install
```

The `--nvram` on `undefine` matters: with UEFI each VM has its own NVRAM variables file, and leaving a stale one behind can contaminate the next boot.

### Takeaway

**When the guest won't talk, ask the hypervisor.** The instinct to look at the disk file collides with the running VM's write lock; `domblkstat`, `domifstat`, `cpu-stats` and the QEMU monitor give the same information without touching the file, and more precisely. The vCPU registers turned ten minutes of speculation into an unambiguous answer.

**Observability is part of the configuration, not an extra.** `--graphics none`, chosen without considering where the firmware writes, created a blind spot exactly where the failure happened. The lesson applies directly to the rest of the project: if Falco detects something and nobody receives the event, that's the same design error wearing a different hat.

**Ruling things out is useful even when it doesn't find the cause.** Three hypotheses fell before the right one — host networking, TCG emulation, an image without BIOS support — and each shrank the search space. And at the point where determining the exact mechanism of the SeaBIOS loop would have cost more than routing around it, routing around it is correct: the goal is a working cluster, not a forensic report on firmware.

### Confirmation

After adding `--boot uefi`, the hypervisor's counters changed unambiguously:

| Metric | SeaBIOS | UEFI/OVMF |
|---|---|---|
| vCPU registers | `CS=f000`, 16-bit real mode | 64-bit, `ffffffffa30...` addresses (kernel space) |
| `wr_req` | 0 | 1106 (317 MB written) |
| `tx_packets` | 0 | 26 |

Those 317 MB written are `growpart` + `resize2fs` expanding the root partition from 3 to 20 GB — which also **proves cloud-init ran** and that the NoCloud seed was read correctly.

---

## #6 — `ufw` drops the virtual network's DHCP

**Date:** 2026-08-17 · **Module:** 1 (VMs) · **Status:** resolved

### Symptom

With the guests booting correctly under UEFI, there were still no DHCP leases. The nodes came up, sat idle, and had no IP address.

### Diagnosis

The sequence that narrowed it down, ruling out from the bottom up:

**1. The guests genuinely have no IP.** A ping sweep across `192.168.122.2-30` left every ARP entry `INCOMPLETE`: nobody answered. This wasn't a lease-recording problem, it was a real absence of addressing.

**2. A self-inflicted false positive, worth recording.** A first socket check concluded there was no DHCP server listening. That was a command error: the `grep` carried a `head -5` that cut off the relevant line. Re-run without truncation, `dnsmasq` *was* listening on `0.0.0.0%virbr0:67`. **Lesson: truncating the output of a diagnostic can manufacture the opposite conclusion.**

**3. The decisive piece: dnsmasq logged nothing.** When dnsmasq receives a request it writes `DHCPDISCOVER(virbr0) <mac>` to syslog. Zero entries since the VMs started. It wasn't rejecting the packets: it **never received them**.

**4. Narrowing where they're lost.** The guest was receiving the bridge's STP BPDUs, which proves layer 2 works in both directions (BPDUs don't traverse IP rules). With the guest transmitting, the bridge operational and dnsmasq listening, the packet could only be lost in netfilter, between `virbr0` and the socket.

**5. The cause.** Checking the host's firewalls:

```
ufw    enabled=enabled    active=active
```

### Root cause

`ufw` was active on the host with its default policy of **denying all inbound traffic**. The DHCP DISCOVERs reached `virbr0`, entered netfilter, and were dropped before reaching dnsmasq. This is a known incompatibility between ufw and libvirt's virtual networks: libvirt installs its own rules in nftables' `libvirt_network` table (verified present), but that doesn't exempt the traffic from ufw's rules.

A side effect that would have bitten later: ufw also blocks forwarding, so even with static addresses the guests would have had no outbound connectivity, and couldn't have downloaded k3s or container images.

### Fix

Explicit, minimal rules instead of disabling the firewall:

```bash
sudo ufw allow in on virbr0 to any port 67 proto udp comment 'libvirt DHCP'
sudo ufw allow in on virbr0 to any port 53 comment 'libvirt DNS'
sudo ufw route allow in on virbr0 out on wlan0 comment 'k3s lab egress'
sudo ufw reload
```

The first two enable the services the host provides to the virtual network (DHCP and DNS via dnsmasq). The third enables forwarding out to the uplink, needed to download packages and images.

The guests' DHCP clients had already exhausted their retries, so the VMs had to be restarted to make them request an address again.

### Takeaway

**The host firewall is part of the lab's topology, not an environment detail.** libvirt installs its nftables rules correctly and the traffic still dies: two firewall systems coexisting, each correct on its own, incompatible together. Worth noting because it's exactly the class of interaction you later have to reason about with Kubernetes NetworkPolicies.

**"Nobody logged anything" is a signal, not an absence of signal.** That dnsmasq had not a single entry was more informative than any error message: it ruled out all of the server's processing at once and moved the search upstream, to the filtering layer.

**The firewall stayed on.** Disabling ufw would have fixed the symptom in one command, and would have been the wrong call in a security project. The rules were kept minimal, commented and documented.

---

## #7 — The Kyverno policy that worked and could be bypassed two ways

**Date:** 2026-08-17 · **Module:** 4 (Kyverno) · **Status:** resolved (see #8)

### Symptom

There wasn't one. That's the point of this entry.

The first version of the policy forbidding the `latest` tag applied without errors, reported `Ready`, and correctly blocked the test case:

```
kubectl run bad --image=nginx:latest -n demo
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request
```

Everything suggested the policy was done.

### Diagnosis

Negative cases the original test didn't cover were tried. Two got through:

```
initContainers with nginx:latest  →  pod created   ← bypassed
image: nginx  (no tag)            →  pod created   ← bypassed
containers with nginx:latest      →  BLOCKED       ← control OK
```

**Bypass 1 — uncovered containers.** The `pattern` validated only `spec.containers`. A Pod has three container lists: `containers`, `initContainers` (which run before the main ones, with access to the same volumes) and `ephemeralContainers` (injected by `kubectl debug` into a running pod). Anyone able to create pods could smuggle in any image through the two unvalidated lists.

**Bypass 2 — the implicit tag.** `image: nginx` doesn't contain the string `:latest`, so the glob `!*:latest` doesn't match it. But the runtime resolves it to `nginx:latest` regardless. The policy covered the explicit mistake and let through the implicit one — which is precisely the one people make without noticing.

### Root cause

The policy was validated against the case it was expected to fail, not against the set of cases it was supposed to cover. An admission rule isn't proven by showing it blocks what it means to block, but by verifying that **no path exists** for what it means to forbid.

### Fix

Rewritten with **two rules** instead of one:

1. `require-image-tag` — pattern `"*:*"`: requires an explicit tag, closing bypass 2.
2. `validate-image-tag` — pattern `"!*:latest"`: forbids `latest`, the original validation.

And each rule covers all three container lists, using Kyverno's *conditional anchor*:

```yaml
pattern:
  spec:
    containers:
      - image: "*:*"
    =(initContainers):
      - image: "*:*"
    =(ephemeralContainers):
      - image: "*:*"
```

The `=(field)` syntax means "if this field exists, validate it". Without the anchor, a Pod with no `initContainers` would fail validation for not declaring a field it has no business having, and the policy would break every legitimate deploy in the cluster.

Two rules were needed rather than one: both validations apply to the same key (`image`), and a single YAML `pattern` can't contain the same key twice (see #8).

### Takeaway

**A security policy is proven with the cases that should fail, not the ones that should pass.** The happy path confirms the rule is installed; the negative cases confirm it's useful.

**Enumerate the full surface before writing the rule.** Both bypasses came from the same error: assuming "a pod's container" is one place. It's three, and two of them are interesting to an attacker precisely because they get forgotten.

**Incidental observation: autogen.** Kyverno automatically generated `autogen-resource-rule`, a variant of the rule for pod controllers (Deployment, DaemonSet, StatefulSet, Job, CronJob). Without it the rejection would happen when the ReplicaSet tried to create the pod, and the error would end up in events instead of returning to the terminal of whoever ran the `apply`.

---

## #8 — Three silent bugs in the hardened policy

**Date:** 2026-08-17 · **Module:** 4 (Kyverno) · **Status:** resolved

### Symptom

The second version of the policy, written to close the bypasses from #7, had three defects. Only one showed up as a visible error.

### Diagnosis

**Bug 1 — invalid name (failed loudly).** The object was called `ExecutionPolicy`:

```
The ClusterPolicy "ExecutionPolicy" is invalid: metadata.name: Invalid value:
"ExecutionPolicy": a lowercase RFC 1123 subdomain must consist of lower case
alphanumeric characters, '-' or '.'
```

Kubernetes object names are RFC 1123 DNS subdomains and don't allow uppercase. It isn't a style convention: those names end up forming part of real DNS names inside the cluster. This was the benign bug, because it stopped the file from being applied at all.

**Bug 2 — duplicate YAML key (failed silently).** Within the same `pattern`, the key `containers` appeared twice — once with `"*:*"` and once with `"!*:latest"`. In a YAML mapping a repeated key produces no error: **the last one overwrites the first**. The explicit-tag check on `containers` was discarded without warning.

**Bug 3 — capitalization error (failed silently).** `=(initcontainers)` was written in lowercase; the real API field is `initContainers`. Combined with the anchor's semantics the effect is particularly deceptive: `=(field)` validates *if the field exists*, and a field called `initcontainers` never exists in a Pod. The rule **didn't fail: it was skipped**.

The dangerous scenario was 1, 2 and 3 together. Fixing only the name, the policy would have applied, reported `Ready`, blocked `nginx:latest` in `containers` — and left both bypasses from #7 intact, with every appearance of working.

### Residual bug (since fixed)

After fixing the name and splitting the two rules, a divergence remained between them:

```yaml
- name: require-image-tag        # rule 1
      =(initcontainers):         # lowercase  ← not fixed
- name: validate-image-tag       # rule 2
      =(initContainers):         # camelCase  ← fixed
```

The typo was corrected in rule 2 and not in rule 1, with a very precise effect:

```
initContainer with latest   →  BLOCKED   (caught by rule 2)
initContainer with NO tag   →  created   ← the open hole
container with latest       →  BLOCKED
container with no tag       →  BLOCKED
compliant pod               →  created   (no false positives)
```

The hole survived exactly at the intersection of the two conditions: initContainer **and** omitted tag. It was closed afterwards, and the negative-test suite now passes 5 of 5.

### Takeaway

**Bugs that fail loudly are the cheap ones.** Of the three, the only one that surfaced was the invalid name. The other two would have stayed in the repository, inside a policy reported as `Ready`, with the validations silently disabled.

**A copy-paste divergence between two rules is invisible in code review.** The two rules *look* the same when you read them. The only way the difference was found was the test case combining both conditions.

**Hence the decision to version the negative cases** under `tests/`, one file per case, runnable together with `kubectl apply -f tests/ --dry-run=server`. The natural next step is the `kyverno test` CLI, which declares each case's expected result and runs without a cluster: that's what allows policies in CI, so a pull request can't merge a rule that opens a hole.

**Methodological note:** during this session bash heredocs (`<<'EOF'`) were suggested for the tests — syntax the environment's shell (fish) doesn't support. The incident redirected the solution toward something better: test cases as version-controlled files in the repository, instead of commands typed into a terminal and lost.

---

## #9 — A detection that did fire and was assumed absent: the priority filter

**Date:** 2026-08-18 · **Module:** 5 (Falco) · **Status:** resolved

### Symptom

After installing Falco and simulating an intrusion (`kubectl exec` into a pod, `cat /etc/shadow`), **only one alert** appeared: the sensitive-file read. The shell being opened didn't show up, despite being the more obvious of the two events.

The first hypothesis was that the rule wasn't in the loaded set: in recent Falco versions the default ruleset was trimmed and several noisy rules moved to the `incubating` and `sandbox` sets, which aren't loaded by default.

### Diagnosis

The hypothesis was wrong. Inspecting the set actually loaded inside the pod:

```
total rules loaded: 25
- rule: Run shell untrusted
- rule: Terminal shell in container    ← present
```

The rule was loaded. Extracting it in full revealed the cause:

```yaml
- rule: Terminal shell in container
  condition: spawned_process and container and shell_procs and proc.tty != 0 and container_entrypoint ...
  priority: NOTICE          ← here
```

The command used to review alerts was `kubectl logs ... | grep -i warning`. The rule's priority is **NOTICE**, so the filter discarded it. Listing without filtering:

```
Notice   | Terminal shell in container      | intruso | sh
Warning  | Read sensitive file untrusted    | intruso | cat /etc/shadow
```

Both alerts had fired from the very first moment.

### Root cause

A filter applied to one's own diagnostic output, without verifying that the filtered severity range included what was being looked for.

Falco's priority scale, highest to lowest: `EMERGENCY`, `ALERT`, `CRITICAL`, `ERROR`, `WARNING`, `NOTICE`, `INFORMATIONAL`, `DEBUG`.

### Takeaway

**This is the project's third incident with the same shape:** the `head -5` that led to concluding there was no DHCP server (#6), the test case that didn't cover the intersection of two conditions (#8), and this severity filter. In all three, **the measuring instrument produced the conclusion**, and in none of them did it announce that it was truncating.

The version of this that applies to the domain is direct and serious: a SOC filtering its dashboard on `severity >= WARNING` misses all reconnaissance activity, which is mostly NOTICE — and reconnaissance is what precedes everything else. A detection pipeline that discards signal by default is indistinguishable from one that doesn't detect.

---

## #10 — The token-theft rule: path aliases, truncated fields and 804 false positives

**Date:** 2026-08-18 · **Module:** 5 (Falco) · **Status:** resolved

### Context

A custom rule to detect reads of the Service Account token (`/var/run/secrets/kubernetes.io/serviceaccount/token`), the classic route from "code execution in a container" to "API server access with the pod's identity". This is a case **admission control cannot cover**: the token is legitimately mounted by the kubelet and reading it is a normal operation for any SDK. It's only distinguishable by behaviour.

Three distinct problems appeared, in three distinct layers.

### Problem 1 — Path aliasing: two routes to the same file

The original condition used `fd.name startswith "/var/run/secrets/kubernetes.io/serviceaccount/"`.

The mount path turned out to differ per image:

```
busybox:1.37        /var/run is a real directory   →  /var/run/secrets/kubernetes.io/serviceaccount
nginx:1.29-alpine   /var/run → ../run (symlink)    →  /run/secrets/kubernetes.io/serviceaccount
```

Same `mountPath` in the pod spec, same kubelet, and the kernel mounts them at different paths because the runtime resolves the image's symlink before creating the mount point.

**Correction of a wrong intermediate diagnosis.** The conclusion drawn from that was that the rule would catch the theft in the busybox pod and miss it in the nginx pod. Verified afterwards, that was false: `fd.name` reports **the path as the process opened it**, not the resolved mount point, so `cat /var/run/...` in the alpine pod matches too.

But the fix was still necessary, for a more serious reason: **the attacker picks the path.** Verified directly:

```
cat /var/run/secrets/.../token  →  fd.name = /var/run/secrets/.../token
cat /run/secrets/.../token      →  fd.name = /run/secrets/.../token      ← same pod
```

With the original rule, the second command went invisible. It isn't a question of which image is used — it's that **two different paths lead to the same file, and path-based detection has to enumerate all of them**: the path-aliasing class of evasion.

The fix uses Falco's `pmatch` operator, which understands directory hierarchy rather than comparing strings:

```yaml
and fd.name pmatch (/var/run/secrets/kubernetes.io/serviceaccount, /run/secrets/kubernetes.io/serviceaccount)
```

### Problem 2 — The output template defines the data schema

While trying to tune the rule, the process fields came back empty:

```
proc.name → null    proc.pname → null    proc.tty → null
proc.cmdline → "cat /var/run/secrets/..."    ← this one worked
```

**Falco only includes in the JSON the fields referenced in the `output` template.** `proc.cmdline` was there because it appeared in the template; the others didn't exist in the event at all.

The consequence goes beyond debugging: **you cannot filter in a SIEM on a field you never emitted.** The `output` isn't cosmetic, it's the schema definition. If a field is missing, you have to modify the rule and wait for the event to happen again.

Hence the ordering rule: **visibility first, tuning second.** You can't tune what you can't see.

### Problem 3 — 804 alerts, 2 real

With the fields visible, the volume became apparent:

```
804 total alerts
  579  namespace kyverno
  222  namespace kube-system
    3  namespace demo    ← 2 are the simulated attacks
```

Signal-to-noise: **0.25%**. A technically correct and operationally useless rule: nobody finds those two lines.

The discriminator came from measuring, not guessing:

```
8 reports-control     ← the apps' own binaries reading their token to authenticate
6 kyverno
5 background-cont
4 cleanup-control
2 metrics-server
1 traefik
1 cat                 ← the attack
```

**Incidental finding: `proc.name` is truncated at 15 characters.** It comes from the kernel's `comm` field, which has that hard limit — hence `reports-control` and `background-cont`. A filter on `proc.name = "background-controller"` would never have matched, producing the project's fourth silent failure. For long names, `proc.exepath` is the right field.

**And the counterexample, in the other direction:** on Alpine, `cat` is a symlink to busybox, so the same attack reports `proc.name=cat` but `proc.exepath=/bin/busybox`. A filter on `proc.exepath in (/bin/cat)` would have missed that pod. Neither field is universally correct: `proc.name` truncates, `proc.exepath` collapses multi-call binaries. You have to know which failure each rule is guarding against.

The fix — filter on *who* reads, not on *where*:

```yaml
- list: token_reader_tools
  items: [cat, head, tail, more, less, base64, xxd, od, strings, curl, wget, nc, socat, tar, cp, dd]

- macro: suspicious_token_reader
  condition: (proc.name in (token_reader_tools) or proc.name in (shell_binaries))
```

Result: **from 804 alerts to 1**, keeping the detection working across both images.

The alternative of excluding by namespace (`not k8s.ns.name in (kube-system, kyverno)`) was explicitly rejected: it's simpler and worse, because it blinds you to an attacker who gains execution in `kube-system` — which is exactly where it would hurt most.

### Declared limitation

The rule filters on process name, so it **does not detect an attacker who reads the token from their own binary** (a Go or Python implant). It covers opportunistic access with command-line tooling, not a prepared adversary. This is written into the rule's `desc`: an honest `desc` about the limits is worth more than one promising total coverage.

### Takeaway

**Tuning is half the work, and it's done with data.** The right discriminator isn't guessed: you emit the field, measure the distribution, and decide on the evidence. Of the two hypotheses raised before measuring — that the noise came from the apps' own binaries, and that the symlink problem varied by image — one was right and one was wrong.

**An untuned detection isn't half a detection, it's none.** 804 events with 2 true positives don't get triaged: they get ignored. And a rule that gets ignored is worse than no rule, because it manufactures a feeling of coverage.

---

## #11 — Exposed evidence: any pod could delete the Falco indices

**Date:** 2026-08-20 · **Module:** 7 (Kubernetes security) · **Status:** mitigated

### Symptom

None. It was found while auditing the cluster's state before writing policies, not from a failure.

### Diagnosis

Elasticsearch is deployed with `xpack.security.enabled=false` (a conscious, documented decision from module 6). Without NetworkPolicies, that meant the Service `elasticsearch.elk.svc.cluster.local:9200` was reachable from **any pod in the cluster**, unauthenticated.

Verified from the attack-simulation pod, in three steps:

```
GET  /_cat/indices    → lists every index, including falco-2026.08.20
PUT  /prueba-netpol   → index created
DELETE /prueba-netpol → index deleted
```

A throwaway test index was used; `falco-*` was never touched.

**A relevant detail of the technique:** BusyBox's `wget` doesn't support `--method`, so the writes were done with `nc`, hand-crafting the HTTP request:

```sh
printf 'DELETE /prueba-netpol HTTP/1.1\r\nHost: es\r\nConnection: close\r\n\r\n' | nc elasticsearch.elk.svc.cluster.local 9200
```

*Living off the land* with what ships in a minimal image. "The image is small" is not a security control.

### Root cause

The SIEM lived inside the trust boundary of the system it watches, with neither authentication nor network isolation. An attacker compromising any container could **delete the indices that recorded their own intrusion**.

### Fix

Three NetworkPolicies in the `elk` namespace:

1. `default-deny-ingress` with `podSelector: {}` — closes the whole namespace.
2. `allow-elasticsearch` — port 9200 only from the `falco` namespace (via `namespaceSelector` on the automatic `kubernetes.io/metadata.name` label) **or** from pods labelled `app: kibana`.
3. `allow-kibana` — port 5601 from `ipBlock: 192.168.122.0/24`.

The `ipBlock` is necessary because traffic entering via the NodePort **doesn't come from a pod**: kube-proxy SNATs it to the node's IP, so no `podSelector` can match it.

**Verification, with a positive control:**

```
DNS from the attacking pod   resolves to 10.42.1.29     ← the block isn't a DNS failure
attacking pod → 10.42.1.29   BLOCKED
falcosidekick → ES           REACHABLE                  ← positive control
Kibana from the host         HTTP 200
Kibana logs                  no errors against ES
```

The positive control is what validates the test: had `falcosidekick` also been blocked, there'd be no way to know whether the policy discriminates or simply broke everything.

### Takeaway

**The semantics of `from` in a NetworkPolicy hinge on one dash.** Two separate list items are **OR**; two selectors under the same item are **AND**:

```yaml
from:
  - namespaceSelector: {...}     # OR
  - podSelector: {...}
from:
  - namespaceSelector: {...}     # AND — here, the empty set
    podSelector: {...}
```

**A silent failure to watch for:** k3s ships its own embedded NetworkPolicy controller, but if it's started with `--disable-network-policy` the API server **accepts** the objects and nothing enforces them. Policy created, zero effect, no error. Only a connectivity test catches it.

### What this doesn't solve

- **No egress control.** Ingress into `elk` is closed; pods can still reach anywhere.
- **Elasticsearch still has no authentication.** The control is network-only: anyone compromising a pod in the `falco` namespace or the Kibana pod retains full read and delete access.
- **The `falco` namespace is now a privileged path** to the evidence, on top of running the most privileged pod in the cluster. It needs its own RBAC.

---

## #12 — Hardening without killing the security tool

**Date:** 2026-08-20 · **Module:** 7 (Kubernetes security) · **Status:** resolved

### Context

Applying Pod Security with Kyverno: forbid `privileged` and require `runAsNonRoot`. The underlying problem is that **Falco is the most privileged component in the cluster**: its DaemonSet runs `privileged: true` and mounts `/boot`, `/lib/modules`, `/usr`, `/etc`, `/sys/kernel`, `/proc` and every container-runtime socket from the host. A badly designed restrictive policy stops the DaemonSet from recreating its pods, and **the result is losing detection without anything failing visibly**: the DaemonSet simply never reaches its desired replicas.

### Inventory first

The policy was applied in **`Audit`** mode first, which rejects nothing and writes results to the `policyreport` objects. Result across the whole cluster:

```
29 violations across 4 namespaces
  require-run-as-nonroot   kube-system 9 · falco 7 · elk 5 · demo 5
  disallow-privileged      falco 3
```

Two conclusions: **nothing** in the cluster declared `runAsNonRoot`, and privilege was concentrated in a single identified, justified component.

That order — audit, measure, decide exemptions, then enforce — is the same loop used to tune the Falco rule in #10: visibility first, restriction second. Enforcing blind on a cluster with live workloads is how you cause incidents in the name of preventing them.

### Exemption criteria

**For `disallow-privileged`, a surgical exemption.** The obvious move is to exempt the whole `falco` namespace; that's wrong, because `falcosidekick` lives there too and needs no privileges at all. The exclusion matches on namespace **and** label:

```yaml
exclude:
  any:
    - resources:
        namespaces: [falco]
        selector:
          matchLabels:
            app.kubernetes.io/name: falco
```

**For `require-run-as-nonroot`, a criterion of ownership.** `kube-system` isn't ours: CoreDNS, local-path-provisioner and svclb are managed by k3s, and the next upgrade overwrites any change. Exempt and document. But `demo` and `elk` *are* ours, and there the right answer **isn't an exemption, it's fixing the workloads**: if you exempt everything that fails, the policy protects nothing and only produces a feeling of compliance.

### The five defects found in review

1. **The `exclude` blocks were missing entirely, with both rules already at `Enforce`.** The serious one: applying it would have stopped Falco from recreating its pods.
2. **The `web` image read `nginx-unprivileged:1.29-alpine`, without the `nginxinc/` prefix.** Verified against the registry: `docker.io/library/nginx-unprivileged` returns `object not found`. Guaranteed `ImagePullBackOff`.
3. **The `web` Deployment had no `securityContext`.**
4. **That `securityContext` had ended up as a root-level field of `03-demo-service.yaml`**, where it does absolutely nothing. It landed in the wrong file.
5. **Kibana had no `securityContext` either**, so it would have been rejected on its next restart.

And once again the drift between edited file and cluster: the policy hadn't been applied. Fourth occurrence in the project; this time it prevented the damage.

### A finding about `runAsNonRoot`

Elasticsearch and Kibana **were already running as uid 1000**: the changes to them were purely declarative. That doesn't make them pointless. `runAsNonRoot: true` makes the **kubelet refuse to start** the container if the image's effective user resolves to root. Without declaring it, it runs as 1000 today because the image says so — and a pull of a future version that changes the `USER` puts the process back to root with nobody noticing. Declaring it turns a habit into a verified guarantee.

The practical corollary showed up when recreating the simulation pod: `runAsNonRoot: true` **alone is not enough** for an image that declares `USER root`, like busybox. The kubelet rejects the start with `CreateContainerConfigError` ("container has runAsNonRoot and image will run as root"). You have to specify `runAsUser` explicitly.

### Verification

```
privileged pod            REJECTED by disallow-privileged AND require-run-as-nonroot
pod without runAsNonRoot  REJECTED
compliant pod             ACCEPTED
Falco after deleting pods 2/2 ready, modern BPF probe, custom-rules.yaml loaded
web                       HTTP 200 as uid=101(nginx)
namespace demo            no violations
```

Deleting Falco's pods is the test that validates the exemption design. Without it, the failure would surface weeks later, on the next node reboot.

### Second-order effect: hardening also reduces telemetry

With the simulation pod running as uid 1000 instead of root, the same attack produces a different result:

```
cat /var/run/secrets/.../token   → read                → Falco ALERT
cat /etc/shadow                  → Permission denied   → NO alert
```

It used to produce two alerts; now it produces one. The reason is that Falco's `open_read` macro requires `fd.num >= 0` — an actually-opened descriptor: **a failed open doesn't match.** The hardening blocked the attack *and* erased the record of the attempt.

It isn't a problem — the attack didn't work — but it's an interaction between the prevention and detection layers worth keeping in mind: **as you harden workloads, the dashboard sees less, and part of what it stops seeing is failed attempts that do have forensic value.** Detecting rejected attempts requires rules built on different conditions.

### Documented follow-up

`falcosidekick` can run non-root: it should be configured via `falco/values.yaml` so the `require-run-as-nonroot` exclusion can be narrowed to the DaemonSet's label instead of exempting the whole namespace.
