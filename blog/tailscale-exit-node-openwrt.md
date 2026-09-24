# Tailscale exit nodes on GL.iNet routers
I just needed to route all LAN client traffic on my router through the Tailscale exit node. My GL.iNet router had other ideas.

<!--more-->

I work remotely, and travel during the weekends. To keep my tech consistent everywhere I go, I always carry around a smaller router (the GL.iNet MT3000) running openWRT.

This router has saved in more ways though, as it has a built-in VPN clients for various VPN provider including my preferred provider, Mullvad. In some countries some services are blocked, sometimes by the client or sometimes by the country, so VPN becomes essential.

Using Mullvad, however convenient, does sometimes introduce issues. For example:

- YouTube refuses to let you watch unless you sign in (yes, I'm aware of Invidious).
- You get vastly more captcha solving.

There are probably other issues that I am forgetting, but, to address this, I asked a family member to help me out. Run a Pi in their home, so I can run Tailscale and use it as an exit node.

So, once the Pi was set up, and booted, I was given SSH access by said family member, and from there installing Tailscale was very simple, and once configured to act as an exit node, it was as easy as updating my laptop's Tailscale config to use said exit node.

And the exit node worked perfectly too! As my family member has a Gigabit Fibre broadband, I was able to max out my measly 50mbps. So next, let's configure it on the router so that all LAN devices are able to use the exit node automatically. Once configured... nothing... There was no network.

This post is the investigation to see why this was not working. It took several hours, multiple wrong hypotheses, and a fix that turned out to be one iptables rule.

## The setup

```
LAN clients (192.168.8.0/24)
    |
    v  (br-lan)
GL.iNet MT3000 router (192.168.8.1 / Tailscale: 100.84.202.35)
    |
    v  (apclix0 -- repeater uplink to 192.168.1.0/24)
Upstream router (192.168.1.1) --> Internet (public IP: 74.247.222.60)

Tailscale exit node:
    Raspberry Pi (Tailscale: 100.76.113.4 / public IP: 104.147.211.28)
```

- Router: GL.iNet MT3000, OpenWRT 21.02-SNAPSHOT, Tailscale 1.80.3
- Exit node: Raspberry Pi, advertising itself as an exit node on the same tailnet
- Router is configured as a repeater (WAN side is 192.168.1.0/24, LAN side is 192.168.8.0/24)
- All LAN clients use 192.168.8.1 as their default gateway

The goal: when I enable the Tailscale exit node on the router, every device on 192.168.8.0/24 should egress through the Pi's internet connection instead of the router's WAN.

## What the docs say

At this point I have almost exclusively used the GL.iNet UI. Later, I'll realise that this was probably a mistake, and I should have followed OpenWRT's instructions.

## What Tailscale support said

Having made sure I followed all the configuration necessary, and validating the information, I was stumped. I contact Tailscale support (super impressed that they have support for free customers by the way!!). They said the router can only use the exit node for its own traffic (I already knew this). For LAN devices to route through the exit node, I would need to configure the router as a subnet router (advertising 192.168.8.0/24) and disable SNAT. I was expecting both of these configurations to be done by default when enabling Tailscale's exit node in the router's UI. The admin panel certainly showed the routes were advertised.

## Checking the baseline

With Tailscale enabled on the router (but no exit node set), the routing rules looked like this:

```shell
root@GL-MT3000:~# ip rule show
# ...
5270:  from all lookup 52
6000:  from all fwmark 0x8000/0xf000 lookup main
# ...
9920:  from all iif br-lan blackhole
# ...
```

Rule 5270 consults table 52 for all traffic. This is the route table created and managed by Tailscale.
Rule 6000 is GL.iNet's default policy, which routes marked traffic via the main table.
Rule 9920 is a kill-switch: if nothing matched above, blackhole LAN traffic (prevents leaks when a VPN tunnel is configured).

Notice rule 5270 has no `fwmark` condition and it is `from all`. This means table 52 is consulted regardless of what GL.iNet's rules do to the packet mark. If table 52 has a matching route, it wins before rule 6000 ever fires.

Let's see what in the route table 52.

```bash
root@GL-MT3000:~# ip route show table 52
100.73.38.121 dev tailscale0
100.76.113.4 dev tailscale0
100.100.100.100 dev tailscale0
100.124.213.124 dev tailscale0
100.126.184.59 dev tailscale0
```

Without an exit node, it only contains routes to specific Tailscale peers, no default route. Internet traffic from LAN clients follows the normal path: the GL.iNet VPN policy routing marks packets with `0x8000/0xf000` in the mangle table, which ip rule priority 6000 matches to send through the main routing table, hitting `default via 192.168.1.1 dev apclix0`: the normal WAN egress.

{{% conversation %}}
{{% say "El Pato" "character" %}}
Mangle table? That's not a real word.
{{% /say %}}
{{% say "Hosh" "author" %}}
The user iptablesnoob [puts this quite well](https://serverfault.com/a/774964):

> The mangle table allows to modify some special entries in the header of packets. (such: Type of Service, Time To Live ) (it also allows to set special marks and security context marks)
{{% /say %}}
{{% /conversation %}}

## Enabling the exit node

Let's see what changes in the Tailscale route table when I enabled the exit node.

```
root@GL-MT3000:~# tailscale debug prefs
{
  "ControlURL": "https://controlplane.tailscale.com",
  "RouteAll": true,
  "ExitNodeID": "npbtV5ZPiC21CNTRL",
  "ExitNodeIP": "",
  "LoggedOut": false,
  "AdvertiseRoutes": [
    "192.168.1.0/24",
    "192.168.8.0/24"
  ],
  "NoSNAT": false
// ...
}
```

The `AdvertiseRoutes` seems correct, though I wasn't expected the WAN subnet to be advertised. But `NoSNAT` seems wrong.

## Hypothesis 1: SNAT issues

{{% conversation %}}
{{% say "El Pato" "character" %}}
SNAT is surely the issue if the support engineer mentioned it?
{{% /say %}}
{{% say "Hosh" "author" %}}
Indeed! I thought I had set this already.
{{% /say %}}
{{% /conversation %}}

So we've enabled exit node within the UI, but there is no way to set the SNAT options. We'll have to do it via the CLI.

```
root@GL-MT3000:~# tailscale set --snat-subnet-routes=false
root@GL-MT3000:~# tailscale debug prefs | grep -A3 -e AdvertiseRoutes -e NoSNAT
        "AdvertiseRoutes": [
                "192.168.1.0/24",
                "192.168.8.0/24"
        ],
--
        "NoSNAT": true,
        "NoStatefulFiltering": true,
        "NetfilterMode": 2,
        "AutoUpdate": {
```

Okay! Perfect. Let's test it.

```

$ curl --connect-timeout 1 https://ifconfig.me
curl: (28) Failed to connect to ifconfig.me port 443 after 1006 ms: Timeout was reached
```

Still nothing.


## Hypothesis 2: routing or firewall is wrong

Now let's look at table 52 again with exit node enabled. We're seeing a new default route, and it's skipping local traffic:

```bash
root@GL-MT3000:~p# ip route show table 52
default dev tailscale0          # <-- new
100.73.38.121 dev tailscale0
100.76.113.4 dev tailscale0
...
throw 127.0.0.0/8
throw 192.168.1.0/24
throw 192.168.8.0/23
```

That `default dev tailscale0` means all internet-bound traffic now matches in table 52, at ip rule priority 5270, before GL.iNet's next routing rule. Both the router's own traffic and forwarded LAN traffic should go into the Tailscale tunnel.

I verified with `ip route get`, simulating a LAN client packet with the GL.iNet mangle mark:

```bash
root@GL-MT3000:~# ip route get 8.8.8.8 from 192.168.8.209 iif br-lan mark 0x8000
8.8.8.8 from 192.168.8.209 dev tailscale0 table 52 mark 0x8000
    cache iif br-lan
```

Great! The kernel confirms this: a packet from a LAN client heading to 8.8.8.8, even with the mangle mark applied, would be routed to `tailscale0` via table 52. The routing layer was correct.

I also checked the firewall. Tailscale's own chains in the filter table:

```
root@GL-MT3000:~# iptables -t filter -S ts-forward
-N ts-forward
-A ts-forward -i tailscale0 -j MARK --set-xmark 0x40000/0xff0000
-A ts-forward -m mark --mark 0x40000/0xff0000 -j ACCEPT
-A ts-forward -s 100.64.0.0/10 -o tailscale0 -j DROP
-A ts-forward -o tailscale0 -j ACCEPT
```

Traffic forwarded to `tailscale0` is explicitly accepted so no firewall block.

This also seems to not be an issue. The kernel routes LAN packets to tailscale0. The firewall allows them through.

I confirmed this with iptables counters by zeroing the counters, 464 packets were forwarded from br-lan to tailscale0 in short period. Zero packets went to apclix0 (the WAN interface).

```
root@GL-MT3000:~# iptables -Z INPUT
root@GL-MT3000:~# iptables -Z OUTPUT
root@GL-MT3000:~# iptables -Z FORWARD
root@GL-MT3000:~# iptables -I INPUT 1 -i tailscale0 -j ACCEPT -m comment --comment "DEBUG_TS_IN"
root@GL-MT3000:~# iptables -I OUTPUT 1 -o tailscale0 -j ACCEPT -m comment --comment "DEBUG_TS_OUT"
root@GL-MT3000:~# iptables -I FORWARD 1 -i br-lan -o tailscale0 -j ACCEPT -m comment --comment "DEBUG_TS_FWD"
root@GL-MT3000:~# iptables -I FORWARD 2 -i br-lan -o apclix0 -j ACCEPT -m comment --comment "DEBUG_WAN_FWD"
root@GL-MT3000:~# conntrack -F
conntrack v1.4.6 (conntrack-tools): connection tracking table has been emptied.
root@GL-MT3000:~# iptables -L INPUT -n -v | grep tailscale0 -C3
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    76 ACCEPT     all  --  tailscale0 *       0.0.0.0/0            0.0.0.0/0            /* DEBUG_TS_IN */
  203 18127 ts-input   all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     icmp --  br-lan *       0.0.0.0/0            0.0.0.0/0            icmptype 8 limit: avg 25/sec burst 5
    0     0 LOG        icmp --  br-lan *       0.0.0.0/0            0.0.0.0/0            icmptype 8 limit: avg 1/sec burst 5 LOG flags 0 level 4
--
    0     0 zone_wan_input  all  --  eth0   *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
   72  2016 zone_wan_input  all  --  apclix0 *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_guest_input  all  --  br-guest *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_tailscale0_input  all  --  tailscale0 *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
root@GL-MT3000:~# iptables -L OUTPUT -n -v | grep tailscale0 -C3
Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    7   457 ACCEPT     all  --  *      tailscale0  0.0.0.0/0            0.0.0.0/0            /* DEBUG_TS_OUT */
    1   180 ACCEPT     all  --  *      lo      0.0.0.0/0            0.0.0.0/0            /* !fw3 */
  595 78591 output_rule  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* !fw3: Custom output rule chain */
  525 70342 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED /* !fw3 */
--
    0     0 zone_wan_output  all  --  *      eth0    0.0.0.0/0            0.0.0.0/0            /* !fw3 */
   69  8197 zone_wan_output  all  --  *      apclix0  0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_guest_output  all  --  *      br-guest  0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_tailscale0_output  all  --  *      tailscale0  0.0.0.0/0            0.0.0.0/0            /* !fw3 */
root@GL-MT3000:~# iptables -L FORWARD -n -v | grep tailscale0 -C3
Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
  371 23838 ACCEPT     all  --  br-lan tailscale0  0.0.0.0/0            0.0.0.0/0            /* DEBUG_TS_FWD */
    0     0 ACCEPT     all  --  br-lan apclix0  0.0.0.0/0            0.0.0.0/0            /* DEBUG_WAN_FWD */
  117  9348 ts-forward  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set GL_MAC_BLOCK src
--
    0     0 zone_wan_forward  all  --  eth0   *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_wan_forward  all  --  apclix0 *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_guest_forward  all  --  br-guest *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_tailscale0_forward  all  --  tailscale0 *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 reject     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
root@GL-MT3000:~# cat /proc/net/nf_conntrack | grep "192.168.8" | grep -v "dst=192.168.8"
root@GL-MT3000:~# iptables -D FORWARD -i br-lan -o tailscale0 -j ACCEPT -m comment --comment "DEBUG_TS_FWD"
root@GL-MT3000:~# iptables -D FORWARD -i br-lan -o apclix0 -j ACCEPT -m comment --comment "DEBUG_WAN_FWD"
root@GL-MT3000:~# iptables -D INPUT -i tailscale0 -j ACCEPT -m comment --comment "DEBUG_TS_IN"
root@GL-MT3000:~# iptables -D OUTPUT -o tailscale0 -j ACCEPT -m comment --comment "DEBUG_TS_OUT"
```

{{% conversation %}}
{{% say "El Pato" "character" %}}
So 464 packets went in. Nothing came back.
{{% /say %}}
{{% /conversation %}}

Let's validate using tcpdump.

```
$ ping -c4 1.1.1.1
PING 1.1.1.1 (1.1.1.1): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2

--- 1.1.1.1 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
```

```
root@GL-MT3000:~# tcpdump -i tailscale0 -n 'host 192.168.8.209 and host 1.1.1.1'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tailscale0, link-type RAW (Raw IP), capture size 262144 bytes
20:45:13.787471 IP 192.168.8.209 > 1.1.1.1: ICMP echo request, id 36938, seq 0, length 64
20:45:14.796386 IP 192.168.8.209 > 1.1.1.1: ICMP echo request, id 36938, seq 1, length 64
20:45:15.844514 IP 192.168.8.209 > 1.1.1.1: ICMP echo request, id 36938, seq 2, length 64
20:45:16.798100 IP 192.168.8.209 > 1.1.1.1: ICMP echo request, id 36938, seq 3, length 64
```

No echo replies. But this is interesting. The source IP is my LAN IP. This is certainly not expected. If this was NATed, I'd expected it the source IP to be the Tailscale IP.

## The packets go in but nothing comes back

Okay, let's check why SNAT is not working.

```
root@GL-MT3000:~# uci show firewall | grep "=zone" | grep tailscale
firewall.tailscale0=zone

root@GL-MT3000:~# uci show firewall.tailscale0=zone
firewall.tailscale0=zone
firewall.tailscale0.name='tailscale0'
firewall.tailscale0.input='ACCEPT'
firewall.tailscale0.mtu_fix='1'
firewall.tailscale0.device='tailscale0'
firewall.tailscale0.output='ACCEPT'
firewall.tailscale0.forward='REJECT'
```

Oh, this certainly would explain it. I would expect it to be `masq='1'`.

```
root@GL-MT3000:~# iptables -t nat -S | grep -i masq
-A zone_wan_postrouting -m comment --comment "!fw3" -j MASQUERADE
```

And there is no `POSTROUTING` rule `MASQUERADE` for `tailscale0`.

Let's force a minimal rule for this.

```
root@GL-MT3000:~# iptables -t nat -A POSTROUTING -o tailscale0 -s 192.168.8.0/24 -j MASQUERADE
```

And let's test it
```
$ curl https://ifconfig.me
45.159.89.9
```

Hallelujah! It's working. Now we need to make this permanent. 

## Making the fix permanent

{{% conversation %}}
{{% say "El Pato" "character" %}}
Did you check the docs for OpenWRT to see why this is?
{{% /say %}}
{{% say "Hosh" "author" %}}
Uuuh...
{{% /say %}}
{{% /conversation %}}

So at this point, I started looking at the [OpenWRT docs for Tailscale](https://openwrt.org/docs/guide-user/services/vpn/tailscale/start?s[]=ad&s[]=commit), and started the setup from scratch.

New Firewall rule... 

There is it!

> Masquerading: **on**

When using the GL.iNet UI to enable traffic through an exit node, it seems it does not set the masquerading option for the new tailscale network. This needs to be done manually in LuCI (available at System > Advanced Settings). The rule is already added, but masquerading is not enabled. Enabling that fixes the traffic in the same way.

## Lesson learned

This seems like something I should have caught early on by simply checking the docs. Well, sometimes you just don't need a manual.

