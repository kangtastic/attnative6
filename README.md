# **attnative6**
Scripts to enable native dual-stack IPv6 on AT&amp;T with your own router

If you skip to the [installation](#installation) section, do read through all of this at some point.

### Problem
A bypassed router on AT&T's residential u-Verse network does not pull a DHCPv6 lease from upstream DHCPv6 servers.

This is because most DHCPv6 clients available as of this writing (December 2017) default to sending a DUID-LL or DUID-LLT, and do not support configuring a [DUID-EN](https://tools.ietf.org/html/rfc3315#section-9.3).

A DUID-EN is stored in the Residential Gateway by the manufacturer, and is sent with every DHCPv6 request from the RG. No DUID-EN, no DHCPv6 assignment.

AT&T has long offered 6rd on its network as an alternative for IPv6 connectivity that, frankly, lacks for nothing in functionality compared to native IPv6 for the vast majority of users. But a thing is possible, so we must do it.

### Investigation
How to figure out the DUID-EN assigned to us? Looking at RFC 3315, a DUID-EN is comprised of 2 fields - an IANA-assigned enterprise number and an arbitrary, vendor-defined hunk of data.

By examining the DHCPv6 requests sent by the RGs with the help of **tcpdump**, the format of the DUID-EN has been decoded. And it turns out that it's quite simple:

<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   unsigned integer: IANA-assigned Private Enterprise Number   |    Field 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-------------------
|    IEEE-assigned OUI (Organizationally Unique Identifier)     |    Field 2
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  of RG mfg'r (2wire or Arris) | Hyphen ("-")  |               |    Also field 2
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               |
.                                                               .    Still field 2
.                    Serial number of RG                        .
.                     (variable length)                         .    Yup, field 2
.                                                               .
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
</pre>

Everything is ASCII except for the PEN at the beginning. In other words, it's a `uint32` followed by something like `001F2E-123456789012`.

One PEN, `3561`, has been discovered so far. It's listed [here](https://www.iana.org/assignments/enterprise-numbers/enterprise-numbers) (warning: ridiculously long text file) as belonging to `The Broadband Forum (formerly 'ADSL Forum')`, so that seems right.

Two OUIs, `00D09E` and `001E46`, have been discovered so far. Those are listed [here](http://standards-oui.ieee.org/oui.txt) (warning: another ridiculously long text file) as belonging to `2Wire Inc` and `ARRIS Group, Inc.`, and as they were sent respectively by a 2Wire Pace 5286AC and an Arris Motorola NVG599, those seem right too.

Hmm, that's enough information to reconstruct each RG's DUID-EN. Go configure your DHCPv6 clients for a DUID-EN now and enjoy. Thanks for reading!

...Oh right, most DHCPv6 clients. No DUID-EN support.

### Solution
[dibbler](http://klub.com.pl/dhcpv6/) is a DHCPv6 server, relay, and client, and it *does* support configuring a DUID-EN. The client portion (aptly named `dibbler-client`) definitely does.

**Obvious Q:** "B-b-but I've never heard of dibbler! Why would I want to install some janky Warsaw Bloc DHCPv6 client?"

**A:** It *is* janky! Don't start it up with its default configuration before you edit the configuration file to make it send a DUID-EN, or it won't actually send the DUID-EN! (If only I'd known this months ago, when I first looked into dibbler and gave up on it as broken.)

But it does work. It made beautiful-looking DHCPv6 packets (fine, *datagrams*) and sent them out, and although the servers proceeded to cheerfully ignore everything contained within them except the parts where dibbler asked for an IPv6 address and a prefix delegation, *the servers replied*. Router Advertisements were sent which my router *did not then ignore*.

### Some caveats to the solution
1. AT&T delegates a /60, and by default `dibbler-client` will split that into a /68 for every non-WAN interface it finds on the system, because of course it will. A partial remedy is to split the delegation yourself, which just means that you divide it up however you like and assign the new subnets to the network interface(s) on your LAN.

2. `dibbler-client` will bind the IA_NA ("IPv6 address") to your WAN interface, because that is its job. `dibbler-client` won't set up a default IPv6 route, or broadcast Router Advertisements to the LAN, or send LAN clients DNS6 server information, which are basic steps required for IPv6 connectivity, because that's not its job.
   * Those duties are for, respectively, the kernel to listen to Router Advertisements on the WAN interface, `radvd` or `dnsmasq`, etc. to send out RAs on the LAN interface, and `radvd` or `dnsmasq`, etc. or a DHCPv6 server to send out the DNS6 information in RAs or DHCPv6 assignments. 

3. Sorry, BSDudes, `dibbler-client` isn't included in BSD or pfSense's package repositories. It builds nicely from the [sources elsewhere on github](https://github.com/tomaszmrugalski/dibbler), though.
   * Just remember to create at least the `/var/lib/dibbler` directory. Also, write a config file at `/etc/dibbler/client.conf`, which may (start out as) the one you generate with the installer script.

### Partial solutions, to some of the caveats, to the solution
#### OK, this is where I get around to talking about the things in this repository.

**`install-attnative6`** is a script to make configuring `dibbler-client` and getting native IPv6 easier. It does the following:
1. Prompts for the necessary information to generate a working configuration for `dibbler-client` from **`client.conf.template`**
2. Also generates a (Linux-specific, but potentially portable) helper script from **`pdsplit.sh.template`**, which will be called when `dibbler-client` gets a DHCPv6 assignment to:
   * Assign the first of the 16 /64s contained in a delegated /60 (well, anything /64 or shorter) to a single LAN interface
   * Force the kernel to listen to Router Advertisements coming in on the WAN
3. Checks for running instances of `dibbler-client` and kills them
4. Checks for an old `dibbler-client` configuration, possibly created by the instances of `dibbler-client` it just killed, and gets rid of them
   * Said old configuration will probably interfere with sending the DUID-EN generated in 1.
5. Prompts to install the configuration generated in 1. and the helper script generated in 2. to appropriate directories
----
## Installation
Sorry, instructions for Linux only for now.
BSDudes need to compile and install the client and can run the generator script, but the helper script shouldn't be installed without at least converting all the relevant calls to Linux's `ip` into calls to BSD's `ifconfig`.
### Prerequisites
1. Disable the current DHCPv6 client from running on your WAN interface, if any.
2. Get and install `dibbler-client` from your Linux distribution's package repository.
   * If the package installer prompts you to ask if you want to start the client during boot, **REFUSE** for now.
   * If you didn't refuse, don't worry, the installer script can remedy your error later.
3. (Optional, highly recommended) Also get and install `rdisc6`, which is contained in a package named `ndisc6`.
### Installation
1. Clone the repository, or download the first release and untar it.
2. `cd` to where the release ended up on your system, run **`./install-attnative6`** as root, and answer the questions.
   * Note that the generated files will make `dibbler-client` start only on the WAN interface you specified, and will only assign the first /64 in the delegated prefix to the LAN interface. If this is not what you want, edit the appropriate files later. Both of them are probably in the same place, after all...
### Usage
1. Once the files were correctly generated and installed, start the client with most likely, `systemctl start dibbler-client` as root.
   * It should go without saying, but if you were using a 6rd tunnel before, stop and remove the tunnel, stop it from being autocreated, rchange your firewall rules if they referenced the tunnel device name, and make any necessary changes to your DNS6, DHCPv6, and Router Advertisement daemon configurations before you start `dibbler-client` for the first time.
2. Read the log files in `/var/log/dibbler` to verify that the client and helper script are working.
   * If you're running into problems, edit `/etc/dibbler/client.conf` to change the log level to 8, and also to make any other changes there and in the helper script at `/etc/dibbler/pdsplit.sh`.
   * Restart the client with, most likely, `systemctl restart dibbler-client` as root.
   * To remove everything dibbler-related and start over, run `apt-get purge dibbler-client` and `rm -f /etc/dibbler /var/log/dibbler /var/lib/dibbler`, both as root, then run `install-attnative6` again.
3. Once the client is fully working, make it start at boot with `systemctl enable dibbler-client` as root.
---
Good luck!
