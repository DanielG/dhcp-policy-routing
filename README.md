Policy routing for additional public IP addresses from consumer Cable modems
============================================================================

Some consumer ISPs actually allow you to be assigned more than one public
IP address when their DOCSIS modem is in what they call "Modem Mode".

Of course these addresses are not in any sane organization like a subnet or
something, noooooo, you get a random assortment from their pool of public
addresses.

This makes routing these in addition to your normal, private IPs, all via
your normal gateway a bit of a challange.

While this certainly could be done it seems overly complicated to me. Sure
you could setup a script that dynamically manages the address pool your
DHCP server is handing out to clients, but since you likely want to just
hand these out to a select few server machienes it's just not necessary and
does just add another single point of failure.

Instead my solution is just to connect the machienes which should get one
of these public address "directly" to the modem via a VLAN on my switch,
though an additional cable would work just as well ;)



## DHCP / MACVLAN

In my particular case the DOCSIS modem is very picky about how many public
addresses it will give out via DHCP out and will just stop responding to
DHCP requests once the limit is reached. It is also _very_ persistent about
this MAC-IP associaction. It seems only a factory reset will dissolve it,
or the lease time is very long.

For this reason I use a set of locall administered, static, MAC addresses
instead of the normal interface MACs on the modem VLAN. This is
accomplished by using macvlan intearfaces.

In my case the macvlan is stacked on top of a VLAN interface but for
simplicity's sake let's just call the underlying interface "eth0" since it
doesn't make a difference:

    ip link add link eth0 name eth0v0 address 02:RA:ND:OM:1M:AC \
        type macvlan mode private

Replace 02:RA:ND:OM:1M:AC with a random, preferably locally administered
MAC address. Protip: Leaving the first byte as "02" is an easy way of
setting the "locally administered" flag without bit fiddling.



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
more specifically "source routing" where the source address of a packet
additionally determines to which router and out which interface it should
be sent.

The idea is we tell the kernel:

> Every outgoing packet with a source address matching the public IP we got
> should be sent out via the interface it came in on.

And that's what the script `IFACE.exit-hook` accomplishes.



## Setup

> TODO: Just a rough sketch, report issues if this is unclear

Add a new routing table-ID to name association:

    $ echo 123 pub >> /etc/iproute2/rt_tables
    $ cat /etc/iproute2/rt_tables
    #
    # reserved values
    #
    255	local
    254	main
    253	default
    0	unspec
    #
    # local
    #
    123 pub

This is technically optional, it's just used by `ip(8)` to display pretty
names instead of the raw ids. The range of ID values you can choose is
`1-253`.

The accompanying script [`IFACE.exit-hook`](./IFACE.exit-hook) implements
the reset by running two simple commands whenever the dhcp lease changes:

    ip rule add from $new_ip_address table pub
    ip route add default via $new_router table pub

For details on Policy Routing on Linux see
http://linux-ip.net/html/routing-rpdb.html, the rest of that site is also
very helpful for IP routing on Linux in general.

To install that part install and configure dhcpcd, on Debian:

    $ apt-get install dhcpcd5

This is a replacement for Debian's old-school /etc/network/interfaces
aka. ifupdown stuff. If you prefer `interfaces(5)` you can also opt to use
the included script with `dhclient(8)`.

Anyways, make dhcpcd run `/etc/dhcpcd/$interface.exit-hook`:

    $ cat > /etc/dhcpcd.exit-hook <<EOF
    #!/bin/sh

    if [ -x /etc/dhcpcd/$interface.exit-hook ]; then
        /etc/dhcpcd/$interface.exit-hook
    fi
    EOF

No idea why they don't just have something like this by default.

Finally install the policy routing exit-hook:

    $ mkdir -p /etc/dhcpcd
    $ cp IFACE.exit-hook /etc/dhcpcd/eth0v0.exit-hook

And there you go.

Don't forget to go though the general dhcpcd config, by default it will
just do DHCPv{4,6}, IPv4LL etc. on all interfaces. It could also conflict
with `interfaces(5)` config, not sure. For all you know it might eat your
babies so RTFM.

I use something like the followin in `/etc/dhcpcd.conf`:

    allowinterfaces eth0,eth0v0
    interface eth0v0
        nogateway #< Adding a default route for this interface in the main
                  # routing table doesn't help things

        noipv4ll  #< Having some random IP on the interface if DHCP doesn't
                  #respond isn't very helpful here

        noipv6    #< I know my ISP doesn't assign any v6 addrs anyways

        # don't mess with hostname and dns
        nohook 20-resolv.conf,30-hostname
