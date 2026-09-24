# Talos Linux won't boot: all-zero UUID on Hetzner dedicated servers

I just needed to upgrade my Kubernetes cluster. My Hetzner dedicated server had other ideas.

<!--more-->

## The setup

I run my homelab on a Kubernetes cluster using Talos Linux. The hardware is a Hetzner dedicated server I bought at auction, a Fujitsu ESPRIMO with a hardware RAID controller.

```text
Hetzner dedicated server (Fujitsu ESPRIMO-FTS, D3401-H2)
  Talos Linux v1.10.4
  Disk: /dev/sda (LogicalDrv 0, hardware RAID)
    sda1: STATE    (LUKS encrypted)
    sda2: BIOS boot
    sda3: BOOT     (XFS)
    sda4: META     (1MB, Talos metadata)
    sda5: EPHEMERAL (LUKS encrypted)
```

Over the past few months, I have been experimenting with various ways to run my homelab. I have a few services currently running at home on a small fleet of Raspberry Pis, and I have been planning to migrate them to something more robust. I tried configuring everything using various different orchestration methods, but none felt really intuitive to me.

I tried Portainer, but that seems to be going towards the closed source route (I know it is not yet, but the trajectory is already set). Then I tried Komo.do. I really wanted it to work, but the workflow was just not great. The system writes back to the git, and I really struggled to get automated syncs to work, and certainly there were a few places where the sync just refused to happen because of a chicken and egg situation that I cannot remember now (IIRC it was a file had to be written but the sync depended on it or something).

As someone who does a lot of platform engineering, I work with Kubernetes every day, which is a fairly complex system especially when start thinking about the ecosystem. There are various homelab targetted systems that try to address the complexity of Kubernetes. I think many have addressed a lot of the complexity of _administrating_ Kubernetes, however, I think the workflows they target do not take into account _users_ of the software.

So I gave in, I went back to what I am comfortable with. _Kubernetes here I come!_

As you saw from a [blog a few months ago](/blog/k3s-hetzner-egress-debugging/), I was setting up K3s, but at some point, I got reminded of Talos, and I had always wanted to mess around with it. I had not migrated anything over to the K3s intance yet, so this was the perfect time to try it. I had installed it and getting from zero to Kubernetes was a much smoother installation than K3s. Once installed, I was not really able to configure the system much besides installing ArgoCD for GitOps, as I went travelling for a long while.

## The upgrade that broke everything

But once I sat still for a little, I went back to it, and I had a bunch of upgrades waiting for me. I had a broader 8-step upgrade path planned for the whole stack, and step one was moving Talos from v1.10.4 to v1.10.9 before anything else.

The plan was simple:
1. Patch-first: Talos v1.10.4 to v1.10.9.
2. Then step through each Talos minor: v1.11.6, v1.12.9, v1.13.4.
3. Then Kubernetes, one minor at a time: v1.33 to v1.34 to v1.35 to v1.36.
4. Finally, we update `talenv.yaml` at the end.

Sounds simple! And really it probably would have been if it wasn't for this initial issue.

Per the docs, the first step is pretty easy:

```bash
talosctl upgrade --nodes "<node-ip>" --image "factory.talos.dev/installer/<schematic>:v1.10.9"
```

And so the upgrade applied, and it triggered a reboot. But the server never came back. I could not ping it or reach the API.

Initially I didn't think much of it. I searched very little, looked at the Hetzner vKVM, but didn't investigate much. Since I had no data on there, I figured it'd be a lot easier to just reinstall it. So I did. But not long after I had to do another reboot, and so it happened again. This time we put on our Superman cape, and rescuse it.

## Rescue mode investigation

So I went digging to understand why this was happening. The system was practically empty, so there was no reason for this to happen.

Hetzner lets you boot into a rescue system (PXE-booted Debian) when your server is unresponsive. From there I could inspect the disks directly. The following commands and outputs are excerpts from my rescue host scripts, with wrappers and locale warnings omitted.

First thing I checked was whether the disk was even intact:

```bash {filename="rescue mode"}
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT
NAME    SIZE TYPE FSTYPE      LABEL MOUNTPOINT
loop0   3.8G loop ext2              
loop1    10M loop ext2              
sda     1.8T disk                   
|-sda1  100M part crypto_LUKS       
|-sda2    1M part                   
|-sda3 1000M part xfs         BOOT  
|-sda4    1M part                   
`-sda5  1.8T part crypto_LUKS       
```

Partitions were all there. The BOOT partition (sda3) mounted fine and contained both the current and upgraded Talos kernel/initramfs (Talos uses A/B slots for upgrades). So the upgrade itself had written successfully. The problem was during boot.

## Finding the error

I checked the Talos console, and from there I could see the boot TUI stuck at stage "Booting":

![Talos boot console showing all-zero UUID](/talos-rescue-mode-all-zero-uuid.png)

The UUID field at the top reads `00000000-0000-0000-0000-000000000000`, which looks odd. We will get back to that. More importantly, there is no hostname, Kubernetes status is unknown, and NTP is timing out. The system was alive but stuck: it could not decrypt the STATE partition, so it could not load the machine config, so nothing could proceed. The console showed the zero UUID.

The error on the console, transcribed from my notes rather than captured as terminal output:

```text
error opening encrypted volume: no handlers available to get encryption keys from:
1 error occurred: machine UUID 00000000-0000-0000-0000-000000000000 entropy check failed
```

So it seems the UUID is the machine UUID, i.e. the [SMBIOS UUID](https://cyberraiden.com/2026/08/03/smbios-uuid/).

## Why the UUID matters

Talos's `KeyNodeID` encryption works like this:

1. Read the machine's SMBIOS UUID (from DMI/BIOS tables)
2. Combine it with the partition label to form the LUKS passphrase
3. The passphrase format is literally `<UUID><partition_label>`, e.g. `<uuid>STATE`

Aah, right. Okay, so Talos encrypts the STATE and EPHEMERAL partitions with LUKS, and the encryption key is derived from the machine's SMBIOS UUID combined with the partition label. This is the [`nodeID` encryption method](https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/storage-and-disk-management/disk-encryption) in Talos, which ties the disk to the specific hardware it was installed on.

This is not a secret key by itself, but since the fixed-label concatenation yields the LUKS passphrase, you should not publish the UUID in a post about the machine. `nodeID` binds the disk to the machine, it does not keep the key hidden.

If the UUID is all zeros, the passphrase is wrong, LUKS will not open, and the STATE partition (which held the machine config, the node identity, and the saved platform network config) is inaccessible. In fact, Talos has a [very basic entropy check](https://github.com/siderolabs/talos/blob/main/internal/pkg/encryption/keys/nodeid.go) before deriving the key: it counts character frequency in the UUID string, and if any single character appears more than half the string length (strictly more than 18 out of 36 characters), it bails. An all-zero UUID has `0` appearing 32 times, so it fails immediately. I suppose this check is there to prevent using a predictable or "empty" UUID as a key source.

So, Talos cannot load its machine config because it is on the encrypted STATE partition, and it cannot decrypt STATE because it is reading a bad UUID.

## But the hardware UUID is fine

From rescue mode, I checked the actual hardware UUID:

```bash {filename="rescue mode"}
$ dmidecode -s system-uuid
bc94b891-0df9-47d8-a7d3-c7c199d66b0e
```

A perfectly valid UUID. The hardware is reporting it correctly. The rescue system sees it fine. So why does Talos see all zeros? We can even verify this by trying to open the LUKS volume manually.

First, a failed attempt with just the UUID:

```bash {filename="rescue mode excerpt"}
$ UUID="bc94b891-0df9-47d8-a7d3-c7c199d66b0e"
$ echo -n "$UUID" | cryptsetup open --type luks2 /dev/sda1 talos-state --key-file=-
No key available with this passphrase.
$ echo "exit: $?"
exit: 2
```

And then the correct one with the partition label appended:

```bash {filename="rescue mode excerpt"}
$ UUID="bc94b891-0df9-47d8-a7d3-c7c199d66b0e"
$ echo -n "${UUID}STATE" | cryptsetup open --type luks2 /dev/sda1 talos-state --key-file=-
$ echo "exit: $?"
exit: 0
```

That opened it. So why was Talos seeing zeros?

## The META partition

Talos has a small (1MB) metadata partition called META. It stores key-value pairs used during early boot, before the STATE partition is available.

It's a binary format where each 256 KB block (ADV) starts with `magic1` (`0x5a4b3c2d`) and ends with `magic2` (`0xa5b4c3d2`). There is also a SHA-256 checksum calculated over the entire block with the checksum field itself zeroed out during the calculation. This is why a simple raw writer would fail: if you do not update the checksum and the magic2 trailer, Talos will reject the block as corrupt.

Let's see what's on there

```bash {filename="rescue mode excerpt"}
$ mkdir -p /mnt/state
$ mount /dev/mapper/talos-state /mnt/state
$ ls -la /mnt/state
total 20
drwx------. 2 root root    80 Jun 19 02:05 .
drwxr-xr-x  1 root root    60 Jun 21 20:46 ..
-rw-------. 1 root root 11706 Jun 19 02:05 config.yaml
-rw-------. 1 root root    53 Jun 19 02:05 node-identity.yaml
-r--------. 1 root root   192 Jun 19 02:05 platform-network.yaml
```

I've now realised that the data is still there, and I can't just rely on reinstalling. There is something fundamentally wrong with the way Talos interacts with this board. So I ended up searching for the error in the code, and traced it, and then did a few more searches. That's when I landed on issue [siderolabs/talos#9400](https://github.com/siderolabs/talos/issues/9400), where the authors suggest hardcoding the UUID in the META partition using `talosctl meta write`.

## Building a tool to write META

The fix I needed was tag `0x0f`: UUIDOverride. If this tag is present in META, Talos uses its value instead of reading from SMBIOS. This is the escape hatch for a machine whose UUID Talos cannot read, whatever the reason.

The problem, however, was that `talosctl meta write` requires a running Talos API and my Talos is not booting. I needed to write directly to the raw block device from rescue mode. So I dug through the source further, found out how the cli writes the metadata to disk.

I had OpenCode write a small Go tool ([`talos-meta`](https://github.com/hoshsadiq/talos-meta)) that manages the META binary format and can read/write tags directly to `/dev/sda4`. I ended up having to vendor the original Talos META packages for this because a simple raw write would fail without the correct checksum and magic2 trailer. The format uses big-endian encoding for tags and lengths, and each block must end with a specific terminator (0 tag and 0 length) followed by the SHA-256 checksum and the `magic2` trailer.

Now I can read and write the tags from the META partition.

```bash {filename="rescue mode"}
$ /tmp/talos-meta read --device /dev/sda4
2026/06/21 22:50:33 META: loading from /dev/sda4
0x06  upgrade                           A
0x09  stateencryptionconfig             {"EncryptionProvider":"luks2","EncryptionKeys":[{"KeyStatic":null,"KeyNodeID":{},"KeyKMS":null,"KeySlot":0,"KeyTPM":null}],"EncryptionCipher":"","EncryptionKeySize":0,"EncryptionBlockSize":0,"EncryptionPerfOptions":null}
2026/06/21 22:50:33 META: loaded 2 keys
```

It had two existing tags:
- `0x06`: Upgrade slot indicator (`A` or `B`)
- `0x09`: StateEncryptionConfig (confirming `KeyNodeID` is the encryption method)

Then from rescue mode we can write the tag directly:

```bash {filename="rescue mode excerpt"}
$ /tmp/talos-meta write --device /dev/sda4 --tag UUIDOverride --value "bc94b891-0df9-47d8-a7d3-c7c199d66b0e"
2026/06/21 22:50:57 META: loading from /dev/sda4
2026/06/21 22:50:57 META: loaded 2 keys
2026/06/21 22:50:57 META: saving to /dev/sda4
2026/06/21 22:50:57 META: saved 3 keys
2026/06/21 22:50:57 META: loading from /dev/sda4
written: 0x0f (uuidoverride) = "bc94b891-0df9-47d8-a7d3-c7c199d66b0e"
verified: read-back matches
2026/06/21 22:50:57 META: loaded 3 keys
```

This writes the UUID override to both the primary and backup regions of the META partition. In my case I just used the UUID from system uuid.

## Finally???

After writing the UUID override, I rebooted. But it's still not working. I then booted into the vKVM guest with the same passed-through disk, and this time the API and health checks came up. The guest successfully proved the META override was usable to decrypt the partition there and it was finally able to decrypt the STATE partition and load the configuration.

```bash
$ talosctl get systeminformation --nodes "<node-ip>" -o yaml
node: "123.123.123.123"
metadata:
    namespace: hardware
    type: SystemInformations.hardware.talos.dev
    id: systeminformation
    version: 1
    owner: hardware.SystemInfoController
    phase: running
    created: 2026-06-21T21:08:51Z
    updated: 2026-06-21T21:08:51Z
spec:
    manufacturer: QEMU
    productName: Standard PC (Q35 + ICH9, 2009)
    version: pc-q35-7.2
    uuid: "bc94b891-0df9-47d8-a7d3-c7c199d66b0e"
    wakeUpType: Power Switch
```

The cluster was healthy:

```bash
$ talosctl health --nodes "123.123.123.123" --wait-timeout 5m
discovered nodes: ["123.123.123.123"]
waiting for etcd to be healthy: OK
waiting for etcd members to be consistent across nodes: OK
waiting for apid to be ready: OK
waiting for kubelet to be healthy: OK
waiting for all k8s nodes to report ready: OK
...
```

And kubectl confirmed the node was Ready:

```bash
$ kubectl get nodes --server "https://123.123.123.123:6443"
NAME                STATUS   ROLES           AGE     VERSION
talos-homelab-cp0   Ready    control-plane   2d21h   v1.33.11
```

Now I need to understand why it refuses to boot outside of vKVM.

## The bare metal complication

The vKVM success was encouraging but not the end. When I exited vKVM and let the server boot on bare metal, it had a different problem: networking did not come up.

To understand why, I ended up getting physical KVM access so I was able to observe what was going on when the system boots up in its natural habitat (directly on the hardware). This doubly confirmed out UUID fix, but the networking is not coming up at all. This explains why Talos doesn't boot. Within the TUI, you can confirm the network by pressing F3.

![Talos not network](/talos-bare-metal-no-gateway.png)

I didn't capture it, but on bare metal, during boot, it was trying to start networking with `eth0`, but this did not exist, so the address, route and gateway never applied. The gateway itself was valid. The config just attached to an interface that was not there. The system could boot and decrypt the disks, but it could not reach the network to join the cluster or even respond to `talosctl`.

Hetzner's vKVM network-boots a small Debian rescue system, which then starts your installed OS as a QEMU/KVM guest with the physical disks passed straight through. After checking the cmdline, I looked at the cmdline, and saw `net.ifnames=0` was there, which turns off predictable interface names.

```text {filename="/proc/cmdline"}
initrd=initrd.img nfsdir=[2a01:4ff:ff00::b007:1]:/nfs RFILE=rescue-amd64-bookworm-v012.ext2 VKVM=vkvm64-bookworm-overlay.ext2 HASH=$6$<redacted> CUSTOM_PASSWORD=true net.ifnames=0 vga=0x317 lang=de quiet IP6=2a01:4f8:120:84e6:: IP6MASK=64 IP6GW=fe80::1 keyboard=us BOOTIF=01-90:1b:0e:e0:d7:06
```

Bare metal Talos is different. Its GRUB entry has no such flag:

```bash
$ grep -c "net.ifnames" /mnt/boot/grub/grub.cfg
0
```

Somewhere somehow, something happened that caused Talos to configure itself with the old unpredictable interface. My guess is the vKVM qemu instance uses virtio NIC, and caused it to write to the network config, but the real hardware is an Intel NIC, which may have caused it to only register on the predictable device path. Regardless, I hardcoded the interface to the correct one via the TUI, rebooted, and voila we're back.

Lastly, I changed the interface in `config.yaml` from `eth0` to `enp0s31f6`, committed that.

## Lessons

**Take a backup before writing to META.** I did not `dd` the META partition before modifying it. The tool read the existing tags and wrote them back with the new one, and the read-back confirmed all three were intact, so nothing was lost. But I should have taken a copy first. `dd if=/dev/sda4 of=/tmp/meta-backup.bin` would have taken seconds and given me a way back if the write had gone wrong. Note that `/tmp` in the rescue RAM is volatile; any backup stored there must be copied to a remote host before rebooting, or it will be lost.

**Talos's UUID override exists for a reason.** The `0x0f` META tag is there for machines whose UUID Talos cannot read. It works regardless of why the UUID is wrong, so it's worth always hardcoding it.

**Get the right console for bare metal.** vKVM runs your installed OS as a QEMU guest, so it only shows you the guest, not the physical boot. To watch a physical boot you need the real KVM console. I spent time looking at the wrong screen because of this.

**The `reclaim-the-stack/talos-manager` project exists.** It's a tool that handles exactly this Hetzner UUID problem (among other Talos management tasks). Worth looking at if you're running Talos on Hetzner at any scale.

## References

- [Talos disk encryption docs](https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/storage-and-disk-management/disk-encryption): `KeyNodeID` encryption method explanation
- [UUID entropy check source](https://github.com/siderolabs/talos/blob/main/internal/pkg/encryption/keys/nodeid.go): the character frequency check that rejects all-zero UUIDs
- [Commit adding UUID override (META key 0x0f)](https://github.com/siderolabs/talos/commit/6eade3d5e)
- [Talos architecture docs](https://docs.siderolabs.com/talos/v1.13/learn-more/architecture): META partition layout and purpose
- [META partition constants](https://github.com/siderolabs/talos/blob/main/pkg/machinery/meta/constants.go): tag definitions (0x06 Upgrade, 0x09 StateEncryptionConfig, 0x0f UUIDOverride)
- [Metal network configuration](https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/bare-metal-platforms/metal-network-configuration): `talosctl meta write` for network config on bare metal
- [reclaim-the-stack/talos-manager](https://github.com/reclaim-the-stack/talos-manager): documents the Hetzner UUID scenario and other Talos management tasks

