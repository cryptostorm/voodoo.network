# Fun with GRE tunnels and some other stuff

We still haven't thought up a decent name for this thing, but our `network.wtf` domain will probably be used.

Basically, this is a way for people to connect to CS and have their outgoing traffic routed through a remote VPS
server without exposing the client's real IP to that VPS, or risking the OpenVPN server keys, or leaking any DNS to the VPS.

That means more exit IPs for clients, and an extra layer that'll confuse the hell out of any correlation attack :-)

# The test instance(s)

On one of the servers in the US west cluster, there were some extra IPs. So using those...

The two extra IPs there
 * `76.164.235.21` for windows people
 * `76.164.235.22` for the nixy crowd
 
For the VPS I'm using some random Indonesian one we just purchased that has two extra IPs
 * `103.56.207.103` for win
 * `103.56.207.104` for nix

On the US west server, four OpenVPN instances are setup
* One for UDP win, listening on `192.168.168.2` on port `443`
 * here clients will get a random local IP in `10.5.0.0/16`
* One for TCP win, listening on `192.168.168.2` on port `443`
 * here clients will get a random local IP in `10.6.0.0/16`
* One for UDP nix, listening on `192.168.169.2` on port `443`
 * here clients will get a random local IP in `10.7.0.0/16`
* One for TCP nix, listening on `192.168.169.2` on port `443`
 * here clients will get a random local IP in `10.8.0.0/16`

Create the VPS' endpoint of the two GRE tunnels
 * `iptunnel add gre1 mode gre local 103.56.207.103 remote 76.164.235.21 ttl 255`
 * `iptunnel add gre2 mode gre local 103.56.207.104 remote 76.164.235.22 ttl 255`

Create the US west server's side of the tunnel
 * `iptunnel add gre1 mode gre local 76.164.235.21 remote 103.56.207.103 ttl 255`
 * `iptunnel add gre2 mode gre local 76.164.235.22 remote 103.56.207.104 ttl 255`

On the US west server, add two [RFC 1918](https://tools.ietf.org/html/rfc1918) IPs to the tunnel so the VPS can communicate with us
 * `ip addr add 192.168.168.2/30 dev gre1`
 * `ip addr add 192.168.169.2/30 dev gre2`

On the VPS, do the same
 * `ip addr add 192.168.168.1/30 dev gre1`
 * `ip addr add 192.168.169.1/30 dev gre2`
    
On both, bring up the `gre1` and `gre2` tunnel interfaces
 * `ip link set gre1 up`
 * `ip link set gre2 up`

Add two poorly named IP rules to both servers
 * `echo '100 tunahl' >> /etc/iproute2/rt_tables`
 * `echo '200 tunahl2' >> /etc/iproute2/rt_tables`

On the VPS, set the default route for the `tunahl` rule to `192.168.168.2`
 * `ip route add default via 192.168.168.2 table tunahl`
 
Also on the VPS, set the default route for the `tunahl2` rule to `192.168.169.2`
 * `ip route add default via 192.168.169.2 table tunahl2`

On the US west server, set the default route for the `tunahl` rule to `192.168.168.2`
 * `ip route add default via 192.168.168.2 table tunahl`
 
On the US west server, set the default route for the `tunahl2` rule to `192.168.169.2`
 * `ip route add default via 192.168.169.2 table tunahl2`
 
Tell the VPS to apply the `tunahl` rule to anything destined for `10.5.0.0/16` or `10.6.0.0/16`
 * `ip rule add to 10.5.0.0/16 table tunahl`
 * `ip rule add to 10.6.0.0/16 table tunahl`

Tell the VPS to apply the `tunahl2` rule to anything destined for `10.7.0.0/16` or `10.8.0.0/16`
 * `ip rule add to 10.7.0.0/16 table tunahl2`
 * `ip rule add to 10.8.0.0/16 table tunahl2`

On the VPS, change the destination of anything going to `103.56.207.103` on `UDP` or `TCP` port `443` to goto `192.168.168.2:443` instead
 * `iptables -t nat -A PREROUTING -p udp -d 103.56.207.103 --dport 443 -j DNAT --to-destination 192.168.168.2:443`
 * `iptables -t nat -A PREROUTING -p tcp -d 103.56.207.103 --dport 443 -j DNAT --to-destination 192.168.168.2:443`

Also on the VPS, change the destination of anything going to `103.56.207.104` on `UDP` or `TCP` port `443` to goto `192.168.169.2:443` instead
 * `iptables -t nat -A PREROUTING -p udp -d 103.56.207.104 --dport 443 -j DNAT --to-destination 192.168.169.2:443`
 * `iptables -t nat -A PREROUTING -p tcp -d 103.56.207.104 --dport 443 -j DNAT --to-destination 192.168.169.2:443`
 
Do source NAT for packets coming into the VPS from the internal OpenVPN IPs so they appear to be coming from the external VPS IP for that VPN instance
 * `iptables -t nat -A POSTROUTING -s 10.5.0.0/16 -j SNAT --to-source 103.56.207.103`
 * `iptables -t nat -A POSTROUTING -s 10.6.0.0/16 -j SNAT --to-source 103.56.207.103`
 * `iptables -t nat -A POSTROUTING -s 10.7.0.0/16 -j SNAT --to-source 103.56.207.104`
 * `iptables -t nat -A POSTROUTING -s 10.8.0.0/16 -j SNAT --to-source 103.56.207.104`
 
And finally (for the VPS), change the source address of packets coming from the external US west IPs so they appear to be coming from the VPS IPs
 * `iptables -t nat -A POSTROUTING -s 76.164.235.21 -j SNAT --to-source 103.56.207.103`
 * `iptables -t nat -A POSTROUTING -s 76.164.235.22 -j SNAT --to-source 103.56.207.104`

Back on the US west server, apply the same `tunahl` IP rules as was done on the VPS, except replace `to` with `from`
 * `ip rule add from 10.5.0.0/16 table tunahl`
 * `ip rule add from 10.6.0.0/16 table tunahl`
 * `ip rule add from 10.7.0.0/16 table tunahl2`
 * `ip rule add from 10.8.0.0/16 table tunahl2`
 
Change all `UDP` and `TCP` packets on any port destined for `76.164.235.21` to `103.56.207.103:443` ([port striping](https://cryptostorm.org/viewtopic.php?t=6034), so clients can connect on any UDP/TCP port)
 * `iptables -t nat -A PREROUTING -p udp -d 76.164.235.21 --dport 1:65534 -j DNAT --to-destination 103.56.207.103:443`
 * `iptables -t nat -A PREROUTING -p tcp -d 76.164.235.21 --dport 1:65534 -j DNAT --to-destination 103.56.207.103:443`
	
Do the same for the nix instance
 * `iptables -t nat -A PREROUTING -p udp -d 76.164.235.22 --dport 1:65534 -j DNAT --to-destination 103.56.207.104:443`
 * `iptables -t nat -A PREROUTING -p tcp -d 76.164.235.22 --dport 1:65534 -j DNAT --to-destination 103.56.207.104:443`
	
And lastly, change the source IP to `76.164.235.21` for `UDP` and `TCP` packets meant for destination port `443` of the windows instance
 * `iptables -t nat -A POSTROUTING -p udp -d 103.56.207.103 --dport 443 -j SNAT --to-source 76.164.235.21`
 * `iptables -t nat -A POSTROUTING -p tcp -d 103.56.207.103 --dport 443 -j SNAT --to-source 76.164.235.21`

Ditto for *nix
 * `iptables -t nat -A POSTROUTING -p udp -d 103.56.207.104 --dport 443 -j SNAT --to-source 76.164.235.22`
 * `iptables -t nat -A POSTROUTING -p tcp -d 103.56.207.104 --dport 443 -j SNAT --to-source 76.164.235.22`


# Ummm, what?
The above is how this whole thing was done, feel free to ignore it if you want :)

But there it is for anyone who cares, and is nerdy enough to understand what the hell is going on.

# Why?
As most of you know, CS only uses dedicated servers for our exit nodes. 
The unmetered dedicated servers we prefer can be very expensive in some locations.
That's the main reason CS has such few IPs.

A lot of VPN providers will run an OpenVPN server on a VPS because a VPS is dirt cheap compared to a dedicated server.
However, it's impossible to secure the OpenVPN process on a VPS since it's just a VM guest, and the server that's the VM host 
can monitor anything going on in the VM guest (or mount the VPS' hard drive image file somewhere and grab the private 
OpenVPN keys and use them to decrypt client traffic).

That's the reason CS has never used a VPS for running an OpenVPN server.

With this setup, we can use a VPS as the exiting point for VPN traffic without any of the aforementioned VPS security issues applying.

# The final plan
World domination! Err, I mean..

I started working on the above after getting the idea of using a VPS as a "jump" node, basically an entry point for VPN traffic.
Since data is encrypted by the client, the "jump" node VPS would only be forwarding pre-encrypted traffic to the true exit node,
and since no OpenVPN keys are on that VPS, there's no way for the VM host to decrypt the traffic going through the VPS.
The added benefit in that setup is that the exit node only sees traffic coming from the VPS' IP, so it never knows a client's real IP.

I was wondering if it was possible to reverse the idea with something tricky like a GRE tunnel + IP rules/routes + iptables.

Turns out it is :-)

Anyways, the final idea is to incorporate a jump node, plus the usual dedicated server, plus this whatever-it's-called exit node VPS thing.
So traffic would hit the entry VPS, goto the dedicated server, then be routed out of the exit VPS.

The befenit of doing it that way is the same as mentioned above, with the addition that we could have a much more redundant network
because we could (if we wanted) prevent the client or the destination from ever knowing the dedicated server's real IP.

The reason that's a good thing is because any DMCA complaint that might get a server killed will goto the exit VPS, and any DDoS attack will
happen against the entry VPS or the exit VPS, which are easier to replace than a dedicated server.

Also, dynamic rules based on packet destination can be added to the dedicated server. 
That means we could set it up where if you visit a place like netflix.com, you automatically get routed to an exit IP in a specific geographical location.
Very useful for services like Netflix, Hulu, Youtube, etc. that restrict access to content based on your geographical location.

# That's all
Now go [buy access tokens](http://cryptostorm.is/#section5) :-P

# Update
Current voodoo instances are as follows:

<sup>Note: I'm unsure what the final terminology will be, but this list should be fairly self-explanatory.
Also, the following "Half voodoo" entries are mapped to the DNS pool for windows.voodoo.network and linux.voodoo.network respectively, but the "Full voodoo" ones are not since I have no clue what the DNS name format will be for that one.</sup>

<sup>Another note: "entry/core" or "entry/jump" is the thing you would tell your OpenVPN client to connect to.</sup>

Half voodoo (entry/core <=> exit):

 	Iceland:
	(win) entry/core: 5.154.191.26 (Moldova),    exit: 151.236.24.12 (Iceland)
	(nix) entry/core: 5.154.191.27 (Moldova),    exit: 151.236.24.85 (Iceland)

	Isle of Man: 
	(win) entry/core: 104.238.194.238 (US west), exit: 37.235.55.73 (Isle of Man)
	(nix) entry/core: 5.154.191.30 (Moldova),    exit: 37.235.55.188 (Isle of Man)
	
Several people have asked me what the point of all this is, if packets have to be routed back to the core before they hit the internet, how does this actually change your "exit" IP?<br>
The short answer is: GRE tunnels :-D

Because of the GRE tunnel between the core and the exit, it allows the core to basically bind() to the exit's IP<br> (Actually, it's binding to an RFC 1918 [192.168.x.x] IP that acts as the endpoint for both sides of the GRE tunnel).<br>
The context of the GRE tunnel and the iptables rules mean that your pre-login packets passing through the exit VPS will only come from the core's IP, and once you're logged into the VPN, the only source IPs the exit VPS will ever see are the RFC 1918 [10.x.x.x] ones randomly assigned to you by the OpenVPN server running on the core. So only the core will know your real IP, or in the case of the "Full voodoo" one, only the "entry/jump" node will know your real IP.

To clarify (and hopefully simplify), for "Half voodoo", the entry/core sees where you're coming from but not where you're going, and the exit VPS sees where you're going but not where you're coming from.

And in "Full voodoo", the entry/jump sees where you're coming from, but not where you're going to, the core doesn't see where you're coming from or where you're going, and the exit sees where you're going but not where you're coming from.

Onto the original README...
