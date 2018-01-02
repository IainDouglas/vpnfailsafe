**NOTE: [ca284a9](../../../../wknapik/vpnfailsafe/commit/ca284a95e5bdcc0956ced2ae6def96c629d19ed6#diff-d7fe32a26c0579199da6c3df52843244) fixes a potential dns leak ([#25](../../../../wknapik/vpnfailsafe/issues/25))**

# What is vpnfailsafe ?

`vpnfailsafe` prevents a VPN user's ISP-assigned IP address from being exposed
on the internet, both while the VPN connection is active and when it goes down.

`vpnfailsafe` doesn't affect traffic to/from private networks, or disrupt existing
firewall rules beyond its intended function.

# How does it work ?

`vpnfailsafe` ensures that all traffic to/from the internet goes through the VPN.
It is meant to be executed by OpenVPN when the tunnel is established (--up), or
torn down (--down).

On --up:
* All configured VPN server domains are resolved and saved in /etc/hosts.
* Routes are set up, so that all traffic to the internet goes over the tunnel.
  The original default route is preserved and two more specific ones are added
  (mimicking --redirect-gateway def1) + routes to all configured VPN servers
  are added.
* /etc/resolv.conf is updated, so only the DNS servers pushed by the VPN server
  are used.
* iptables rules are inserted at the beginning of INPUT, OUTPUT and FORWARD
  chains to ensure that the only traffic to/from the internet is between the
  VPN client and the VPN server.

On --down:
* The /etc/hosts entries for VPN servers remain in place, so the VPN connection
  can be re-established without allowing traffic to DNS servers outside the VPN.
* Previously added routes are removed.
* Previous /etc/resolv.conf is restored.
* Firewall rules remain in place, allowing only the re-establishment of the vpn
  tunnel.

# How do I install/use it ?

Save vpnfailsafe&#46;sh in /etc/openvpn, make it executable and add the
following lines to /etc/openvpn/\<your_provider\>.conf:

```
script-security 2
up /etc/openvpn/vpnfailsafe.sh
down /etc/openvpn/vpnfailsafe.sh
```

That's it.

Since `vpnfailsafe` contains the functionality of the popular
update-resolv-conf&#46;sh script, the two don't need to be combined.

A complete configuration example is included as
[extras/example.conf](extras/example.conf).

Arch Linux users may choose to install the
[vpnfailsafe-git](https://aur.archlinux.org/packages/vpnfailsafe-git/) package
from AUR instead.

# What are the requirements/assumptions/limitations ?

`vpnfailsafe` works only on Linux.

Dependencies are minimal (listed in the [PKGBUILD](PKGBUILD) file). Of note is
the openresolv requirement. There are at least two different, popular packages
providing the resolvconf binary, which are not compatible (one supports the
`-x` switch used by `vpnfailsafe` and the other does not). On distributions
where multiple implementations are available, openresolv should be chosen.
On CentOS/RHEL systems this [openresolv](https://github.com/rsmarples/openresolv)
project is known to work. 

The only assumption is that the VPN server will push at least one DNS server to
the client.

`vpnfailsafe` does not handle [ipv6](https://en.wikipedia.org/wiki/IPv6) at
all. To prevent leaks, ipv6 should be disabled and/or blocked. See:
[extras/disable_ipv6.conf](extras/disable_ipv6.conf) for an example of a sysctl
config file that disables it and [extras/block_ipv6.sh](extras/block_ipv6.sh)
for firewall rules to block it.

`vpnfailsafe` has been tested with all device types and topologies supported by
OpenVPN.

# I'm getting an "RTNETLINK answers: Permission denied" error.

This usually means that OpenVPN was executed without sufficient privileges. But
if the line is followed by "Linux ip -6 addr add failed: external program
exited with error status: 2", then it probably means, that ipv6 is disabled on
the system, but the VPN server is pushing ipv6-related options and the client
fails trying to run `ip -6' to honor them. The following two options can be
added to the client config to make it ignore the ipv6-related configuration:
```
pull-filter ignore "ifconfig-ipv6 "
pull-filter ignore "route-ipv6 "
```
(included in [extras/example.conf](extras/example.conf))

# I'm getting an "RTNETLINK answers: File exists" error every time I connect.

Those errors can be ignored safely. They appear when OpenVPN tries to set up a
route, that's already been created by `vpnfailsafe`.  Adding the `route-noexec`
option will tell OpenVPN to leave routing to `vpnfailsafe` and prevent those
errors from appearing.

# How do I make OpenVPN reconnect when the underlying network connection is re-established ?

Send the HUP signal to OpenVPN upon reconnection.

Dhcpcd users would use dhcpcd-run-hooks, NetworkManager users would use a
dispatcher script (e.g.: [extras/pkill_hup_openvpn](extras/pkill_hup_openvpn)).

# How do I restore my system to the state from before running vpnfailsafe ?

`vpnfailsafe` will revert all changes when the tunnel is closed, except for the
firewall rules. You can restore those using the init script that set the
iptables rules on boot, or by using iptables-restore, or by otherwise removing
the VPNFAILSAFE_INPUT, VPNFAILSAFE_OUTPUT and VPNFAILSAFE_FORWARD chains.

The /etc/hosts entries may eventually become stale and also require removal.

The [extras/vpnfailsafe_reset.sh](extras/vpnfailsafe_reset.sh) script can be
used to achieve that.

# Will vpnfailsafe protect me against DNS leaks ?

Yes. See "How does it work ?" for more details.

That being said, if your life, job, or whatever you care about depend on your
IP not leaking, consider that this script has been tested by only a handful of
people. YMMV.

# Will vpnfailsafe protect me against all forms of IP leaks ?

No. Application level leaks can still happen, via protocols like WebRTC. The
user can also announce their identity to the world and no script will stop
them.

# Do I still need to configure a firewall ?

Yes. `vpnfailsafe` limits what kind of traffic is allowed, but only to achieve
its goals. Otherwise everything is passed through to pre-existing firewall
rules.

An example of a basic firewall is included as
[extras/basic_firewall.sh](extras/basic_firewall.sh).

# Aren't there already scripts that do all that ?

One would think so, but then one would be wrong.

What is out there are mostly "applications", with non-optional GUIs and
thousands of lines of code behind them, often VPN-provider specific.

# What else can I do to improve my security/privacy ?

As far, as OpenVPN goes - you can check the [hardening
section](https://openvpn.net/index.php/open-source/documentation/howto.html#security)
of the official documentation.

The steps necessary to run OpenVPN as an [unprivileged
user](https://community.openvpn.net/openvpn/wiki/UnprivilegedUser) can be run
automatically via the [openvpn-unroot
script](https://github.com/wknapik/openvpn-unroot).
