From my prereading, I understood that the steps of segmenting an infrastructure involve first giving a "physical room" at Layer 2, using VLAN filtering under a Linux bridge. The second main step involves subnetting, assigning addressing within these separation domains at Layer 3, which gives them an identity. Each of these complements the other and they work together. Then finally, rules are assigned across these zones via firewall zones and traffic rules, at the layer above.

**VLAN filtering stage**

The first step I tried was to create all 5 "rooms" within my `br-lan`. This failed terribly, since I hadn't understood it well enough, and I confirmed the changes within the 90 second window (don't even know why I did that), which meant the rollback never triggered and I locked myself out.

After some learning, I decided it was a good idea to first move my current LAN into a temporary VLAN 1. This failed again, but I quickly found that it was because my physical ports had been moved into the new VLAN 1 while the `lan` interface itself remained bound to the bare `br-lan` device, hence a separation between where the traffic was and where the router was listening.

Finally, this worked by creating the temporary VLAN 1 and repointing the `lan` interface to the `br-lan.1` device.

**Creating all 5 VLANs**

First I tested `br-lan.10` by connecting my desktop directly via cable. Everything worked well, and DHCP on the router gave it `192.168.10.122`. Secondly, I tested that ping packets could not reach another subnet, such as my laptop on the temporary old network at `192.168.123.72`.

I then created the rest, VLANs 20/30/40/50, the same way, and completed the firewalling.

**Testing the VLANs**

First I moved my PC to VLAN 10 and configured the switch port 4 as untagged. This was successful, since I cleared out my IP on the PC and got a new lease of `192.168.10.122`. Since the admin zone forwards to all other zones, I was able to reach my switch and other devices, which was expected.

What narrowed it down was comparing a forced query against the default one. `dig @192.168.123.35 google.com` worked, so the Pi was resolving fine and the path to it was open. But plain `dig google.com` failed, and that goes to whatever the client was handed by DHCP, which was the router at `192.168.30.1`. So the Pi wasn't the problem at all, the router was refusing the query. The `svc` zone had input reject with no rule permitting port 53 to the router itself.

The fix is either a forwarding rule to the Pi on port 53, or an input rule allowing port 53 to the router. I will probably go for the former, since per-device query logging in Pi-hole is valuable information.

**Moving the Pi**

This part was the most challenging, since I rely heavily on my DNS server. I started by changing the router's DNS forwarding to an upstream at `1.1.1.1`. Since I have input rejected on these zones, this was more of a backup. The main temporary change I needed was DHCP option 6 per zone, pointing at AdGuard so clients had a working resolver while the Pi was offline.

I then loosened the UFW rules on the Pi via SSH to make sure I didn't lose access, moved the Pi into VLAN 30, and tightened them back up. Lastly I reset option 6 across the interfaces to point at the Pi's new address.

Although this should have been one of the more annoying migrations, it went fairly well, since I did most steps in the right order, especially the UFW part which I usually forget.

To finish, I moved the switch to VLAN 20 and did a final check that no devices were left in VLAN 1, then retired the `192.168.123.0/24` subnet.