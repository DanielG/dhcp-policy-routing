Policy routing for additional public IP addresses from consumer Cable modems
============================================================================

Some consumer ISPs actually allow you to be assigned more than one public
IP address when their DOCSIS modem is in what they call "Modem Mode".

Of course these addresses are not in any sane organization like a subnet or
something, noooooo, you get a random assortment from their pool of public
addresses.

This makes routing these in addition to your normal, private IPs, all via
your normal gateway a bit of a challange.

While this certainly could be done it seems less than ideal to me. Sure you
could set up a script that dynamically manages the address pool your DHCP
server is handing out to clients, but since I just want to hand these out
to a select few server machienes anyway it's just not necessary and the
router would be another single point of failure.

Instead my solution is to connect the machienes which should get one of
these public address "directly" to the modem via a VLAN on my switch,
though an additional cable would work just as well ;)

## Routing

Whether or not we're routing on the gateway we encounter the complication
that bog standard IP routing simply isn't setup for having more than one IP
addresses where each MUST be used as the source address only on the correct
interface or else the paket will be dropped.

Normally when one machine has multiple IPs in a network they are either in
the same Ethernet segment or at least handled by the same gateway. This is
not so in our case though.

Fortunately the Linux kernel has facilities for crazy use-cases such as
this. The general concept we need to apply is called "policy routing" or
more specifically "source-specific routing" where the source address of a
packet additionally determines to which router and out which interface it
should be sent.

The idea is simple, we tell the kernel:

> Every outgoing packet with a source address matching the public IP we got
> must be sent out via the interface having this address.

We do this by setting up one routing table for each interface which only
contains a default route pointing at the gateway. We then use ip-rule
policy routing to direct packets to the right table when we want to force
them to egress via a particular interface.

For details on Linux Policy Routing see
http://linux-ip.net/html/routing-rpdb.html, the rest of that site is also
very helpful for IP routing on Linux in general.

## Setup

Copy the accompanying script [`multihomed`](./multihomed) to `/etc/dhcp/`
and symlink it to `/etc/dhcp/dhclient-exit-hooks.d`.

The script will then add policy rules for all dhcp interface
undiscriminately. Report an issue if this doesn't work for you.

Optionally it also supports setting up FWMARK based rules for WireGuard
which doesn't (yet) allow binding it's UDP socket to an interface, but it
does support fwmarks.

To enable this you can use an `multihomed-fwmark <some-0xf00-hex-id>`
stanza in `/etc/network/interfaces`. We use `ifquery` to get an interface's
options.

The script will also run hooks in /etc/dhcp/multihomed.d using
`run-parts(8)` whenever the interface address might have changed. I use
this to implement a workaround for WireGuard via a shitty 5G modem's NAT
implementation.

## Policy Route Details

Here's an example configuration and the rules we will
install. In `/etc/network/interfaces` we have:

    auto enp1s0
    iface enp1s0 inet dhcp
    	metric 1024

    auto  enp2s0.5
    iface enp2s0.5 inet dhcp
    	metric 2048
    	multihomed-fwmark 0x2005

Here we use the `metric` setting to determine which interface is used by
default (greater number is less preferred).

And the resulting rules and per-interface route tables are as follows:

    $ ip rule
    32752:	from all fwmark 0x2005 lookup main suppress_prefixlength 0
    32753:	from all fwmark 0x2005 lookup 16909060
    32754:	from all fwmark 0x2005 unreachable
    32755:	from 1.2.3.4 lookup main suppress_prefixlength 0
    32756:	from 1.2.3.4 lookup 16909060
    32757:	from 192.168.42.222 lookup main suppress_prefixlength 0
    32758:	from 192.168.42.222 lookup 3232246494
    32766:	from all lookup main
    32767:	from all lookup default

    $ ip route show table 3232246494
    default via 192.168.42.1 dev enp1s0

    $ ip route show table 16909060
    default via 1.2.3.1 dev enp2s0.5
    
Notice that the `lookup` aka. table IDs are just the 32-bit integer
representation of the associated IP address.

The `lookup main suppress_prefixlength 0` rules are interesting too. See if
we only insert the per-interface table lookup rules LAN destinations will
become unreachable which is not quite what you'd want usually.

To fix this we use the `suppress_prefixlength 0` match. When the default
route matches in the lookup table it is ignored and policy rule evaluation
continues. In effect you get uniform access to local LAN subnets while
still routing via the right interface for global destinations.
