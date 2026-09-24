# Hetzner's odd default Firewall rule

Pods in my k3s cluster on a Hetzner dedicated server were failing to reach external services. Spoiler: Hetzner's default Firewall rules (probably?) don't make sense.

<!--more-->

It took a few hours of investigation to finally understand this issue.

## The setup

I am running a single-node k3s cluster on a Hetzner dedicated server.

```text
k3s single-node cluster (Hetzner dedicated, Rocky Linux 9.7)
  IP: 203.0.113.50
  Interface: enp0s31f6
  |
  +-- flannel.1 (VXLAN)
  +-- cni0 (bridge)
  |     +-- veth* (pod interfaces)
  +-- tailscale0 (MTU 1280)

Pod CIDR: 10.42.0.0/16
Service CIDR: 10.43.0.0/16
Flannel backend: VXLAN
iptables: v1.8.10 (nf_tables backend)
```

Some details about the setup:
- I'm using Rocky Linux 9.7
- Running k3s v1.34.4
- Tailscale is running on the host for remote access
- I have not enabled host-level firewalls such as firewalld

I was trying to set up ArgoCD on this k3s installation, but for some reason, the repo-server timed out while adding a git repository with `context deadline exceeded (Client.Timeout exceeded while awaiting headers)`. Very strange as, anecdotally, I am able to connect to GitHub normally within the cluster from some pods.

## Isolating the problem

First step: determine if this is an ArgoCD problem or a network problem. I created a debug pod using the netshoot image to run some tests. This pod will be used regularly down the line so let's just keep it running and exec into it.

```bash {filename="pod-network test"}
$ kubectl run debug-net --image=nicolaka/netshoot --restart=Never -- sleep infinity
pod/debug-net created
```

I already confirmed I could clone once, so to see if this was a transient issue, I ran a simple loop to test connectivity to GitHub from within the pod network.
```bash
$ seq 1 10 | kubectl exec -i debug-net -- \
    xargs -I{} \
      curl -sS -o /dev/null -w "attempt {}: %{http_code} %{time_connect}\n" --connect-timeout 5 https://github.com

attempt 1: 200 0.006835
attempt 2: 000 0.000000
attempt 3: 000 0.000000
attempt 4: 200 0.006701
attempt 5: 200 0.006855
attempt 6: 200 0.006965
attempt 7: 000 0.000000
attempt 8: 200 0.006903
attempt 9: 200 0.006652
attempt 10: 200 0.006639
```

Notice we have 3 items where the http code is `000`, and the connect time is 0. That means 3 out of 10 failures. Now let's try the same test from a host-network pod:

```bash {filename="host-network test"}
$ seq 1 10 | kubectl run debug-host -i --rm --image=nicolaka/netshoot --restart=Never \
  --overrides='{"spec":{"hostNetwork":true}}' -- \
    xargs -I{} \
      curl -sS -o /dev/null -w "attempt {}: %{http_code} %{time_connect}\n" --connect-timeout 5 https://github.com
...
attempt 1: 200 0.005891
attempt 2: 200 0.006012
attempt 3: 200 0.005934
...
attempt 10: 200 0.005998
pod "debug-host" deleted from default namespace
```

10 out of 10 success. So this is not GitHub being flaky. Something about the pod network path is different.

## Ruling out the usual suspects

### Hypothesis: MTU fragmentation

This was an unnecessary wild goose chase I got into due to AI. It made no sense, but I figured I'd try and eliminate this. The hypothesis was that VXLAN adds overhead to every packet, so our 1500-byte MTU wasn't leaving enough room (i.e. packets were getting too big). The breakdown is: outer IP (20) + UDP (8) + VXLAN header (8) + inner Ethernet (14) = 50 bytes. Since the host MTU is 1500, the inner MTU must be 1450.

The Maximum Segment Size (MSS) advertised in SYN packets should be inner MTU - 20 (IP header) - 20 (TCP header) = 1410.

I checked the pod MTU:

```bash
$ kubectl exec debug-net -- cat /sys/class/net/eth0/mtu
1450
```

Okay, so that's correct. If MTU was the issue, I'd expect successful connections that break during data transfer, not connections that fail to establish at all. Meaning MTU is unlikely to be the cause. I confirmed later in tcpdump that both successful and failed SYNs showed `mss 1410`, which matches the expected 1450 inner MTU.

### Hypothesis: conntrack exhaustion

Conntrack is the kernel's connection tracking table. Every connection gets an entry. If the table fills up, new connections are dropped.

```bash
$ ssh hetzner-server cat /proc/sys/net/netfilter/nf_conntrack_count
823
$ ssh hetzner-server cat /proc/sys/net/netfilter/nf_conntrack_max
262144
$ ssh hetzner-server conntrack -S
cpu=0  found=0 invalid=22283 insert=0 insert_failed=212 drop=212 early_drop=0 ...
```

Only 823 out of 262,144 entries used. The stats showed 212 `insert_failed` and 212 `drop` over 12 million forwarded packets. Conntrack is unlikely to be the issue here.

## tcpdump on the physical interface

This is where the investigation got interesting. I ran tcpdump on the host on `enp0s31f6`, the physical NIC. Capturing here is key because it shows packets after they have passed through the entire kernel networking stack, that is iptables, NAT, and routing. I.e. if a packet appears here, it left the host.

I generated some pod traffic to github.com (`140.82.121.3`) from the same `debug-net` pod and watched the physical interface on the host.
```text {filename="tcpdump -i enp0s31f6 -nn host 140.82.121.3"}
10:18:59.827 203.0.113.50.23475 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:00.868 203.0.113.50.23475 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:02.916 203.0.113.50.23475 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:06.948 203.0.113.50.23475 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:07.870 203.0.113.50.13577 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:08.932 203.0.113.50.13577 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:10.980 203.0.113.50.13577 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:15.012 203.0.113.50.13577 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:15.840 203.0.113.50.59127 > 140.82.121.3.443: Flags [S], mss 1410, ...
10:19:15.845 140.82.121.3.443 > 203.0.113.50.59127: Flags [S.], mss 1436, ...
10:19:15.845 203.0.113.50.59127 > 140.82.121.3.443: Flags [.], ack 1, ...
```

Okay, this is interesting, the only difference is the source port. Three connections were attempted. Two failed, one succeeded.
- **Source port 23475**: Four SYN retransmits. No SYN-ACK ever arrives.
- **Source port 13577**: Same pattern. Four retransmits. No SYN-ACK.
- **Source port 59127**: SYN goes out, SYN-ACK comes back in 5ms. Handshake completes.

I checked the conntrack table on the host to see how these connections were being tracked:

```text {filename="ssh hetzner-server conntrack -L -d 140.82.121.3"}
ipv4 2 tcp 6 52 SYN_SENT src=10.42.0.125 dst=140.82.121.3 sport=58162 dport=443
    [UNREPLIED] src=140.82.121.3 dst=203.0.113.50 sport=443 dport=13577
ipv4 2 tcp 6 44 SYN_SENT src=10.42.0.125 dst=140.82.121.3 sport=46978 dport=443
    [UNREPLIED] src=140.82.121.3 dst=203.0.113.50 sport=443 dport=23475
```

The pod's original source port (e.g. 58162) was being SNAT'd to a new port (e.g. 13577) by flannel's MASQUERADE rule. The `[UNREPLIED]` flag means the SYN went out, but nothing came back.

My first thought was that maybe GitHub has some DDoS protection which was dropping some SYNs. It would be very weird, but I've seen weirder stuff in my life. Anyway, this happened with various other destinations I tested, so it couldn't have been GitHub.

## More dead ends

### Theory: bridge-nf-call-iptables

`bridge-nf-call-iptables` is a sysctl that makes bridged frames (Layer 2 traffic on `cni0`) pass through iptables rules. Kubernetes requires this so that iptables DNAT rules can redirect pod DNS queries to the CoreDNS ClusterIP.

I suspected that with this enabled, packets from pods might be traversing the iptables `FORWARD` chain twice, once as a bridged frame and once as a routed packet. Maybe something in the second pass was dropping them. Let's find out by using `nsenter`:

```bash
$ kubectl run --rm -it debug-brtest --image=alpine --restart=Never \
    --overrides='{"spec":{"hostNetwork":true,"hostPID":true,
      "containers":[{"name":"debug-brtest","image":"alpine",
        "securityContext":{"privileged":true},
        "command":["sh","-c",
          "nsenter -t 1 -m -u -n -i -- sysctl -w net.bridge.bridge-nf-call-iptables=0"]}]}}'
net.bridge.bridge-nf-call-iptables = 0
pod "debug-brtest" deleted from default namespace
```

Then I retested from the debug-net pod:

```bash
$ kubectl exec debug-net -- nslookup github.com
;; communications error to 10.43.0.10#53: timed out
;; communications error to 10.43.0.10#53: timed out
;; communications error to 10.43.0.10#53: timed out
;; no servers could be reached
```

This change broke DNS is completely.

CoreDNS runs as a pod behind a ClusterIP service (10.43.0.10). Pods send DNS queries to that IP. iptables DNAT rewrites the destination to the actual CoreDNS pod IP. But DNS queries from pods travel over the `cni0` bridge. Without `bridge-nf-call-iptables`, those bridged frames skip iptables entirely, and the DNAT never happens.

Reverted immediately:

```bash
$ kubectl run --rm -it debug-brrevert --image=alpine --restart=Never \
    --overrides='{"spec":{"hostNetwork":true,"hostPID":true,
      "containers":[{"name":"debug-brrevert","image":"alpine",
        "securityContext":{"privileged":true},
        "command":["sh","-c",
          "nsenter -t 1 -m -u -n -i -- sysctl -w net.bridge.bridge-nf-call-iptables=1"]}]}}'
net.bridge.bridge-nf-call-iptables = 1
pod "debug-brrevert" deleted from default namespace
```

DNS came back. Not the cause of the egress failures, and disabling it makes things worse.

### Theory: Tailscale routing

Tailscale adds policy routing rules with fwmark. I checked what the routing setup looked like on the host:

```bash {filename="on the host"}
$ ip rule list
0:      from all lookup local
5210:   from all fwmark 0x80000/0xff0000 lookup main
5230:   from all fwmark 0x80000/0xff0000 lookup default
5250:   from all fwmark 0x80000/0xff0000 unreachable
5270:   from all lookup 52
32766:  from all lookup main
32767:  from all lookup default
```

Rule 5270 sends all traffic through table 52. What is in table 52?

```bash {filename="on the host"}
$ ip route show table 52
100.73.38.121 dev tailscale0
100.76.113.4 dev tailscale0
100.84.202.35 dev tailscale0
100.100.100.100 dev tailscale0
100.126.184.59 dev tailscale0
$ ip route show table main
default via 203.0.113.1 dev enp0s31f6 proto static metric 100
10.42.0.0/24 dev cni0 proto kernel scope link src 10.42.0.1
203.0.113.1 dev enp0s31f6 proto static scope link metric 100
```

Table 52 only contains routes to Tailscale peers (100.x.x.x addresses). For anything else, such as github.com, the lookup finds no match and falls through to the main table, which has the correct default route via 203.0.113.1 on `enp0s31f6`. Tailscale is not interfering.

## Final hypothesis

I kept coming back to the tcpdump. The only variable between success and failure was the source port. Failed connections used ports 23475, 13577, and 4905. Successful ones used ports 59127, 32980, and 33364.

Then I noticed the pattern: every successful port was at or above 32767. Every failed port was below it.

Normal host-originated connections use the kernel's ephemeral port range, defined by `net.ipv4.ip_local_port_range`. This defaults to 32768-60999. This explains why host-network pods always work as they always pick from this range.

But flannel's MASQUERADE rule uses `--random-fully`, which picks from the entire 1024-65535 range. So this defines roughly half to be below 32768.

What if Hetzner's upstream network, perhaps their firewall, drops packets with source ports below the ephemeral range? It'd be a strange rule, but it's worth testing.

## Proving it

I tested this hypothesis by forcing a host-network pod to use specific source ports. Luckily curl makes this very easy (I say that, but I had no idea curl could do this! Goes to show that you can still learn in the age of AI)!

### Test 1: Host-network with random ports across full range (1024-65535)

```bash
$ seq 1 20 | kubectl run --rm -i debug-hostent --image=nicolaka/netshoot --restart=Never \
    --overrides='{"spec":{"hostNetwork":true}}' -- \
      xargs -P4 -I{} sh -c '
        port=$((1024 + (RANDOM * 32768 + RANDOM) % 64512))
        curl -4 -sS -o /dev/null --connect-timeout 1 --local-port "$port" -w "attempt {}: %{http_code} %{time_connect} port=$port>%{remote_port}\n" \
          https://github.com 2>/dev/null
      ' |
  awk '
    /^attempt/{ suffix = ($3 == "000" ? "# FAIL" : "# OK"); print $0, suffix }
    !/^attempt/{print}
  '
...
attempt 1: 000 0.000000 port=29107>-1 # FAIL
attempt 2: 000 0.000000 port=13023>-1 # FAIL
attempt 3: 000 0.000000 port=12952>-1 # FAIL
attempt 4: 000 0.000000 port=32583>-1 # FAIL
attempt 5: 000 0.000000 port=30166>-1 # FAIL
attempt 6: 000 0.000000 port=7871>-1 # FAIL
attempt 7: 200 0.006227 port=36653>443 # OK
attempt 8: 000 0.000000 port=4996>-1 # FAIL
attempt 9: 200 0.006177 port=64349>443 # OK
attempt 10: 000 0.000000 port=27149>-1 # FAIL
attempt 11: 200 0.006101 port=33977>443 # OK
attempt 12: 000 0.000000 port=28723>-1 # FAIL
attempt 13: 000 0.000000 port=17720>-1 # FAIL
attempt 14: 200 0.006135 port=39524>443 # OK
attempt 15: 000 0.000000 port=20350>-1 # FAIL
attempt 16: 200 0.006291 port=45679>443 # OK
attempt 17: 200 0.006395 port=50737>443 # OK
attempt 18: 000 0.000000 port=30045>-1 # FAIL
attempt 19: 000 0.000000 port=21561>-1 # FAIL
attempt 20: 200 0.006196 port=35555>443 # OK
pod default/debug-hostent terminated (Error)
pod "debug-hostent" deleted from default namespace
```

So 13 out of 20 failed, all successful ports were above 32767.

### Test 2: Host-network with ephemeral ports only (32768-65535)

```bash {filename="host-network pod"}
$ seq 1 20 | kubectl run --rm -i debug-hostent --image=nicolaka/netshoot --restart=Never \
    --overrides='{"spec":{"hostNetwork":true}}' -- \
      xargs -P4 -I{} sh -c '
        port=$((RANDOM + 32768))
        curl -4 -sS -o /dev/null --connect-timeout 1 --local-port "$port" -w "attempt {}: %{http_code} %{time_connect} port=$port>%{remote_port}\n" \
          https://github.com 2>/dev/null
      ' |
  awk '
    /^attempt/{ suffix = ($3 == "000" ? "# FAIL" : "# OK"); print $0, suffix }
    !/^attempt/{print}
  '
...
attempt 2: 200 0.006302 port=64072>443 # OK
attempt 1: 200 0.006250 port=34944>443 # OK
attempt 3: 200 0.006293 port=35875>443 # OK
attempt 4: 200 0.006265 port=51814>443 # OK
attempt 5: 200 0.006103 port=55567>443 # OK
attempt 8: 200 0.006113 port=37204>443 # OK
attempt 7: 200 0.006007 port=45957>443 # OK
attempt 6: 200 0.006120 port=54278>443 # OK
attempt 10: 200 0.006099 port=61168>443 # OK
attempt 9: 200 0.006177 port=55307>443 # OK
attempt 12: 200 0.006225 port=33839>443 # OK
attempt 11: 200 0.006135 port=50660>443 # OK
attempt 13: 200 0.006047 port=39438>443 # OK
attempt 15: 200 0.006071 port=51600>443 # OK
attempt 16: 200 0.006132 port=47960>443 # OK
attempt 14: 200 0.006105 port=58884>443 # OK
attempt 17: 200 0.006617 port=42085>443 # OK
attempt 18: 200 0.006116 port=62988>443 # OK
attempt 19: 200 0.006086 port=51691>443 # OK
attempt 20: 200 0.006161 port=37760>443 # OK
pod "debug-hostent" deleted from default namespace

```

All requests succeeded. It seems something somewhere in Hetzner's network was possibly dropping incoming SYN-ACK packets with source ports below less than 32767.

## Root cause

So it seems the root cause was a combination of Hetzner's upstream filtering and flannel's default MASQUERADE behavior.

Flannel uses `--random-fully` in its MASQUERADE rules.

{{% conversation %}}
{{% say "El Pato" "character" %}}
Is `--random-fully` the right default? Why does flannel use it?
{{% /say %}}
{{% say "Hosh" "author" %}}
It reduces the chance of conntrack 5-tuple collisions when many pods make connections simultaneously. The kernel's `--random` option picks from the ephemeral range, but `--random-fully` uses the full range to reduce collision. It is a reasonable default for most networks, but it is clearly incompatible with some firewall rule in Hetzner's network.
{{% /say %}}
{{% /conversation %}}

I checked the source code to see if k3s or flannel could be configured to avoid this.
- Flannel has a `--ip-masq-fully-random-disable` CLI flag that switches from `--random-fully` to `--random`.
- The flannel traffic manager interface accepts this as the [`ipMasqRandomFullyDisable` parameter](https://github.com/flannel-io/flannel/blob/f17c3c7fb58ba68e725736badeeaf4af63ec38c0/pkg/trafficmngr/trafficmngr.go#L54).
- K3s embeds flannel and [always passes `false`](https://github.com/k3s-io/k3s/blob/39ccf3e075b465b6969f5eb7cdce52acebb1aa53/pkg/agent/flannel/flannel.go#L111-L114) for this parameter.

So, there is no k3s server flag or config option to change it.

## The workaround

At this point I wanted to figure why Hetzner's network was dropping these packets so my next thought was to raise a ticket with them. But I figured before I do that, I needed a workaround. I decided to constrain MASQUERADE for pod egress to use only ephemeral ports. This meant inserting two iptables rules in the `POSTROUTING` chain before flannel's own jump rule.

```bash {filename="on the host"}
$ iptables -t nat -I POSTROUTING 1 \
    -s 10.42.0.0/16 ! -d 224.0.0.0/4 -p tcp \
    -m comment --comment "hetzner-ephemeral-masq" \
    -j MASQUERADE --to-ports 32768-60999

$ iptables -t nat -I POSTROUTING 2 \
    -s 10.42.0.0/16 ! -d 224.0.0.0/4 -p udp \
    -m comment --comment "hetzner-ephemeral-masq" \
    -j MASQUERADE --to-ports 32768-60999
```

After applying these, I get all successes again.

{{% conversation %}}
{{% say "El Pato" "character" %}}
Why not insert these rules into flannel's own chain?
{{% /say %}}
{{% say "Hosh" "author" %}}
Because flannel will delete them. K3s has embedded Flannel, and calls various Flannel functions to resync every minute or so. Any custom rules inserted there get wiped. Rules in the top-level `POSTROUTING` chain, placed before the jump to `FLANNEL-POSTRTG`, survive the resync.
{{% /say %}}
{{% /conversation %}}

## Making it permanent

The workaround obviously needs to survive reboots, and since k3s won't let me configure it correctly, I created a systemd service to apply these changes at boot.

```ini {filename="/etc/systemd/system/k3s-hetzner-masq-fix.service"}
[Unit]
Description=Restrict flannel MASQUERADE to ephemeral ports (Hetzner workaround)
After=k3s.service
Requires=k3s.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/bash -c 'until iptables -t nat -L FLANNEL-POSTRTG &>/dev/null; do sleep 2; done'
ExecStart=/usr/local/bin/k3s-hetzner-masq-fix.sh
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

```bash {filename="/usr/local/bin/k3s-hetzner-masq-fix.sh"}
#!/usr/bin/env bash
set -euo pipefail

CIDR="10.42.0.0/16"
COMMENT="hetzner-ephemeral-masq"

for proto in tcp udp; do
  iptables -t nat -D POSTROUTING \
    -s "$CIDR" ! -d 224.0.0.0/4 \
    -p "$proto" \
    -m comment --comment "$COMMENT" \
    -j MASQUERADE --to-ports 32768-60999 2>/dev/null || true

  iptables -t nat -I POSTROUTING 1 \
    -s "$CIDR" ! -d 224.0.0.0/4 \
    -p "$proto" \
    -m comment --comment "$COMMENT" \
    -j MASQUERADE --to-ports 32768-60999
done
```

## The actual fix

After deploying the workaround, I was in the middle of raising a ticket with Hetzner to understand this, but as I searched a bit further, I came across issue [k3s-io/k3s#5220](https://github.com/k3s-io/k3s/issues/5220).

It turned out that the dedicated Hetzner server has a provider-level firewall outside the machine itself. I had noticed it when I bought the server at auction, but had forgotten it existed. One of its default rules accepts TCP packets with the ACK flag set only when their destination port is in the range `32767–65535`. TCP packets with `ACK` set and destination ports below that range are dropped.

Checking for `ACK` is just a rough way to distinguish reply traffic from new inbound TCP connections, and often can make sense as a firewall setting. However, it is not equivalent to connection tracking since any packet can have `ACK` set.

The problem is that the rule assumes packets are sent and on the kernel’s usual ephemeral-port range, 32767–65535, and thus, packets will come back on the same range. This is of course a default, and individual applications can override this, as Flannel's `MASQUERADE --random-fully` does. It can choose any source port including one below 32767, same way we were able to do so with `curl`. This means the reply comes back to a port Hetzner’s firewall drops.

So I updated it to be 1024-65535. I disabled the iptables workaround and tested low local ports again:

```bash
$ seq 1 20 | kubectl run --rm -i debug-hostent --image=nicolaka/netshoot --restart=Never \
    --overrides='{"spec":{"hostNetwork":true}}' -- \
      xargs -P4 -I{} sh -c '
        port=$((1024 + RANDOM % 31745))
        curl -4 -sS -o /dev/null --connect-timeout 1 --local-port "$port" -w "attempt {}: %{http_code} %{time_connect} port=$port>%{remote_port}\n" \
          https://github.com 2>/dev/null
      ' |
  awk '
    /^attempt/{ suffix = ($3 == "000" ? "# FAIL" : "# OK"); print $0, suffix }
    !/^attempt/{print}
  '
...
attempt 2: 200 0.006344 port=14771>443 # OK
attempt 1: 200 0.006288 port=6139>443 # OK
attempt 4: 200 0.006352 port=24501>443 # OK
attempt 3: 200 0.006274 port=12774>443 # OK
attempt 7: 200 0.006076 port=3857>443 # OK
attempt 5: 200 0.006291 port=29052>443 # OK
attempt 6: 200 0.006066 port=1610>443 # OK
attempt 8: 200 0.007296 port=9568>443 # OK
attempt 9: 200 0.006178 port=15316>443 # OK
attempt 10: 200 0.006077 port=20026>443 # OK
attempt 11: 200 0.005950 port=27545>443 # OK
attempt 12: 200 0.006437 port=31795>443 # OK
attempt 15: 200 0.005950 port=16112>443 # OK
attempt 14: 200 0.005901 port=31181>443 # OK
attempt 13: 200 0.006270 port=15968>443 # OK
attempt 16: 200 0.006159 port=18248>443 # OK
attempt 17: 200 0.006543 port=11026>443 # OK
attempt 19: 200 0.005981 port=29464>443 # OK
attempt 18: 200 0.006164 port=32431>443 # OK
attempt 20: 200 0.006216 port=1130>443 # OK
pod "debug-hostent" deleted from default namespace

```

All 20 succeeded. The ports that were being blackholed now work fine.

It's a strange rule to add as a default, but at least there's a fix.

I do wonder if I had given Claude full responsibility to figure this out, if it would have found the issue sooner.

